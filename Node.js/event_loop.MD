#### 参考文章
1. https://zhuanlan.zhihu.com/p/35039878
2. https://zhuanlan.zhihu.com/p/35918797

#### 浏览器事件循环：
同步任务执行完之后，会把所有micro task执行完，接着执行一个macro task
1. macro task: setInterval、setTimeout
2. micro task: Promise.then、Mutation Observer

#### Node.js事件循环：
先调用uv__loop_alive判断loop中是否还有异步任务，若没有则直接跳出循环  
事件循环是一个while循环，当loop中还有异步任务，循环就会不停地运行，循环做的事情有：  
1. update_time: 获取一下系统时间，以保证之后的timer有个计时的标准
2. timers: 使用最小堆存储，检查是否有到期的timer
3. I/O callbacks: 处理异步事件的回调，比如网络I/O，比如文件读取I/O。当这些I/O动作都结束的时候，在这个阶段会触发它们的回调。
4. idle, prepare: 这个阶段内部做一些动作，与理解事件循环没啥关系
5. I/O poll: 这个阶段是选择运行的，且会获得了一个最多是32767的一个等待时间，其实只要有任务需要马上执行的时候，
这个函数是不会被调用的。那么被调用的时候则是所有被注册的异步任务都没有完成（返回）的时候，
等的就是这些异步任务会不会在这么极其短暂的时间内发生I/O完毕。
6. check: 执行setImmediate操作
7. close callbacks: 关闭I/O的动作，比如文件描述符的关闭，链接断开，等等等

除了上述的过程之外，还有额外的nextTickQueue和microTaskQueue。
这两个队列追加在本轮循环，即同步任务一旦执行完成，就开始执行它们，而setTimeout、setInterval、setImmediate的回调函数，追加在次轮循环。
因此process.nextTick和Promise的回调函数优先级较高。
