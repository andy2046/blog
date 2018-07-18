# callback

> Call me when you’re ready, Node

事件驱动的最简单形式是回调函数，例如fs.readFile(file, cb)，在这种形式下，当Node就绪时，事件会发动一次，回调函数当作事件处理器被调用

回调函数只是你传递给其他函数的函数，这狠容易因为在JavaScript中函数是一等公民

记住，回调不代表代码中的异步调用，一个函数可以同步或者异步地调用回调函数

例如，这个函数fileSize接受一个回调函数cb并且可以同步或者异步调用它：

```js
function fileSize (fileName, cb) {
  if (typeof fileName !== 'string') {
    return cb(new TypeError('argument should be string')); // Sync
  }
  fs.stat(fileName, (err, stats) => {
    if (err) { return cb(err); } // Async
    cb(null, stats.size); // Async
  });
}
```

请注意，这不是一个好的做法，会导致意外的错误。函数应该总是同步或总是异步消费回调函数。

下面是一个用回调风格编写的典型异步Node函数的简单示例：

```js
const readFileAsArray = function(file, cb) {
  fs.readFile(file, function(err, data) {
    if (err) {
      return cb(err);
    }
    const lines = data.toString().trim().split('\n');
    cb(null, lines);
  });
};

readFileAsArray('./numbers.txt', (err, lines) => {
  if (err) throw err;
  const numbers = lines.map(Number);
  const oddNumbers = numbers.filter(n => n%2 === 1);
  console.log('Odd numbers count:', oddNumbers.length);
});
```

记住，你应该在主函数中把回调函数作为最后的参数传递，并在回调函数中把error作为第一个参数传递

在现代JavaScript中，我们有Promise对象。Promise可以替代异步API的回调。与其将回调作为参数传递并在同一处处理错误，Promise对象允许我们分别处理成功和错误情况，并且还允许我们链接多个异步调用，而不是嵌套它们，示例如下：

```js
const readFileAsArray = function(file, cb = () => {}) {
  return new Promise((resolve, reject) => {
    fs.readFile(file, function(err, data) {
      if (err) {
        reject(err);
        return cb(err);
      }
      const lines = data.toString().trim().split('\n');
      resolve(lines);
      cb(null, lines);
    });
  });
};

readFileAsArray('./numbers.txt')
  .then(lines => {
    const numbers = lines.map(Number);
    const oddNumbers = numbers.filter(n => n%2 === 1);
    console.log('Odd numbers count:', oddNumbers.length);
  })
  .catch(console.error);
```

一个使用异步代码的更好的替代方法是使用异步函数，它允许我们将异步代码视为同步代码，从而使整体更具可读性。

```js
async function countOdd () {
  try {
    const lines = await readFileAsArray('./numbers');
    const numbers = lines.map(Number);
    const oddCount = numbers.filter(n => n%2 === 1).length;
    console.log('Odd numbers count:', oddCount);
  } catch(err) {
    console.error(err);
  }
}

countOdd();
```

示例中首先创建一个异步函数，它只是一个正常的函数，在它之前有async关键字。在异步函数内部，调用readFileAsArray函数，它返回行变量，为了实现这个功能，我们使用关键字await。之后继续执行代码，就好像readFileAsArray调用是同步的

这样执行异步函数非常简单，更具可读性。要处理错误，则需要将异步调用包装在try/catch语句中

有了这个async/await功能，不必使用任何特殊的API (如.then和.catch)

可以使用支持Promise接口的任何函数的async/await功能。但是，不能将其与回调式异步函数（例如setTimeout）一起使用

# Event Emitter

EventEmitter是一个便于Node中对象间通信的模块。EventEmitter是Node异步事件驱动架构的核心。许多Node的内置模块继承自EventEmitter

它的概念很简单：Emitter对象发出命名事件，导致先前注册的侦听器被调用。所以，一个Emitter对象基本上有两个主要特点：

> 发送命名事件

> 注册和取消注册监听器

要使用EventEmitter，只需创建一个扩展EventEmitter的类

```js
class MyEmitter extends EventEmitter {
    // todo
}

const myEmitter = new MyEmitter();

myEmitter.emit('something-happened');
```

发出事件是发生某种情况的信号。这种情况通常是关于Emitter对象的状态变化

可以使用on方法添加侦听器函数，并且每当emitter对象发出关联的命名事件时，都会执行这些侦听器函数

## Events !== Asynchrony
先来看一个🌰

```js
const EventEmitter = require('events');

class WithLog extends EventEmitter {
  execute(taskFunc) {
    console.log('Before executing');
    this.emit('begin');
    taskFunc();
    this.emit('end');
    console.log('After executing');
  }
}

const withLog = new WithLog();
withLog.on('begin', () => console.log('About to execute'));
withLog.on('end', () => console.log('Done with execute'));

withLog.execute(() => console.log('*** Executing task ***'));
// Before executing
// About to execute
// *** Executing task ***
// Done with execute
// After executing
```

这一切都是同步发生的，这段代码没有任何异步

就像普通的回调一样，不要认为事件意味着同步代码或异步代码

这很重要，因为如果传递的是一个异步taskFunc来执行，发出的事件将不再准确，例如：
```js
withLog.execute(() => {
  setImmediate(() => {
    console.log('*** Executing task ***')
  });
});

// Before executing
// About to execute
// Done with execute
// After executing
// *** Executing task ***
```

要在异步函数完成后发出事件，我们需要将回调（或Promise）与基于事件的通信结合起来

使用事件而不是常规回调的一个好处是可以通过定义多个侦听器多次对相同的信号作出反应。为了在回调中实现同样的效果，你必须在单个可用的回调中编写更多的逻辑。事件是应用程序允许多个外部插件在应用程序核心之上构建功能的好方法。你可以将它们想象成钩点hook points以允许定制关于状态变化的故事

## Asynchronous Events
再来看一个🌰
```js
const fs = require('fs');
const EventEmitter = require('events');

class WithTime extends EventEmitter {
  execute(asyncFunc, ...args) {
    this.emit('begin');
    console.time('execute');
    asyncFunc(...args, (err, data) => {
      if (err) {
        return this.emit('error', err);
      }

      this.emit('data', data);
      console.timeEnd('execute');
      this.emit('end');
    });
  }
}

const withTime = new WithTime();
withTime.on('begin', () => console.log('About to execute'));
withTime.on('end', () => console.log('Done with execute'));
withTime.on('data', (data) => {
  // do something with data
});

withTime.execute(fs.readFile, __filename);
```
error事件通常是一个特殊的事件。在基于回调的例子中，如果我们不用监听器处理错误事件，节点进程将实际退出
```js
withTime.on('error', (err) => {
  console.log(err)
});
```
如果我们这样做，将会报告第一次执行调用的错误，但节点进程不会崩溃并退出，另一个执行调用将正常完成

处理发出的错误的异常的另一种方法是为全局uncaughtException进程事件注册侦听器。但是，通过该事件捕捉全局错误是一个坏主意
```js
process.on('uncaughtException', (err) => {
  console.error(err); // don't do just that.
  // FORCE exit the process too.
  process.exit(1);
});
```

如果asynFunc支持Promise，我们可以使用async/await功能来执行相同的操作：
```js
class WithTime extends EventEmitter {
  async execute(asyncFunc, ...args) {
    this.emit('begin');
    try {
      console.time('execute');
      const data = await asyncFunc(...args);
      this.emit('data', data);
      console.timeEnd('execute');
      this.emit('end');
    } catch(err) {
      this.emit('error', err);
    }
  }
}
```

## Order of Listeners
如果我们为同一事件注册多个侦听器，那么这些侦听器的调用将按顺序进行。我们注册的第一个侦听器是第一个被调用的侦听器
