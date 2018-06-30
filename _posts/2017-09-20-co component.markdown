---
title:  “tj/co模块分析”
date:   2017-09-20 08:00:12 +0800
categories: javascript
---

JavaScript中存在一个“回调金字塔“地狱，因为多数函数返回是异步的，代码会非常难看，而且不方便阅读。

{% highlight javascript %}
function getData(param, callback) {
  // your work
  if (err) {
    callback(err);
    return;
  }
  callback(null, data);
};
// function getMoreData(param, callback) {}
// function getMeta(param, callback) {}
// function getMoreMeta(param, callback) {}

getData(param1, function(err, data){
  getMoreData(param2, function(err, moreData){
    getMeta(param3, function(err, meta){
      getMoreMeta(param4, function(err, moreMeta){
        // ...
      });
    });
  });
});
{% endhighlight %}


解决的方法有多种:
* Async.js库，使用Async中的辅助方法series, parallel, waterfall等等
* Promises，类似于Java中的Future，使用then，catch的链式语法
* Async/Await， ES7功能，暂不支持Node，但是可以用Babel转换

Async/Await感觉上像是一个终极解决方案，可以编写看起来像是同步，但仍然是异步的代码。可惜的是目前
主流不管是前端还是后端都还不支持ES7。

{% highlight javascript %}
async function getUser(id) {  
    if (id) {
        return await db.user.byId(id);
    } else {
        throw 'Invalid ID!';
    }
}

try {  
    let user = await getUser(123);
} catch(err) {
    console.error(err);
}
{% endhighlight %}

之前项目中使用的是tj/co模块，可以很方面的写出“同步”代码，使代码看起来非常清爽。据说tj是一个大神，
两百多行代码就把这个问题解决了，所以决定研究一下[tj/co](https://github.com/tj/co)的代码。
第一反应就是co应该是对generator做了一层封装，因为generator也是可以把异步回调变成“同步”代码。
generator由function*定义，可以用yield多次返回。

{% highlight javascript %}
/**
 * slice() reference.
 */

var slice = Array.prototype.slice;

/**
 * Expose `co`.
 */

module.exports = co['default'] = co.co = co;

/**
 * Wrap the given generator `fn` into a
 * function that returns a promise.
 * This is a separate function so that
 * every `co()` call doesn't create a new,
 * unnecessary closure.
 *
 * @param {GeneratorFunction} fn
 * @return {Function}
 * @api public
 */

co.wrap = function (fn) {
  createPromise.__generatorFunction__ = fn;
  return createPromise;
  function createPromise() {
    return co.call(this, fn.apply(this, arguments));
  }
};

/**
 * Execute the generator function or a generator
 * and return a promise.
 *
 * @param {Function} fn
 * @return {Promise}
 * @api public
 */

/* co使用例子
 co(function* () {
   var result = yield Promise.resolve(true);  // gen.next().value是一个Promise
   return result;   // gen.next().value返回result
 }).then(function (value) {
   console.log(value);
 }, function (err) {
   console.error(err.stack);
 });
 */

function co(gen) {
  var ctx = this;
  // 上述例子中arguments显示为{'0'：[Function]}，传入参数gen仍然是一个Function类型
  // 将函数参数转换成数组，并去掉第一个，args此时为[]
  var args = slice.call(arguments, 1);

  // we wrap everything in a promise to avoid promise chaining,
  // which leads to memory leak errors.
  // see https://github.com/tj/co/issues/180
  // 返回一个Promise
  return new Promise(function(resolve, reject) {
    // 如果gen类型是一个function，gen.apply(ctx, args)获取generator对象
    if (typeof gen === 'function') gen = gen.apply(ctx, args);
    // 对于一个generator对象, 判断gen.next是一个方法；否则返回gen
    if (!gen || typeof gen.next !== 'function') return resolve(gen);

    onFulfilled();

    /**
     * @param {Mixed} res
     * @return {Promise}
     * @api private
     */

    function onFulfilled(res) {
      var ret;
      try {
        // 第一次调用gen.next的时候，res等于undefined，ret是一个对象，
        //  ret: { value: Promise { true }, done: false }
        // 第二次调用gen.next的时候，res等于true，ret是一个对象，
        // ret: { value: true, done: true }
        ret = gen.next(res);
      } catch (e) {
        return reject(e);
      }
      // 继续处理generator下一个值
      next(ret);
      return null;
    }

    /**
     * @param {Error} err
     * @return {Promise}
     * @api private
     */

    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    /**
     * Get the next value in the generator,
     * return a promise.
     *
     * @param {Object} ret
     * @return {Promise}
     * @api private
     */

    function next(ret) {
      // 如果generator已经完成最后一次调用，调用resolve方法
      if (ret.done) return resolve(ret.value);
      // 将ret.value包装成一个Promise
      var value = toPromise.call(ctx, ret.value);
      // value现在已经是一个Promise，调用then把onFulfilled()函数和onRejected()函数
      // 添加到Promise对象的回调链中， 当resolve()或者reject()方法执行的时候，回调链中
      // 的回调函数会根据PromiseStatus的状态情况而被依次调用。

      // resolve(value)和reject(e)分别将返回值赋给onFulfilled(value)和
      // onRejected(reason)实参。此例中调用onFulfilled(true)。
      // 如果resolve返回值是一个Promise的对象，那么剩下的由then()构造的回调链会转交给新的
      // Promise对象并完成调用。
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
      // 到这一步就肯定有问题了，reject吧
      return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
        + 'but the following object was passed: "' + String(ret.value) + '"'));
    }
  });
}

/**
 * Convert a `yield`ed value into a promise.
 *
 * @param {Mixed} obj
 * @return {Promise}
 * @api private
 */

function toPromise(obj) {
  // 如果obj为空，直接返回
  if (!obj) return obj;
  // 如果obj是一个Promise，直接返回
  if (isPromise(obj)) return obj;
  // 如果obj是一个generator或者generator function，调用co.call
  if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);
  // 如果obj类型是函数，调用thunkToPromise.call
  if ('function' == typeof obj) return thunkToPromise.call(this, obj);
  // 如果obj类型是数组，调用arrayToPromise.call
  if (Array.isArray(obj)) return arrayToPromise.call(this, obj);
  // 如果obj类型是对象，调用objectToPromise.call
  if (isObject(obj)) return objectToPromise.call(this, obj);
  return obj;
}

/**
 * Convert a thunk to a promise.
 *
 * @param {Function}
 * @return {Promise}
 * @api private
 */

function thunkToPromise(fn) {
  var ctx = this;
  return new Promise(function (resolve, reject) {
    fn.call(ctx, function (err, res) {
      if (err) return reject(err);
      if (arguments.length > 2) res = slice.call(arguments, 1);
      resolve(res);
    });
  });
}

/**
 * Convert an array of "yieldables" to a promise.
 * Uses `Promise.all()` internally.
 *
 * @param {Array} obj
 * @return {Promise}
 * @api private
 */

function arrayToPromise(obj) {
  return Promise.all(obj.map(toPromise, this));
}

/**
 * Convert an object of "yieldables" to a promise.
 * Uses `Promise.all()` internally.
 *
 * @param {Object} obj
 * @return {Promise}
 * @api private
 */

function objectToPromise(obj){
  var results = new obj.constructor();
  var keys = Object.keys(obj);
  var promises = [];
  for (var i = 0; i < keys.length; i++) {
    var key = keys[i];
    var promise = toPromise.call(this, obj[key]);
    if (promise && isPromise(promise)) defer(promise, key);
    else results[key] = obj[key];
  }
  return Promise.all(promises).then(function () {
    return results;
  });

  function defer(promise, key) {
    // predefine the key in the result
    results[key] = undefined;
    promises.push(promise.then(function (res) {
      results[key] = res;
    }));
  }
}

/**
 * Check if `obj` is a promise.
 *
 * @param {Object} obj
 * @return {Boolean}
 * @api private
 */

function isPromise(obj) {
  return 'function' == typeof obj.then;
}

/**
 * Check if `obj` is a generator.
 *
 * @param {Mixed} obj
 * @return {Boolean}
 * @api private
 */

function isGenerator(obj) {
  return 'function' == typeof obj.next && 'function' == typeof obj.throw;
}

/**
 * Check if `obj` is a generator function.
 *
 * @param {Mixed} obj
 * @return {Boolean}
 * @api private
 */

function isGeneratorFunction(obj) {
  var constructor = obj.constructor;
  if (!constructor) return false;
  if ('GeneratorFunction' === constructor.name || 'GeneratorFunction' === constructor.displayName) return true;
  return isGenerator(constructor.prototype);
}

/**
 * Check for plain object.
 *
 * @param {Mixed} val
 * @return {Boolean}
 * @api private
 */

function isObject(val) {
  return Object == val.constructor;
}
{% endhighlight %}
