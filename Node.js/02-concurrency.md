# Event Loop

NodeJS事件循环可能是Node中最容易被误解的概念之一，不服来看一个🌰

```js
const fs = require('fs');
const setTimeOutlogger = ()=>{
    console.log('setTimeout logger');
}
const setImmediateLogger = ()=>{
    console.log('setImmediate logger');
}
//For timeout 
setTimeout(setTimeOutlogger, 1000);
//File I/O operation
fs.readFile(__filename, 'utf-8',(data)=>{
    console.log('Reading data 1');
});
fs.readFile(__filename, 'utf-8',(data)=>{
    console.log('Reading data 2');
});
fs.readFile(__filename, 'utf-8',(data)=>{
    console.log('Reading data 3');
});
fs.readFile(__filename, 'utf-8',(data)=>{
    console.log('Reading data 4');
});
fs.readFile(__filename, 'utf-8',(data)=>{
    console.log('Reading data 5');
});
//For setImmediate
setImmediate(setImmediateLogger);
setImmediate(setImmediateLogger);
setImmediate(setImmediateLogger);
```
输出结果是什么呢？根据文件的大小以及文件中的内容，结果可能会有所不同

当Node.js启动时，它会初始化Event Loop，处理输入的Script（可能会发出异步API调用），然后开始处理Event Loop

注意，只有一个线程，那就是Event Loop运行的线程。事件循环以周期顺序工作，具有不同的阶段。Event循环的操作顺序如下所示：

Event循环中有六个阶段，但有一个阶段只在内部工作。以下是Node.js文档中每个阶段的概述

> timers：此阶段执行由setTimeout()和setInterval()调度的回调

> I/O callbacks：执行几乎所有的回调函数，例如系统错误回调，除了关闭回调函数close callbacks，定时器的回调函数和setImmediate()

> idle, prepare：只在内部使用

> poll：获取新的I/O事件，适当时Node将在此处阻塞，当poll队列为空，此时如果有setImmediate调度，则继续到下一个check阶段，如果没有setImmediate，并且有timer就绪，则event循环将回卷到timers阶段

> check：setImmediate()回调在这里被调用

> close callbacks：关闭连接回调(connection callbacks)，例如socket.on('close', () => {})。 注意，如果socket或handle突然关闭(例如socket.destroy())，则在此阶段将发出'close'事件，否则它将通过process.nextTick()发出

## setTimeout vs setImmediate

```js
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```

```sh
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```

if you schedule the two calls within an I/O cycle, the immediate callback is always executed first:

```js
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

```sh
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```

**setTimeout(fn,0)**

回调函数调用直到事件循环进入timer阶段才会执行。因此，当你在close callbacks阶段中同时调度setTimeout(fn,0)和setImmediate()，将保证在setImmediate()之前执行setTimeout(fn,0)

**setImmediate()**

首先，通过事件循环的工作流程，现在我们可以说setImmediate()并不是立即执行的，但包含此setImmediate回调的队列将在每次迭代中执行一次(当事件循环处于check阶段时)。所以这是非确定性的，因为它取决于process性能。但是，如果我们在I/O回调阶段中执行这段代码，我们可以保证在setTimeout之前调用setImmediate的回调

## process.nextTick

process.nextTick()狠有趣，它与事件循环的当前阶段无关，将在当前操作完成后立即开始执行。因此，如果事件循环处于timers中，并且timer队列中有5个回调，事件循环正忙于执行第三个。那时，如果有一些process.nextTick()回调被推送到nextTickQueue，则事件循环将在完成当前回调执行（即第三个回调）之后同步执行所有这些回调，并且将从第4个回调再次恢复执行timer回调

为什么process.nextTick被包含在Node.js中? 其实是一种设计理念，即即使不需要，API也应该始终是异步的

```js
function apiCall(arg, callback) {
  if (typeof arg !== 'string')
    return process.nextTick(callback,
                            new TypeError('argument should be string'));
}
```

```js
let bar;

function someAsyncApiCall(callback) { callback(); }

someAsyncApiCall(() => {
  console.log('bar', bar); // undefined
});

bar = 1;
```

```js
let bar;

function someAsyncApiCall(callback) {
  process.nextTick(callback);
}

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

bar = 1;
```

```js
const server = net.createServer(() => {}).listen(8080);

server.on('listening', () => {});
```

# Quiz

```js

// quiz-1
(function test() {
    setTimeout(function() {console.log(4)}, 0);
    new Promise(function executor(resolve) {
        console.log(1);
        for( var i=0 ; i<10000 ; i++ ) {
            i == 9999 && resolve();
        }
        console.log(2);
    }).then(function() {
        console.log(5);
    });
    console.log(3);
})();



// quiz-2
console.log('golb1');

setImmediate(function() {
    console.log('immediate1');
    process.nextTick(function() {
        console.log('immediate1_nextTick');
    })
    new Promise(function(resolve) {
        console.log('immediate1_promise');
        resolve();
    }).then(function() {
        console.log('immediate1_then')
    })
});

setTimeout(function() {
    console.log('timeout1');
    process.nextTick(function() {
        console.log('timeout1_nextTick');
    })
    new Promise(function(resolve) {
        console.log('timeout1_promise');
        resolve();
    }).then(function() {
        console.log('timeout1_then')
    })
    setTimeout(function() {
      console.log('timeout1_timeout1');
    process.nextTick(function() {
        console.log('timeout1_timeout1_nextTick');
    })
    setImmediate(function() {
      console.log('timeout1_setImmediate1');
    })
    });
});

new Promise(function(resolve) {
    console.log('glob1_promise');
    resolve();
}).then(function() {
    console.log('glob1_then')
});

process.nextTick(function() {
    console.log('glob1_nextTick');
});

```

# Tips

I/O用于标示CPU中的进程与CPU外部（包括内存/磁盘网络，甚至是另一个进程）之间的通信，在Node中这通常用于指磁盘和网络资源

事件循环是处理外部事件并将其转换为回调函数调用的实体

当没有更多事件要执行时，Node将退出事件循环

V8调用堆栈可以看作是函数的列表，堆栈是FILO(First In Last Out)数据结构，当调用堆栈变空时，event队列不为空，如果有回调函数则将队列出队，然后调用事件的回调

一个函数应该始终是同步或异步

# Reference

[Event Loop Bible](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
