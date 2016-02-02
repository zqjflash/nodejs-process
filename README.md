### Node.js进程

> Node在选型时决定在V8引擎之上构建，也就意味着它的模型与浏览器类似。JS将会运行在单个进程的单个线程上。好处是程序状态是单一的，没有多线程的锁、线程同步问题，操作系统调度时也因为较少上下文切换，可以很好的提高CPU的使用率。单线程并非完美，如今CPU基本是多核，真正的服务器（非VPS）往往还有多个CPU。一个Node进程只能利用一个核，这将抛出Node实际应用的第一个问题：如何充分利用多核CPU服务器？

由于Node执行在单线程上，一旦单线程上抛出的异常没有被捕获，将会引起整个进程的崩溃。这给Node实际应用抛出了第二个问题：如何保证进程的健壮性和稳定性？

从严格意义上而言，Node并非真正的单线程架构，Node自身还有一定的I/O线程存在，这些I/O线程由底层libuv处理，这部分线程对于JS开发者而言是透明的，JS代码运行在V8上，是单线程的。

### 服务模型的变迁

* 石器时代：同步

> 执行模型是同步，服务模式是一次只为一个请求服务，所有请求都得按次序等待服务。这意味着除了当前的请求被处理外，其余请求都处于耽误的状态。假设每次响应服务耗时为N秒，这类服务的QPS为1/N。

* 青铜时代：复制进程

> 为解决同步架构问题，简单改进通过进程复制同时服务更多的请求和用户。每个连接需要一个进程来服务，100个连接需要启动100个进程，非常昂贵。进程复制过程中，需要复制进程内部的状态，对于每个连接都进行这样的复制，相同状态将在内存中存在很多份，造成浪费。并且这个过程由于要复制较多的数据，启动是较为缓慢的。

  __为了解决启动缓慢问题，预复制（prefork）被引入服务模型中，即预先复制一定数量的进程。同时将进程复用，避免进程创建、销毁带来的开销。同时将进程复用，避免进程创建、销毁带来的开销，但这个模型不具备伸缩性，一旦并发请求过高，内存使用随着进程数的增长将会被耗尽。假设通过进行复制和预复制的方式搭建的服务器有资源的限制，且进程数上限为M，那这类服务的QPS为M/N。__

* 白银时代：多线程

> 为了解决进程复制中浪费问题，多线程被引入服务模型，让一个线程服务一个请求。线程相对进程的开销要小许多，并且线程之间可以共享数据，内存浪费的问题可以得到解决，并且利用线程池可以减少创建和销毁线程的开销。但是多线程所面临的并发问题只能说比多进程略好，因为每个线程都拥有自己独立的堆栈，这个堆栈都需要占用一定的内存空间。

__由于一个CPU核心在一个时刻只能做一件事情，操作系统只能通过将CPU切分为时间片的方法，让线程可以较为均匀地使用CPU资源，但是操作系统内核在切换线程的同时也要切换线程的上下文，当线程数量过多时，时间将会被耗用在上下文切换中。所以在大并发量时，多线程结构还是无法做到强大的伸缩性。如果忽略掉多线程上下文切换的开销，假设线程所占用的资源为进程的1/L，受资源上限的影响，它的QPS则为M*L/N__

* 黄金时代：事件驱动

> Apache就是采用多线程/多进程模型实现的，当并发增长到上万时，内存耗用的问题将会暴露出来，这即是注明的C10k问题。为了解决高并发问题，基于事件驱动的服务模型出现，Node与Nginx均是基于事件驱动的方式实现的，采用单线程避免了不必要的内存开销和上下文切换开销。

__基于事件的服务模型存在两个问题：1、CPU的利用率；2、进程的健壮性。__

__对于Node来说，所有请求的上下文都是统一的，稳定性是亟需解决的问题，由于所有处理都在单线程上进行，影响事件驱动服务模型性能的点在于CPU的计算能力，它的上限决定这类服务模型的性能上限，不受多进程和多线程模式中资源上限的影响，可伸缩性远比前两者高。如果解决掉多核CPU的利用问题，带来的性能提升是可观的。__

### 多进程架构

> 面对单进程单线程对多核使用不足的问题，前人的经验是启动多进程即可。理想状态下每个进程各自利用一个CPU，以此实现多核CPU的利用。Node提供了child_process模块，并且提供了child_process.fork()函数供我们实现进程的复制。

示例代码worker.js文件：

```js
var http = require('http');
http.createServer(function(req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
}).listen(Math.round((1+Math.random()) * 1000), '127.0.0.1');
```

通过node workder.js启动它，将会侦听1000到2000之间的一个随机端口。

将以下代码存为master.js，并通过node master.js启动它：

```js
var fork = require('child_process').fork;
var cpus = require('os').cpus();
for (var i = 0; i < cpus.length; i++) {
  fork('./worker.js');
}
```

这段代码将会根据当前机器上的CPU数量复制出对应Node进程数。在*nix系统下可以通过ps aux | grep worker.js查看进程的数量，如下所示：

```js
  $ ps aux | grep worker.js
  jacksontian 1475 0.0 0.0 2432768 600 s003 S+ 3:27AM 0:00.00 grep worker.js
  jacksontian 1440 0.0 0.2 3022452 12680 s003 S 3:25AM 0:00.14 /usr/local/bin/node ./worker.js
  jacksontian 1439 0.0 0.2 3023476 12716 s003 S 3:25AM 0:00.14 /usr/local/bin/node ./worker.js
  jacksontian 1438 0.0 0.2 3022452 12704 s003 S 3:25AM 0:00.14 /usr/local/bin/node ./worker.js
  jacksontian 1437 0.0 0.2 3031668 12696 s003 S 3:25AM 0:00.15 /usr/local/bin/node ./worker.js
```

Master-Worker模式，又称主从模式。分为主进程和工作进程。这是典型的分布式架构中用于并行处理业务的模式，具备较好的可伸缩性和稳定性。主进程不负责具体的业务处理，只负责调度或管理工作进程，趋向于稳定。工作进程负责具体的业务处理，业务变化多样，开发者主要关注工作进程。

  ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-process/master/master-worker.png)

通过fork()复制的进程都是一个独立的进程，这个进程中有着独立而全新的V8实例。它需要至少30毫秒的启动时间和至少10MB的内存。尽管Node提供了fork()供我们复制进程使每个CPU内核都使用上，但fork()进程依然是昂贵的。启动多个进程只是为了充分将CPU资源利用起来。

* 创建子进程

child_process模块给予Node可以随意创建子进程能力，提供4个方法：

  * spawn()：启动一个子进程来执行命令。
  * exec()：启动一个子进程来执行命令，并且有一个回调函数获知子进程状况。
  * execFile()：启动一个子进程来执行可执行文件。
  * fork()：与spawn()类似，不同点在于它创建Node的子进程只需指定要执行的js文件模块即可。

spawn()与exec()、execFile()不同的是，后两者创建时可以指定timeout属性设置超时时间，一旦创建的进程运行超时设定的时间将会被杀死。

exec()与execFile()不同的是，exec()适合执行已有的命令，execFile()适合执行文件。举个示例代码worker.js，如下：

```js
var cp = require('child_process');
cp.spawn('node', ['worker.js']);
cp.exec('node worker.js', function (err, stdout, stderr) {
// some code
});
cp.execFile('worker.js', function (err, stdout, stderr) {
// some code
});
cp.fork('./worker.js');
```

以上4个方法在创建子进程之后均会返回子进程对象。差异对比：

<table cellspacing="0" cellpadding="0">
  <thead><td>类型</td><td>回调/异常</td><td>进程类型</td><td>执行类型</td><td>可设置超时</td></thead>
  <tbody>
      <tr><td>spawn()</td><td>不支持</td><td>任意</td><td>命令</td><td>不支持</td></tr>
      <tr><td>exec()</td><td>支持</td><td>任意</td><td>命令</td><td>支持</td></tr>
      <tr><td>execFile()</td><td>支持</td><td>任意</td><td>可执行文件</td><td>支持</td></tr>
      <tr><td>fork()</td><td>不支持</td><td>Node</td><td>JS文件</td><td>不支持</td></tr>
  </tbody>
</table>

这里的可执行文件是指可以直接执行的文件，如果是JS文件通过execFile()运行，它的首行内容必须添加如下代码：

```js
`#`!/usr/bin/env node
```

尽管4种创建子进程的方式有些差别，但事实上后面3种方法都是spawn()的延伸应用。

* 进程间通信

    在Master-Worker模式中，要实现主进程管理和调度工作进程的功能，需要主进程和工作进程之间的通信。对于child_process模块，创建好子进程，然后与父子进程间通信是十分容易的。

    在前端浏览器中，JS主线程与UI渲染共用同一个线程。执行JS的时候UI渲染是停滞的，渲染UI时，JS是停滞的，两者互相阻塞。长时间执行JS将会造成UI停顿不响应。为了解决这个问题，H5提供了WebWorker API。WebWorker允许创建工作线程并在后台运行，使得一些阻塞较为严重的计算不影响主线程上的UI渲染。它的API如下所示：

  ```js
  var worker = new Worker('worker.js');
  worker.onmessage = function(event) {
    document.getElementById("result").textContent = event.data;
  };
  ```

  其中，worker.js如下所示：

  ```js
  var n = 1;
  search: while(true) {
    n += 1;
    for (var i = 2; i <= Math.sqrt(n); i += 1) {
      if (n % i == 0) {
        continue search;
      }
    }
    postMessage(n);
  }
  ```

  主线程与工作线程之间通过onmessage()和postMessage()进行通信，子进程对象则由send()方法实现主进程向子进程发送数据，message事件实现收听子进程发来的数据，与API在一定程度上相似。通过消息传递内容，而不是共享或直接操作相关资源

  Node中对应示例：

  ```js
  // parent.js
  var cp = require('child_process');
  var n = cp.fork(__dirname + '/sub.js');
  n.on('message', function(m) {
    console.log('PARENT got message:', m);
  });
  n.send({hello: 'world'});

  // sub.js
  process.on('message', function(m) {
    console.log('CHILD got message:', m);
  });
  process.send({foo: 'bar'});
  ```

  通过fork()或者其他API，创建子进程之后，为了实现父子进程之间的通信，父进程与子进程之间将会创建IPC通道。通过IPC通道，父子进程之间才能通过message和send()传递消息。

  * 进程间通信原理

    IPC的全称是Inter-Process Communication，即进程间通信。进程间通信的目的是为了让不同的进程能够互相访问资源并进行协调工作。实现进程间通信的技术有很多，如命名管道、匿名管道、socket、信号量、共享内存、消息队列、Domain Socket等。Node中实现IPC通道的是管道（pipe）技术。在Node中管道是个抽象层面的称呼，具体细节实现由libuv提供，在windows下由命名管道（named pipe）实现，*nix系统则采用Unix Domain Socket实现。表现在应用层上的进程间通信只有简单的message事件和send()方法，接口十分简洁和消息化。

      ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-process/master/node-ipc.png)

    父进程在实际创建子进程之前，会创建IPC通道并监听它，然后才真正创建出子进程，并通过环境变量（NODE_CHANNEL_FD）告诉子进程这个IPC通道的文件描述符。子进程在启动的过程中，根据文件描述符去连接这个已存在的IPC通道，从而完成父子进程之间的连接。

    IPC管道步骤示意图：

      ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-process/master/node-ipc-pipe.png)

    建立连接之后的父子进程就可以自由地通信了。由于IPC通道是用命名管道或Domain Socket创建的，它们与网络socket的行为比较类似，属于双向通信。不同的是它们在系统内核中就完成了进程间的通信，而不用经过实际的网络层，非常高效。在Node中，IPC通道被抽象为Stream对象，在调用send()时发送数据（类似于write()），接收到的消息会通过message事件（类似于data）触发给应用层。

    注：只有启动的子进程是Node进程时，子进程才会根据环境变量去连接IPC通道，对于其他类型的子进程则无法实现进程间通信，除非其他进程也按约定去连接这个已经创建好的IPC通道。

* 句柄传递

> 建立好进程之间的IPC后，如果仅仅只用来发送一些简单的数据，显然不够我们的实际应用使用。如果让服务都监听到相同的端口，将会有什么结果？示例代码如下：

```js
var http = require('http');
http.createServer(function(req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
}).listen(8888, '127.0.0.1');
```

再次启动master.js文件，如下所示：

```js
events.js:72
  throw er; // Unhandled 'error' event

Error: listen EADDRINUSE
  at errnoException (net.js:884:11）
```

这时只有一个工作进程能够监听到该端口上，其余的进程在监听的过程中都抛出了EADDRINUSE异常，这是端口被占用的情况，新的进程不能继续监听该端口了。这个问题破坏了我们将多个进程监听同一个端口的想法，要想解决这个问题，通常的做法是让每个进程监听不同的端口，其中主进程监听主端口（如80），主进程对外接收所有的网络请求，再将这些请求分别代理到不同的端口的进程上。示意图如下：

  ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-process/master/node-master.png)

* 小结

### 集群稳定之路

* 进程事件
* 自动重启
* 负载均衡
* 状态共享

### Cluster模块

* Cluster工作原理
* Cluster事件

### 总结

### 参考资源