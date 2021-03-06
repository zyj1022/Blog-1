# js功能函数的模拟实现

这里的模拟实现，目的是在于提升对重要函数的理解

## call、apply

call: 在调用的时候，修改调用对象的this指向，可以附带多个参数，可以有返回值

模拟思路：主要是修改this的指向，让其指向call的第一个参数

实现方式：让调用的函数对象作为传入的第一个参数的属性，执行完成之后删除它

```javascript
// call
Function.prototype.call2 = Function.prototype.call2 || function (context) {
    var context = context || window;
    context.fn = this;

    var args = [];
    for(var i = 1, len = arguments.length; i < len; i++) {
        args.push('arguments[' + i + ']');
    }

    var result = eval('context.fn(' + args +')');

    delete context.fn
    return result;
}

// apply
Function.prototype.apply = Function.prototype.apply || function (context, arr) {
    var context = Object(context) || window;
    context.fn = this;

    var result;
    if (!arr) {
        result = context.fn();
    }
    else {
        var args = [];
        for (var i = 0, len = arr.length; i < len; i++) {
            args.push('arr[' + i + ']');
        }
        result = eval('context.fn(' + args + ')')
    }

    delete context.fn
    return result;
}
```

## bind

需要解决的几个问题：

1. bind返回的是一个函数
2. 当bind返回的函数作为构造器调用时，其this指向指向实例对象，不指向bind方法的第一个参数对象
3. 调用bind，可以传入参数作为预置参数

```javascript
// bind
Function.prototype.bind2 = Function.prototype.bind2 || function (context) {

    if (typeof this !== "function") {
      throw new Error("Function.prototype.bind - what is trying to be bound is not callable");
    }

    var self = this; // 保存最原始的函数对象
    var args = Array.prototype.slice.call(arguments, 1);//预置参数

    var fNOP = function () {};

    var fBound = function () {
        var bindArgs = Array.prototype.slice.call(arguments);
        return self.apply(this instanceof fNOP ? this : context, args.concat(bindArgs));
    }

    fNOP.prototype = this.prototype;
    fBound.prototype = new fNOP(); // 继承 fNOP
    return fBound;
}
```
