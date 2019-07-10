## 背景

通常在开发一个项目的时候，总会有不少场景需要创建定时器，这会导致项目中出现很多重复的代码。为了解决这个问题，不妨构建一个Ticker来维护整个项目的时间线。

先用一个思维导图来理清思路：

![思维导图](https://github.com/yilishabuBai/myblog/blob/master/images/ticker/mind.png)

## 特点

**全局使用**

当Ticker需要作用在整个项目中时，最好的设计模式就是单例。为了使用方便，结合静态方法构造Ticker的雏形。

```ts
export class Ticker {
    static _ticker: Ticker = new Ticker();
    
    _sayHello () {
        console.log('Hello.');
    }
    
    static sayHello () {
        this._ticker._sayHello();
    }
}
```

***方法友好便于维护***

采用单例模式结合静态方法，就可以达到通过`类名.方法`的方式直接调用实例中的方法，例如执行`Ticker.sayHello()`，会得到控制台打印`Hello.`的结果。

***定时器的隐式问题***

相信很多人都对`setTimeout()`和`setInterval()`非常熟悉，但不是所有人都会它们的运行机制有了解。

定时器会把方法放入异步队列，哪怕第二个参数设置为0。异步队列中的方法会在同步队列执行完毕之后才会执行。

这里简单介绍一下`setTimeout()`与`setInterval()`。`setTimeout()`入参中的delay是仅仅是等待时间，而`setInterval()`入参中的delay还包括了执行时间，也就是说同样设置delay，`setTimeout()`间隔会略长于`setInterval()`。同时，现代浏览器对`setInterval()`有一个优化，就是当主线程阻塞时，浏览器只会保持`setInterval()`回调方法队列中仅存在一个待执行方法，而不会像之前出现连续执行若干次回调方法的情况。

另外，考虑到上述`setInterval()`特性，在主线程阻塞的情况下会有机会出现两次回调函数结束时间小于delay的情况，为了避免这种情况可能导致的问题，采用链式setTimeout调用来维护时间线。

> 之前对这里描述比较模糊，感谢[碎碎酱](https://juejin.im/user/582a78d02e958a0069a5e52c)提醒。

## 属性与方法

简单粗暴一点，直接上代码，并在代码中逐步解释作用。

```ts
/*
 * @Author: 伊丽莎不白 
 * @Date: 2019-07-05 17:17:30 
 * @Last Modified by: 伊丽莎不白
 * @Last Modified time: 2019-07-10 15:01:30
 */
export class Ticker {
    static _ticker: Ticker = new Ticker();
    
    _running: boolean = false;  // 正在运行
    _systemTime: number = 0;    // 系统时间
    _lastTime: number = 0;  // 上次执行时间
    _timerId: NodeJS.Timeout = null;    // 计时器id
    _delay: number = 33;    // 延时设定
    _funcs: Array<Function> = [];   // 钩子函数队列
    _executeFuncs: Array<ExecuteValue> = [];   // 定时执行函数队列，按执行时间升序排序

    constructor () {
    }

    /**
     * 查找第一个大于目标值的值的下标
     * @param time 
     */
    _searchIndex (time: number) {
        let funcs: Array<ExecuteValue> = this._executeFuncs;
        let low: number = 0;
        let high: number = funcs.length;
        let mid: number = 0;
        while (low < high) {
            mid = Math.floor(low + (high - low) / 2);
            if (time >= funcs[mid].time) {
                low = mid + 1;
            } else {
                high = mid - 1;
            }
        }
        return low;
    }

    /**
     * 注册钩子函数
     * @param func 执行函数
     */
    _register (func: Function) {
        if (this._funcs.includes(func)) {
            return;
        }
        this._funcs.push(func);
    }

    /**
     * 注册一个函数，在一段时间之后执行
     * @param func 执行函数
     * @param delay 延时
     * @param time 执行时系统时间
     * @param loop 循环次数
     */
    _registerDelay (func: Function, delay: number, time: number, loop: number) {
        // 先查找后插入
        let index: number = this._searchIndex(time);
        let value: ExecuteValue = { func: func, time: time, delay: delay, loop: loop };
        this._executeFuncs.splice(index, 0, value);
    }

    /**
     * 注册一个函数，在某个时间点执行
     * @param func 执行函数
     * @param time 执行时间
     */
    _registerTimer (func: Function, time: number) {
        // 先查找后插入
        let index: number = this._searchIndex(time);
        let value: ExecuteValue = { func: func, time: time };
        this._executeFuncs.splice(index, 0, value);
    }

    /**
     * 移除钩子函数
     * @param func 执行函数
     */
    _unregister (func: Function) {
        this._funcs.map((value: Function, index: number) => {
            if (func === value) {
                this._funcs.splice(index, 1);
            }
        });
    }

    /**
     * 启动Ticker，并设置当前系统时间，通常与服务器时间同步
     * @param systemTime 系统时间
     */
    _start (systemTime: number = 0) {
        if (this._running) {
            return;
        }
        this._running = true;
        this._systemTime = systemTime;
        this._lastTime = new Date().getTime();
        this._update();
    }

    /**
     * 链式执行定时器，钩子函数队列为每次调用必执行，定时执行函数队列为系统时间大于执行时间时调用并移出队列
     */
    _update () {
        let currentTime: number = new Date().getTime();
        let delay: number = currentTime - this._lastTime;
        this._systemTime += delay;
        // 钩子函数队列，依次执行即可
        this._funcs.forEach((value: Function) => {
            value(delay);
        });

        this._executeFunc();
        
        this._lastTime = currentTime;
        this._timerId = setTimeout(this._update.bind(this), this._delay);
    }

    /**
     * 执行定时函数
     */
    _executeFunc () {
        // 取数组首项进行时间校验
        if (this._executeFuncs[0] && this._executeFuncs[0].time < this._systemTime) {
            // 取出数组首项并执行
            let value: ExecuteValue = this._executeFuncs.shift();
            value.func();

            // 递归执行下一项
            this._executeFunc();
            
            // 判断重复执行次数
            if (value.hasOwnProperty('loop')) {
                if (value.loop > 0 && --value.loop === 0) {
                    return;
                }
                // 计算下次执行时间，插入队列
                let fixTime: number = value.time + value.delay;
                this._registerDelay(value.func, value.delay, fixTime, value.loop);
            }
        }
    }

    /**
     * 停止Ticker
     */
    _stop () {
        if (this._timerId) {
            clearTimeout(this._timerId);
            this._timerId = null;
        }
        this._running = false;
    }
    
    /**
     * 公开的钩子函数注册方法
     * @param func 执行函数
     */
    static register (func: Function) {
        this._ticker._register(func);
    }

    /**
     * 公开的钩子函数移除方法
     * @param func 执行函数
     */
    static unregister (func: Function) {
        this._ticker._unregister(func);
    }

    /**
     * 公开的延时执行函数方法，用户可设置执行次数，loop为0时无限循环
     * @param func 执行函数
     * @param delay 延时
     * @param loop 循环次数
     */
    static registerDelay (func: Function, delay: number, loop: number = 1) {
        let time: number = this._ticker._systemTime + delay;
        this._ticker._registerDelay(func, delay, time, loop);
    }

    /**
     * 公开的定时执行函数方法
     * @param func 执行函数
     * @param time 执行时间
     */
    static registerTimer (func: Function, time: number) {
        this._ticker._registerTimer(func, time);
    }

    /**
     * 公开的启动方法
     * @param systemTime 系统时间
     */
    static start (systemTime: number = 0) {
        this._ticker._start(systemTime);
    }

    /**
     * 公开的停止方法
     */
    static stop () {
        this._ticker._stop();
    }

    /**
     * 系统时间
     */
    static get systemTime (): number {
        return this._ticker._systemTime;
    }

    /**
     * 正在运行
     */
    static get running (): boolean {
        return this._ticker._running;
    }
}

interface ExecuteValue {
    func: Function;
    time: number;
    delay?: number;
    loop?: number;
}
```

***如何使用***

建议在项目启动的时候执行`Ticker.start()`方法，此时Ticker中的systemTime将从0开始计时；或者在获取到服务器时间之后执行并传入服务器时间`Ticker.start(serverTime)`，这样项目将会在项目中维持服务器时间线。如果是需要时间校验的业务，可以考虑第二种方法。

## 扩展

对于Ticker的扩展，往大说或许可以说很多，甚至发展为一个时间线框架。但我目前也只是将它用作平时项目里的一个趁手的工具。如果有兴趣，可以与我讨论，也可以随意增添一些个性化的功能。

完整的代码与使用用法，请移步[GitHub](https://github.com/yilishabuBai/Ticker)。

最后，谢谢阅读，欢迎大家转发。：）

