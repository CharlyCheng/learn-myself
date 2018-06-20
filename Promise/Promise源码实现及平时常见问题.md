
Promise/A+源码实现及平时常见问题

## Promise
promise是解决回调的问题的，通过then的链式调用，让我们能更清晰的理解阅读代码，下面我们剖析Promise内部结构，一步一步实现一个完整的、能通过所有Test case的Promise类。

### Promise标准解读

#### 常用词汇解释：

1.1 “promise”是一个对象或函数，它的行为符合这个规范

1.2 “thenable”是定义then方法的对象或函数

1.3 “value”是任何合法的JavaScript值（包括undefined，thenable或promise）

1.4 “异常”是使用throw语句抛出的值

1.5 “原因”是一个值（结果）表明promise被拒绝的原因


#### 常用promise要求：
2.1 一个promise必须包含初始态, 成功（完成）态, 或者失败（拒绝）态这三个状态中的一种

2.2 当状态是初始态, promise可能转换到成功态或失败态。

2.3 当状态是成功态或者失败态，promise不能更改成别的状态，必须有个不能更改的值（结果）或者不能更改的失败（错误）原因

2.4 上面“不能改变”的意思是不可改变的状态（即 ===），但并不意味着深不可变。

2.5 一个promise必须有一个then方法来获取成功的值（结果）或失败（错误）的原因。一个promise方法接收两个参数：promise.then(onFulfilled, onRejected)

2.6 onFulfilled和onRejected都是可选参数：如果onFulfilled不是函数，则必须忽略它。如果onRejected不是函数，则必须忽略它。

2.7 如果onFulfilled是一个函数：必须在promise执行完成后调用，promise的返回值作为第一个参数。在promise执行前不得调用。只能调用一次。

2.8 如果onRejected是一个函数：必须在promise执行完成后调用，promise的错误原因作为第一个参数。在promise执行前不得调用。只能调用一次。

2.9 在执行上下文堆栈仅包含平台代码之前，不能调用onFulfilled或onRejected

3.0 onFulfilled和onRejected必须是函数（即 没有这个值）

3.1 then方法可能会在相同的promise被多次调用。如果/当promise成功时，所有各自的onFulfilled回调必须按照其始发调用的顺序执行。如果/当promise失败时，所有各自的onRejected回调必须按照其始发调用的顺序执行

3.2 then方法必须返回一个promise。例如：promise2 = promise1.then(onFulfilled, onRejected);如果onFulfilled或onRejected引发异常e，promise2必须以e作为拒绝原因
如果onFulfilled不是一个函数并且promise1是被成功，那么promise2必须用与promise1相同的值执行。如果onRejected不是函数并且promise1是被失败，那么promise2必须用与promise1相同的失败原因。

3.3 Promise的解决程序是一个抽象操作，取得输入的promise和一个值（结果），我们表示为 Resolve(promise, x)。
Resolve(promise, x)的意思是创建一个方法Resolve方法执行时传入两个参数promise和x（promise成功态时返回的值）。如果x是一个thenable（见上文术语1.2），它试图创造一个promise采用x的状态，假设x的行为至少貌似promise。否则，它用x值执行promise。

3.4 对thenable的这种处理允许promise实现交互操作，只要它们暴露Promise / A +兼容的方法即可。 它还允许Promises / A +实现通过合理的方法“吸收”不合格的实现

3.5 去运行 [[Resolve]](promise, x)，需执行以下步骤：如果promise和x引用同一个对象，则以TypeError为原因拒绝promise

3.6 如果x是一个promise，采用它的状态：
如果x是初始态，promise必须保持初始态(即递归执行这个解决程序)，直到x被成功或被失败。（即，直到resolve或者reject执行）
如果/当x被成功时，用相同的值（结果）履行promise。
如果/当x被失败时，用相同的错误原因履行promise。

3.7 否则，如果x是一个对象或函数,
让then等于x.then。
如果x.then导致抛出异常e，拒绝promise并用e作为失败原因。
如果then是一个函数，则使用x作为此参数调用它，第一个参数resolvePromise，第二个参数rejectPromise，其中：
如果使用值（结果）y调用resolvePromise，运行[[Resolve]]（promise，y）我的解决程序的名字是resolveExecutor。
如果使用拒绝原因r调用resolvePromise，运行reject(r)。
如果resolvePromise和rejectPromise都被调用，或者对同一个参数进行多次调用，则第一次调用优先，并且任何进一步的调用都会被忽略。
如果调用then方法抛出异常e，
如果resolvePromise或rejectPromise已经调用了，则忽略它。
否则，以e作为失败原因拒绝promise。
如果then不是一个对象或者函数，则用x作为值（结果）履行promise。

3.8 如果x不是一个对象或函数，则用x作为值履行promise。
如果一个primse是通过一个thenable参与一个循环的可链接表达式来解决的thenable链，那么[[Resolve]]（promise，thenable）的递归性质最终会导致[[Resolve]]（promise，thenable）被再次调用， 上述算法将导致无限递归。 支持这种实现，但不是必需的，来检测这种递归并以一个信息性的TypeError为理由拒绝promise。


### Promise 实现
#### 构造函数
因为标准并没有指定如何构造一个Promise对象，所以我们同样以目前一般Promise实现中通用的方法来构造一个Promise对象，也是ES6原生Promise里所使用的方式，即：

``` bash
// Promise构造函数接收一个executor函数，executor函数执行完同步或异步操作后，调用它的两个参数resolve和reject
var promise = new Promise(function(resolve, reject) {
  /*
    如果操作成功，调用resolve并传入value
    如果操作失败，调用reject并传入reason
  */
})
```

我们先实现构造函数的框架如下：
```
function Promise(executor) {
    var self = this;            //缓存this
    self.status = "pending"     //设置初始态
    self.value = undefined      //定义成功的值默认undefined
    self.reason = undefined     //定义失败的原因默认undefined
    self.onResolvedCallbacks = []//定义成功的回调数组
    self.onRejectedCallbacks = []//定义失败的回调数组
    executor (resolve, reject)
}
```

上面的代码基本实现了Promise构造函数的主体，但目前还有两个问题：

我们给executor函数传了两个参数：resolve和reject，这两个参数目前还没有定义

executor有可能会出错（throw），类似下面这样，而如果executor出错，Promise应该被其throw出的值reject：

new Promise(function(resolve, reject) {
  throw 2
})
所以我们需要在构造函数里定义resolve和reject这两个函数：
基本上就是在判断状态为pending之后把状态改为相应的值，并把对应的value和reason存在self的value与reason属性上面，之后执行相应的回调函数，逻辑很简单，这里就不多解释了。
```
function Promise(executor) {
    var self = this;            //缓存this
    self.status = "pending"     //设置初始态
    self.value = undefined      //定义成功的值默认undefined
    self.reason = undefined     //定义失败的原因默认undefined
    self.onResolvedCallbacks = []//定义成功的回调数组
    self.onRejectedCallbacks = []//定义失败的回调数组
    let resolve = (value) => {
        if (value instanceof MyPromise) return value.then(resolve, reject)
        setTimeout( () => {
            if (self.status === "pending") {
                self.status = "fulfilled"
                self.value = value
                self.onResolvedCallbacks.forEach ((onFulfilled) => {
                    onFulfilled(value)
                })
            }
        })
    }
    let reject = (reason) => {
        setTimeout ( () => {
            if (self.status === "pending") {
                self.status = "reject"
                self.value = value
                self.onRejectedCallbacks.forEach ((onRejected) => {
                    onRejected(reason)
                })
            }
        })
    }
    try {
        executor (resolve, reject)
    } catch (e) {
        reject (e)
    }
}
```

#### then方法

Promise对象有一个then方法，用来注册在这个Promise状态确定后的回调，很明显，then方法需要写在原型链上。then方法会返回一个Promise，关于这一点，Promise/A+标准并没有要求返回的这个Promise是一个新的对象，但在Promise/A标准中，明确规定了then要返回一个新的对象，目前的Promise实现中then几乎都是返回一个新的Promise(详情)对象，所以在我们的实现中，也让then返回一个新的Promise对象。

```bash
Promise.prototype.then = (onResolved = (value) => {value}, onRejected = (reason) => {throw reason}) => {
    let self = this;
    let promise2
    if (self.status === "pending") {
        return (promise2 = new Promise((resolve, reject) => {

        }))
    }
    if (self.status === "rejected") {
        return (promise2 = new Promise((resolve, reject) => {

        }))
    }
    if (self.status === "fulfilled") {
        return (promise2 = new Promise((resolve, reject) => {

        }))
    }
}
```

Promise总共有三种可能的状态，我们分三个if块来处理，在里面分别都返回一个new Promise。
根据标准，我们知道，对于如下代码，promise2的值取决于then里面函数的返回值:

```bash
promise2 = promise1.then(function(value) {
  return 4
}, function(reason) {
  throw new Error('sth went wrong')
})
```

需要在then里面执行onResolved或者onRejected，并根据返回值(标准中记为x)来确定promise2的结果，并且，如果onResolved/onRejected返回的是一个Promise，promise2将直接取这个Promise的结果：

```bash
Promise.prototype.then = (onResolved = (value) => {value}, onRejected = (reason) => {throw reason}) => {
    let self = this;
    let promise2
    if (self.status === "pending") {
        return (promise2 = new Promise((resolve, reject) => {
            self.onResolvedCallbacks.push(() => {
                    try {
                        let x = onfulfilled(self.value)
                        resolveExecutor(promise2, x , resolve, reject)
                    } catch (e) {
                        reject(e)
                    }
                });
                self.onRejectedCallbacks.push(() => {
                    try {
                        let x = onRejected(self.reason)
                        resolveExecutor(promise2, x , resolve, reject)
                    } catch (e) {
                        reject(e)
                    }
                });
        }))
    }
    if (self.status === "rejected") {
        return (promise2 = new Promise((resolve, reject) => {
            setTimeout(() => {
                try {
                    let x = onRejected(self.reason)
                    resolveExecutor(promise2, x , resolve, reject)
                } catch (e) {
                    reject(e)
                }
            });
        }))
    }
    if (self.status === "fulfilled") {
        return (promise2 = new Promise((resolve, reject) => {
            setTimeout(() => {
                try {
                    let x = onfulfilled(self.value)
                    resolveExecutor(promise2, x , resolve, reject)
                } catch (e) {
                    reject(e)
                }
            });
        }))
    }
    static catch(onRejected) {
        this.then(null, onRejected)
    }
    static resolve(value) {
       return new MyPromise(resolve => {
           resolve(value)
       })
    }
    static reject(reason) {
       return new MyPromise(resolve => {
           reject(value)
       })
    }
    static all(promises) {
       return new MyPromise((resolve, reject) => [
           let len = promises.length
           let resolveAry = []
           let count = 0
           for (let i = 0; i< len; i++) {
               promises[i].then(value => {
                   resolveAry[i] = value
                   if (++count === len) resolve(resolveAry)
               }, reject)
           }
       ])
    }
    static race (promises) {
       return new MyPromise((resolve, reject) => {
           for (let i = 0, l = promises.length; i < l;i++) {
               promises[i].then(resolve, reject)
           }
       })
    }

}
```

```bash
//promise主要解决程序，也是promise的难点
let resolveExecutor = (promise2, x, reslove, reject) => {
    let isThenCalled = false    // 定义个标识 promise2是否已经resolve 或 reject了
    if (promise2 === x) {       //如果promise和x引用同一个对象，则以TypeError为原因拒绝promise
        return reject(new TypeError("循环引用！！！"))
    }
    if (x instanceof MyPromise) {   //如果x是一个promise，采用它的状态
        if (x.status === "pending") { //如果x是初始态，promise必须保持初始态(即递归执行这个解决程序)，直到x被成功或被失败。（即，直到resolve或者reject执行）
            x.then(function(y){
                resolveExecutor(promise2, y , resolve, reject)
            }, reject)
        } else {
            //如果/当x被成功时，用相同的值（结果）履行promise。
            //如果/当x被失败时，用相同的错误原因履行promise
            x.then(resolve, reject)
        }
    } else if (x !== null && (typeof x === "object" || typeof x === "function")){ //否则，如果x是一个对象或函数,
        try {
            let then = x.then
            if (typeof then === "function") {
                let resolvePromise = y => {
                    if (isThencalled){return}
                    isThenCalled = true
                    resolveExecutor(promise2, y, resolve, reject)
                }
                let rejectPromise = r => {
                    if (isThencalled){return}
                    isThenCalled = true
                    reject(r)
                }
                then.call(x, resolvePromise, rejectPromise)
            } else {
                //到此的话x不是一个thenable对象，那直接把它当成值resolve promise2就可以了
                resolve(x)
            }
        } catch (e) {
            if (isThenCalled) return
            isThenCalled = true
            reject(e)
        }
    }else {
        resolve(x)
    }
}
```

至此，我们基本实现了Promise标准中所涉及到的内容，但还有几个问题

#### 关于Promise的其它问题

Promise的性能问题
如何停止一个Promise链？
Promise链上返回的最后一个Promise出错了怎么办？
Angular里的$q跟其它Promise的交互
出错时，是用throw new Error()还是用return Promise.reject(new Error())呢？

#### 最佳实践
1.一是不要把Promise写成嵌套结构
2.二是链式Promise要返回一个Promise，而不只是构造一个Promise


#### 总结
[一步一步实现promise](https://github.com/xieranmaya/blog/issues/3)
[promise原文解读](https://juejin.im/post/5a982003518825556b6c30b8)
