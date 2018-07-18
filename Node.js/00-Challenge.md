# Node.js challenge

> What is the relationship between Node.js and V8? Can Node work without V8?

> How come when you declare a global variable in any Node.js file it’s not really global to all modules?

> When exporting the API of a Node module, why can we sometimes use exports and other times we have to use module.exports?

> Can we require local files without using relative paths?

> Can different versions of the same package be used in the same application?

> What is the Event Loop? Is it part of V8?

> What is the Call Stack? Is it part of V8?

> What is the difference between setImmediate and process.nextTick?

> How do you make an asynchronous function return a value?

> Can callbacks be used with promises or is it one way or the other?

> What Node module is implemented by most other Node modules?

> What are the major differences between spawn, exec, and fork?

> How does the cluster module work? How is it different than using a load balancer?

> What are the --harmony-* flags?

> How can you read and inspect the memory usage of a Node.js process?

> What will Node do when both the call stack and the event loop queue are empty?

> What are V8 object and function templates?

> What is libuv and how does Node.js use it?

> How can you make Node’s REPL always use JavaScript strict mode?

> What is process.argv? What type of data does it hold?

> How can we do one final operation before a Node process exits? Can that operation be done asynchronously?

> What are some of the built-in dot commands that you can use in Node’s REPL?

> Besides V8 and libuv, what other external dependencies does Node have?

> What’s the problem with the process uncaughtException event? How is it different than the exit event?

> What does the _ mean inside of Node’s REPL?

> Do Node buffers use V8 memory? Can they be resized?

> What’s the difference between Buffer.alloc and Buffer.allocUnsafe?

> How is the slice method on buffers different from that on arrays?

> What is the string_decoder module useful for? How is it different than casting buffers to strings?

> What are the 5 major steps that the require function does?

> How can you check for the existence of a local module?

> What is the main property in package.json useful for?

> What are circular modular dependencies in Node and how can they be avoided?

> What are the 3 file extensions that will be automatically tried by the require function?

> When creating an http server and writing a response for a request, why is the end() function required?

> When is it ok to use the file system *Sync methods?

> How can you print only one level of a deeply nested object?

> What is the node-gyp package used for?

> The objects exports, require, and module are all globally available in every module but they are different in every module. How?

> If you execute a node script file that has the single line: console.log(arguments);, what exactly will node print?

> How can a module be both requirable by other modules and executable directly using the node command?

> What’s an example of a built-in stream in Node that is both readable and writable?

> What happens when the line cluster.fork() gets executed in a Node script?

> What’s the difference between using event emitters and using simple callback functions to allow for asynchronous handling of code?

> What is the console.time function useful for?

> What’s the difference between the Paused and the Flowing modes of readable streams?

> What does the --inspect argument do for the node command?

> How can you read data from a connected socket?

> The require function always caches the module it requires. What can you do if you need to execute the code in a required module many times?

> When working with streams, when do you use the pipe function and when do you use events? Can those two methods be combined?
