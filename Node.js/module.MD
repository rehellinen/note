#### 参考文章
1. https://segmentfault.com/a/1190000023828613

#### 简单介绍
1. 使用CommonJS规范，使用module.exports导出，使用require引入
2. 当require某个模块时，并不是只拿他的module.exports，而是会从头开始运行这个文件
3. 支持.js / .json(会自动调用JSON.parse) / .node(C++编译后的二进制文件)

#### 加载顺序
1. 优先加载内置模块，即使有同名文件，也会优先使用内置模块。
2. 不是内置模块，先去缓存找。
3. 缓存没有就去找对应路径的文件。
4. 不存在对应的文件，就将这个路径作为文件夹加载。  
文件夹加载规则：  
（1）先看看这个文件夹下面有没有package.json，如果有就找里面的main字段，main字段有值就加载对应的文件。  
（2）如果没有package.json或者package.json里面没有main就找index文件。
如果这两步都找不到就报错了。
5. 对应的文件和文件夹都找不到就去node_modules下面找。
6. 还找不到就报错了。

#### 循环引用
a.js:
```
console.log('a 开始');
exports.done = false;
const b = require('./b.js');
console.log('在 a 中，b.done = %j', b.done);
exports.done = true;
console.log('a 结束');
```
b.js:
```
console.log('b 开始');
exports.done = false;
const a = require('./a.js');
console.log('在 b 中，a.done = %j', a.done);
exports.done = true;
console.log('b 结束');
```
main.js:
```
console.log('main 开始');
const a = require('./a.js');
const b = require('./b.js');
console.log('在 main 中，a.done=%j，b.done=%j', a.done, b.done);
```
当main.js加载a.js时，a.js又加载b.js。此时，b.js会尝试去加载a.js。
为了防止无限的循环，会返回一个a.js的exports对象的未完成的副本给b.js模块。然后b.js完成加载，并将exports对象提供给a.js模块。

那么这个效果是怎么实现的呢？答案就在我们的MyModule._load源码里面，注意这两行代码的顺序:
```
MyModule._cache[filename] = module;
module.load(filename);
```
上述代码中我们是先将缓存设置了，然后再执行的真正的load，顺着这个思路我能来理一下这里的加载流程:
1. main加载a，a在真正加载前先去缓存中占一个位置
2. a在正式加载时加载了b
3. b又去加载了a，这时候缓存中已经有a了，所以直接返回a.exports，即使这时候的exports是不完整的。

