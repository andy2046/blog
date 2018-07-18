# Node's Architecture
Node架构中最重要的两个部分是`V8`和`libuv`，`V8`是Node默认的VM引擎，另外一个VM选项是来自微软的[Chakra](https://github.com/Microsoft/ChakraCore)，[node-chakracore](https://github.com/nodejs/node-chakracore)致力于在Chakra引擎上运行Node
## V8
`V8`引擎上面的feature有三个不同的group，`Shipping`／`Staged`／`In Progress`，默认情况下Shipping group的feature可以直接使用，Staged group的feature需要使用`--harmony`选项来开启，In Progress group的feature不稳定，但你也可以使用特定的flag来开启，示例如下：

`padEnd`是Staged group的feature，需要使用`--harmony`选项来开启
```sh
➜ node -v
v7.9.0
➜ node -p 'process.versions.v8'
5.5.372.43
➜ node -p "'Node'.padEnd(8, '*')"
[eval]:1
'Node'.padEnd(8, '*')
       ^

TypeError: "Node".padEnd is not a function
    at [eval]:1:8
    at ContextifyScript.Script.runInThisContext (vm.js:23:33)
    at Object.runInThisContext (vm.js:95:38)
    at Object.<anonymous> ([eval]-wrapper:6:22)
    at Module._compile (module.js:571:32)
    at evalScript (bootstrap_node.js:387:27)
    at run (bootstrap_node.js:120:11)
    at run (bootstrap_node.js:423:7)
    at startup (bootstrap_node.js:119:9)
    at bootstrap_node.js:538:3
➜ node --harmony -p "'Node'.padEnd(8, '*')"
Node****
```
`trailing commas`是In Progress group的feature，需要`--harmony_trailing_commas`来开启
```sh
➜ node --v8-options | grep "in progress"
  --harmony_array_prototype_values (enable "harmony Array.prototype.values" (in progress))
  --harmony_function_sent (enable "harmony function.sent" (in progress))
  --harmony_sharedarraybuffer (enable "harmony sharedarraybuffer" (in progress))
  --harmony_simd (enable "harmony simd" (in progress))
  --harmony_do_expressions (enable "harmony do-expressions" (in progress))
  --harmony_restrictive_generators (enable "harmony restrictions on generator declarations" (in progress))
  --harmony_regexp_named_captures (enable "harmony regexp named captures" (in progress))
  --harmony_regexp_property (enable "harmony unicode regexp property classes" (in progress))
  --harmony_for_in (enable "harmony for-in syntax" (in progress))
  --harmony_trailing_commas (enable "harmony trailing commas in function parameter lists" (in progress))
  --harmony_class_fields (enable "harmony public fields in class literals" (in progress))
➜ node -p 'function tc(a,b,) {}'
[eval]:1
function tc(a,b,) {}
                ^
SyntaxError: Unexpected token )
    at createScript (vm.js:53:10)
    at Object.runInThisContext (vm.js:95:10)
    at Object.<anonymous> ([eval]-wrapper:6:22)
    at Module._compile (module.js:571:32)
    at evalScript (bootstrap_node.js:387:27)
    at run (bootstrap_node.js:120:11)
    at run (bootstrap_node.js:423:7)
    at startup (bootstrap_node.js:119:9)
    at bootstrap_node.js:538:3
➜ node --harmony_trailing_commas -p 'function tc(a,b,) {}'
undefined
```

你也可以使用如下command来查看Node的可用选项，比如gc选项：
```sh
➜ node --v8-options | grep gc                    

  --expose_gc (expose gc extension)
  --expose_gc_as (expose gc extension under the specified name)
  --gc_global (always perform global GCs)
  --gc_interval (garbage collect after <n> allocations)
  --retain_maps_for_n_gc (keeps maps alive for <n> old space garbage collections)
  --trace_gc (print one trace line following each garbage collection)
  --trace_gc_nvp (print one detailed trace line in name=value format after each garbage collection)
  --trace_gc_ignore_scavenger (do not print trace line after scavenger collection)
  --trace_gc_verbose (print more details following each garbage collection)
  --trace_mutator_utilization (print mutator utilization, allocation speed, gc speed)
  --track_gc_object_stats (track object counts and memory usage)
  --trace_gc_object_stats (trace object counts and memory usage)
  --cleanup_code_caches_at_gc (Flush inline caches prior to mark compact collection and flush code caches in maps during mark compact cycle.)
  --log_gc (Log heap samples on garbage collection for the hp2ps tool.)
  --gc_fake_mmap (Specify the name of the file for fake gc mmap used in ll_prof)
        type: string  default: /tmp/__v8_gc__
```

## libuv
Node自身包含`Core Modules`，用于调用filesystem／network／timer等Node API，这些API会调用`C++ Bindings`里面的c++代码

`libuv`是一个C library，Node使用libuv来处理异步操作，比如非阻塞IO（file system／TCP socket／child process），当异步操作完成时，通过callback返回给V8

# REPL (Read–Eval–Print Loop)
你可以在terminal里面执行`node`来启动CLI，如下所示，REPL十分方便

例如，你定义一个array，当你`arr.`然后tab-tab(tab两次)，array自身的方法会显示出来

```sh
➜ node
> var arr = [];
undefined
> arr.
arr.toString              arr.valueOf
arr.concat                arr.copyWithin         arr.entries               arr.every              arr.fill                  arr.filter
arr.find                  arr.findIndex          arr.forEach               arr.includes           arr.indexOf               arr.join
arr.keys                  arr.lastIndexOf        arr.length                arr.map                arr.pop                   arr.push
arr.reduce                arr.reduceRight        arr.reverse               arr.shift              arr.slice                 arr.some
arr.sort                  arr.splice             arr.unshift               
```
你也可以输入`.help`，然后可以看到各种快捷键如下：
```sh
> .help
.break    Sometimes you get stuck, this gets you out
.clear    Alias for .break
.editor   Enter editor mode
.exit     Exit the repl
.help     Print this help message
.load     Load JS from a file into the REPL session
.save     Save all evaluated commands in this REPL session to a file
```
你还可以用`_`(underscore)来得到上次evaluated的值：
```sh
> 3 - 2
1
> _
1
> 3 < 2
false
> _
false
```
你还可以自定义REPL选项，如下，你自定义`repl.js`并选择忽视undefined，这样output里面就不会有undefined输出，同时你还可以预先加载你需要的library比如lodash
```js
// repl.js
let repl = require('repl');
let r = repl.start({ ignoreUndefined: true  });
r.context.lodash = require('lodash');
```
```sh
➜ node ~/repl.js
> var i = 2;
> 
> 
```
你可以用下面的command来查看更多的选项
`node --help | less`
```sh
-p, --print     evaluate script and print result

-c, --check     syntax check script without executing

-r, --require   module to preload (option can be repeated)
```

例如，`node -c bad-syntax.js`可以用来检查语法错误，
`node -p 'os.cpus()'`可以用来执行script并输出结果，你还可以传入参数，如下所示
```sh
➜ node -p 'process.argv.slice(1)' test 666
[ 'test', '666' ]
```
`node -r babel-core/register`可以用来预加载，相当于`require('babel-core/register')`

# global ／ process
`global`相当于浏览器里面的`window`，你可以`global.a = 1;`这样`a`就是全局变量，但一般不推荐这样做

`process`是application和running env之间的桥梁，可以得到运行环境相关信息，如下所示：
```sh
> process.
process.arch
process.argv
process.argv0                       process.assert                      process.binding                     
process.chdir
process.config                      process.cpuUsage                    
process.cwd                         process.debugPort
process.dlopen                      process.emitWarning                 
process.env                         process.execArgv
process.execPath                    
process.exit                        process.features                    process.getegid
process.geteuid                     process.getgid                      process.getgroups                   process.getuid
process.hrtime                      process.initgroups                  
process.kill                        process.memoryUsage
process.moduleLoadList              process.nextTick                    process.openStdin                   
process.pid
process.platform                    process.reallyExit                  process.release                     process.setegid
process.seteuid                     process.setgid                      process.setgroups                   process.setuid
process.stderr                      process.stdin                       process.stdout                      
process.title
process.umask                       process.uptime                      process.version                     process.versions
process._events                     process._maxListeners               process.addListener                 process.domain
process.emit                        process.eventNames                  process.getMaxListeners             process.listenerCount
process.listeners                   
process.on                          
process.once                        process.prependListener
process.prependOnceListener         process.removeAllListeners          process.removeListener              process.setMaxListeners
```

同时可以看到，`process`也是一个`event emitter`，例如：
```js
process.on('exit', code => {
  console.log(code)
})

process.on('uncaughtException', err => {
  console.error(err)
  process.exit(1)
})
```

# require module
`require` module用来require单个module，`Module` module用于管理我们required的modules

当我们require一个module时，整个过程有五个步骤：
> `Resolving` 找到module的绝对文件路径

> `Loading` 将文件内容加载到内存

> `Wrapping` 给每个module创造一个private scope并确保require对每个module来说是local变量

> `Evaluating` VM执行module代码

> `Caching` 缓存module以备下次使用

如下所示，Node会遍历paths里面每一个path去查找module，找不到会报错，对于core modules，`Resolving`会立刻返回

```js
Module {
  id: '.',
  exports: {},
  parent: undefined,
  filename: '/Users/xxx/lib/find.js',
  loaded: false,
  children: [],
  paths: 
   [ '/Users/xxx/lib/node_modules',
     '/Users/xxx/node_modules',
     '/Users/node_modules',
     '/node_modules' ] }
```

在Module对象里面，`id` 是module的identity，通常它的值是module文件的全路径，除非是root，这时它的值是`.`(dot)

`filename` 是文件的路径

`paths` 从当前路径开始，往上一直到根路径

`require.resolve` 和require一样，但是它不会加载文件，只是resolve

module也可以是一个包含`index.js`文件的路径，你还可以用`package.json`来改变文件名称，如下所示，将会查找`start.js`而不是`index.js`

```js
{
    "name": "find-me",
    "main": "start.js"
}
```

在任何module中，`exports`是一个特殊对象，我们放入它的任何变量都可以在require时得到

Module对象的`loaded`属性会保持false，直到所有content都被加载

Node.js可以处理circular require的情况，例如A require B，B require A

我们也可以require JSON文件和C++ Addons，Node会首先查找`.js`文件，再查找`.json`文件，最后`.node`文件

你可以通过`require.extensions`来查看Node支持的文件扩展名:
```js
> require.extensions
{ '.js': [Function], '.json': [Function], '.node': [Function] }
```

```js
exports.id = 1; // this is ok
exports = { id: 1 }; // this is not ok
module.exports = { id: 1 }; // this is ok
// because exports = module.exports
var g = 2; // local to this file
```
为什么👆上面的代码，第二行会有问题呢？

Node在compile代码之前，在`Wrapping`那一步，Node把module代码包裹在一个函数里面，这个函数有五个参数，正是这个函数使得`module`/`exports`/`require`看起来是全局的但实际上是specific to each module

你可以用`require('module').wrapper`来查看这个函数和它的参数

```js
> require('module').wrapper
[ '(function (exports, require, module, __filename, __dirname) { ',
  '\n});' ]
```

当你执行一个script文件时，`require.main` 等于 `module`

当我们require时，required文件存在于`require.cache`数组中，这样Node下一次不会再次加载同样的文件

