# 在Python中处理长时间运行的子进程
> 原文来自：https://eli.thegreenplace.net/2017/interacting-with-a-long-running-child-process-in-python/#id2
---
Python中的`subprocess`模块是创建子进程并与之交互的强大瑞士军刀。它包含几个高级API，例如`call`、`check_output`以及`run`（Python3.5引入），专门用于在程序中创建子进程并等待其完成。

本文将要讨论该任务的变种，可以间接解决长时间运行的子进程。考虑这样的场景——测试服务器，例如一个HTTP服务器。我们以子进程启动它，然后让客户端连接它并运行一些测试序列。测试完之后我们要有序关闭子进程。如果用那些运行子进程后阻塞直到子进程结束的API将很难完成（因为服务器不会自己结束），所以我们试试更底层的API。

当然，我们可以在一个线程中用`subprocess.run`启动一个子进程并在另一个线程中和它交互（通过某个已知的端口）。但这样的话就很难在结束的时候彻底关闭子进程。如果子进程有一个有序的关闭序列（比如发送某些退出命令），这是可行的。不幸的是大部分服务器不是这样的，在被强杀之前会不断运行下去。这是本文试图解决的案例。

## 启动、交互、关闭并在结束时获取所有输出
第一个最简单的例子是启动一个HTTP服务器、和它交互、彻底关闭它并在结束时获取服务器的所有标准输出和标准错误。下面是主要的代码片段（完整代码看[这里](https://github.com/eliben/code-for-blog/tree/master/2017/python-interact-subprocess)），已经在Python3.6下测试过。
```python
def main():
    proc = subprocess.Popen(['python', '-u', '-m', 'http.server', '8070'],
                            stdout=subprocess.PIPE,
                            stderr=subprocess.STDOUT)

    try:
        time.sleep(0.2)
        resp = urllib.request.urlopen('http://localhost:8070')
        assert b'Directory listing' in resp.read()
    finally:
        proc.terminate()
        try:
            outs, _ = proc.communicate(timeout=0.2)
            print('== subprocess exited with rc =', proc.returncode)
            print(outs.decode('utf-8'))
        except subprocess.TimeoutExpired:
            print('subprocess did not terminate in time')
```
这里的子进程是用Python自带的`http.server`模块创建的HTTP服务器，将启动位置的目录作为服务内容。我们用底层的`Popen`API来异步启动子进程（意味着`Popen`会立即返回，而子进程将在后台运行）。

注意调用Python时传递的`-u`参数：这可以避免标准输出缓存，当子进程被杀死时可以让我们尽可能看到更多的标准输出。和子进程交互时缓存容易引起大问题，我们之后会看到更多这方面的例子。

这个例子中主要部分发生在`finally`代码块中。`proc.terminate()`给子进程发送一个`SIGTERM`信号。然后，`proc.communicate`等待子进程退出并捕获所有的标准输出。自从Python3.3开始，`communicate`引入一个非常方便的`timeout`参数，<span style="border-bottom:2px dashed;">[1]</span>这样因为某些原因子进程没能退出我们也可以知道。如果`SIGTERM`不能退出子进程，更复杂的技术是发送`SIGKILL`信号（通过`proc.kill`）。

如果你在POSIX上运行脚本，你会看到这样的输出，Windows则是1：
```
PS C:\Users\YL> python -u interact-http-server.py
== subprocess exited with rc = -15
Serving HTTP on 0.0.0.0 port 8070 (http://0.0.0.0:8070/) ...
127.0.0.1 - - [20/Jul/2020 21:34:35] "GET / HTTP/1.1" 200 -
```
子进程的返回码是-15（负数意味着被信号中断，15则是`SIGTERM`的数字码）。标准输出被正确地捕获并打印出来。

## 启动、交互、实时获取输出、终止
一个相关的例子是实时获取子进程的标准输出，而不是在最后一下子全部获取。这时我们需要谨慎对待缓存，因为它很容易会让程序死锁。Linux的进程通常会在交互模式下按行缓存，其他情况下全部缓存。很少有进程会完全不缓存。因此，我不推荐读取不到一行的标准输出。真的，别那么做。标准I/O是以行方式使用的（想想Unix命令行工具是怎么工作的）；如果你需要比行更小的粒度，别用标准输出（用socket或者别的什么）。

总之，下面是我们的例子：
```python
def output_reader(proc):
    for line in iter(proc.stdout.readline, b''):
        print('got line: {0}'.format(line.decode('utf-8')), end='')


def main():
    proc = subprocess.Popen(['python', '-u', '-m', 'http.server', '8070'],
                            stdout=subprocess.PIPE,
                            stderr=subprocess.STDOUT)

    t = threading.Thread(target=output_reader, args=(proc,))
    t.start()

    try:
        time.sleep(0.2)
        for i in range(4):
            resp = urllib.request.urlopen('http://localhost:8070')
            assert b'Directory listing' in resp.read()
            time.sleep(0.1)
    finally:
        proc.terminate()
        try:
            proc.wait(timeout=0.2)
            print('== subprocess exited with rc =', proc.returncode)
        except subprocess.TimeoutExpired:
            print('subprocess did not terminate in time')
    t.join()
```
这个例子在处理标准输出上不太一样，这次没有调用`communicate`，而是用`proc.wait`等待子进程退出（在发送了`SIGTERM`之后）。用一个线程来轮询子进程的`stdout`属性，只要新的行可以获取就立刻将其打印出来。如果你运行这个例子，你会发现子进程的标准输出会实时打印出来，而不是在最后一下子打印出来。

`iter(proc.stdout.readline, b'')`这段代码会持续调用`proc.stdout.readline()`（当然，只有在有输出行的时候才会返回，否则将会阻塞），直到函数返回空的字节字符串。只有当`proc.stdout`被关闭的时候才会返回空的字节字符串，也就是子进程退出的时候。因此，尽管读取输出的线程看似永远不会终止——实际上是一定会终止的！只要子进程还在运行，线程将会在`readline`上阻塞；当子进程终止的时候，调用`readline`将会返回`b''`，之后线程退出。

如果我们不光只是想把捕获的标准输出打印出来，还想对它做一些事情（比如搜寻预期的模式），用Python的线程安全的队列很容易组织，把读取线程变成这样：
```python
def output_reader(proc, outq):
    for line in iter(proc.stdout.readline, b''):
        outq.put(line.decode('utf-8'))
```
可以这样启动它：
```python
outq = queue.Queue()
t = threading.Thread(target=output_reader, args=(proc, outq))
t.start()
```
这样在任何时候我们都可以使用非阻塞模式来查看队列中的元素（完整代码看[这里](https://github.com/eliben/code-for-blog/blob/master/2017/python-interact-subprocess/interact-server-with-thread-queue.py)）
```python
try:
    line = outq.get(block=False)
    print('got line from outq: {0}'.format(line), end='')
except queue.Empty:
    print('could not get line from queue')
```

## 直接和子进程的标准输入输出交互
这个例子有点危险，`subprocess`模块的文档警告不要像这边的描述一样去做，因为可能会引起死锁，但有时别无选择！一些程序喜欢用标准输入输出来交互。或者，你可能有一个程序（解释器），你想测试它的交互式模式——比如Python解释器自身。有时候可以一下子把所有的输入给到程序然后检查它的输出；这可以也应该通过`communicate`来完成——这是用于这个目的的最佳API。它会正确给定标准输入，在完成的时候关闭（标志着游戏结束了许多互动程序），等等。但如果我们想基于子进程之前的输出来提供新的输入呢？比如：
```python
def main():
    proc = subprocess.Popen(['python', '-i'],
                            stdin=subprocess.PIPE,
                            stdout=subprocess.PIPE,
                            stderr=subprocess.PIPE)
    # To avoid deadlocks: careful to: add \n to input, flush input, use
    # readline() rather than read()
    proc.stdin.write(b'2+2\n')
    proc.stdin.flush()
    print(proc.stdout.readline())

    proc.stdin.write(b'len("foobar")\n')
    proc.stdin.flush()
    print(proc.stdout.readline())

    proc.stdin.close()
    proc.terminate()
    proc.wait(timeout=0.2)
```
我来解释一下代码里的注释：
* 当在一个基于行的解释器中进行输入时，别忘了显式输入换行符。
* 在文本流中输入了数据后要清空缓冲区，因为输入的数据可能被缓存。
* 用`readline`来获取解释器的输出。

我们应该小心，避免下面的情况：
1. 我们给子进程的标准输入发送数据，但因为某些原因没有获取完整输入（缺少换行符，被缓存了等等）。
2. 然后我们调用`readline`等待返回。

因为子进程还在等待输入完成（第1步），第2步将永远等着。这是一个*典型的死锁*。

在最后，我们关闭了子进程的`stdin`（这是可选的，但对某些子进程很有用），调用了`terminate`，然后`wait`。更好的做法是给子进程发送某种退出命令（在Python解释器的例子中是`quit()`）；这边的`terminate`是为了演示如果其他选项不可用，我们必须做什么。注意这边我们也可以用`communicate`而不是`wait`来捕获标准错误的输出。

# 用非阻塞的读取和可停止的线程来交互
最后的例子演示了一个略微更高级的方案。假设我们正在测试一个长期运行的套接字服务器，我们感兴趣的是和它复杂的交互，可能有多个并存的客户端。我们也想彻底关闭整个线程和子进程。[完整的代码在这里](https://github.com/eliben/code-for-blog/blob/master/2017/python-interact-subprocess/interact-socket-server-noblock.py)，下面是几个有代表性的代码片段，关键的组成部分是这个套接字读取函数，它可以在自己的线程中运行：
```python
def socket_reader(sockobj, outq, exit_event):
    while not exit_event.is_set():
        try:
            buf = sockobj.recv(1)
            if len(buf) < 1:
                break
            outq.put(buf)
        except socket.timeout:
            continue
        except OSError as e:
            break
```
用的时候最好设置超时时间，这个函数会反复监视套接字，发现新数据就推送到`outq`中，<span style="border-bottom:2px dashed;">[2]</span>这边的`outq`是`queue.Queue`。当套接字关闭（`recv`返回空的字节字符串）或者`exit_event`（一个`threading.Event`）被调用者设置的时候函数都会退出。

调用者可以在线程中启动这个函数并且偶尔尝试以非阻塞的形式从队列读取新的元素：
```python
try:
    v = outq.get(block=False)
    print(v)
except queue.Empty:
    break
```
当一切都结束时，调用者可以设置用于退出的`Event`来停止线程（当读取的套接字被关闭后线程自己会停止，但是这边设置的事件让我们能更直接地控制）。

## 结语
本文描述的任务没有一劳永逸的解决方法；我提出了一系列解决更常见情况的方法，但是很可能特定的例子没法用它们解决。如果你碰到了有意思的例子，不管这边的方法有没有帮你解决，都请告诉我。像往常一样，也欢迎其他的反馈。

---
[1]Python早期的版本需要用线程来模拟这个功能。

[2]这个例子中一次只获取一个字节，但是想要接收更大的块，改起来也很简单。