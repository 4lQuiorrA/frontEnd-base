### Proxy 的定义

> Proxy 对象用于定义基本操作的自定义行为（如属性查找、赋值、枚举、函数调用），用一句简短的话来说就说：当前对象的自带的功能，不足以满足当前业务要求，需要使用者拓展对象的能力.同时经常配合反射`Reflect`来使用

### Proxy 基本用法

```
new Proxy(target,handler)
```

- handler 包含一些**陷阱**的占位对象
- traps 提供属性访问的方法。用作捕获对象的一些操作
- target 被`Proxy`代理虚拟化的对象。他常被作为代理的存储后端。

**返回一个`proxy`的实例化对象，去操作对象方法内容**

### Proxy API

1. handler.getPrototypeOf() -> Object.getPrototypeOf()的陷阱

```
var arr = [1,2,3]

Object.getPrototypeOf(arr)

var proxy = new Proxy(arr,{
getPrototypeOf:function(val){
    console.log(val)
return {
a:1
}
}
})

Object.getPrototypeOf(proxy) // {a:1}
```

2. handler.setPrototypeOf() -> Object.setPrototypeOf 的陷阱

```
let testObject = {}

function testFun() {}

testObject = new Proxy(testObject, {
  setPrototypeOf() {
    return testFun.prototype
  },
})

Object.setPrototypeOf(testObject, testFun)
```

3. handler.isExtensible -> Object.isExtensible 的陷阱

```
let testObj = {}

let testProxy = new Proxy(testObj, {
  isExtensible: function () {
    return Reflect.isExtensible(testObj)
  },
})

Object.isExtensible(testProxy)

```

在这里，无法直接返回一个`primitive`值来作为`isExtensible`的展示结果。需要通过`Reflect`来反射出结果，当然之前的也可以通过`Reflect`来反射出结果来作为`Proxy`的返回值，这也是比较推荐的，作为获取内容的时候的返回值

4. handler.preventExtensions -> Object.preventExtensions 的陷阱

```
let testObj = {}

let testProxy = new Proxy(testObj, {
  preventExtensions: function () {
    console.log('控制扩展',Reflect.preventExtensions(testObj)) // 执行反射，将动作执行，禁止对象扩展
    console.log(Object.isExtensible(testObj))
    return Reflect.preventExtensions(testObj)
  },
})

console.log(Object.preventExtensions(testProxy))

```

5. handler.getOwnPropertyDescriptor -> Object.getOwnPropertyDescriptor 的陷阱

```
let testObj = {
  a: 1,
}

let testProxy = new Proxy(testObj, {
  getOwnPropertyDescriptor: function () {
    return Reflect.getOwnPropertyDescriptor(testObj, 'a')
  },
})

console.log(Object.getOwnPropertyDescriptor(testProxy, 'a'))

```

6. handler.defineProperty -> Object.defineProperty 的陷阱

```
let testObj = {
  a: 1,
}

let testProxy = new Proxy(testObj, {
  defineProperty: function () {
    return Reflect.defineProperty(testObj, 'b', {
      value: 1,
      enumerable: false,
      writable: false,
    })
  },
})

console.log(
  Object.defineProperty(testProxy, 'b', {
    enumerable: false,
  })
)

```

自动生成了`value:1`

7. handler.has -> in 操作符的陷阱

```
let testObj = {
  a: 1,
}

let testProxy = new Proxy(testObj, {
  has: function () {
    return false
  },
})

'a' in testProxy

```

**注意，如果我`return:1`，当我打印的时候，他会将他转成 `Boolean`**

8. handler.get -> 属性读取的陷阱

9. handler.set -> 属性设置的陷阱

```
var testObj = { a: 1 }
var testProxy = new Proxy(testObj, {
  set: function (target, key, value) {
    Reflect.set(target, key, value)
  },
})

testProxy.a = 2
testProxy.b = 2
console.log(testProxy, testObj)



```

10. handler.deleteProperty -> delete 属性使用的陷阱
11. handler.ownKeys -> Object.getOwnPropertyNames 和 Object.getOwnPropertySymbols 方法的陷阱
12. handler.apply -> 函数调用的陷阱

```
function foo() {
  console.log(1)
}

let testProxy = new Proxy(foo, {
  apply: function () {
    console.log(2)
  },
})

testProxy() //2
```

13. handler.construct -> new 函数调用的陷阱

### 为什么要使用 Proxy

可以使用`Proxy`来对 Object 的行为进行控制，同时不用失去实用性和简单性

### Proxy 的应用场景

1. 负索引
   访问如：`arr[-1]`

```
var a = [1, 2, 3]
let p = new Proxy(a, {
  get: function (oTarget, sKey) {
    if (sKey < 0) {
      return Reflect.get(oTarget, Number(oTarget.length) + Number(sKey))
    } else {
      return Reflect.get(oTarget, sKey)
    }
  },
})
console.log(p[-1])
console.log(p[-2])
console.log(p[-3])
```

2. 隐藏属性

```
var a = {
  _b: 1,
  c: 2,
}

// _b为私有属性，不能被外部所获取到，get ，无法获取property
var aStatic = new Proxy(a, {
  get: function (value, prop) {
    console.log(value, prop)
    if (prop.startsWith('_') || !(prop in value)) {
      return undefined
    } else {
      return value[prop]
    }
  },
  has: function (val, prop) {
    return !prop.startsWith('_') && prop in val
  },
  ownKeys: function (val) {
    console.log(val, Reflect.ownKeys(val))
    let ownKeyLists = Reflect.ownKeys(val).filter(
      (item) => !item.startsWith('_')
    )
    return ownKeyLists
  },
})

console.log(aStatic.c)
```

3. 缓存
   > 根据需要将对象包装为无效（和重新同步）的属性，每次要访问对象属性，都得重新获取，保护一些信息，如：银行卡号，余额，等等。

```
const cookieControl = (target, ttl = 60) => {
  const createTime = Date.now()
  const isExpired = () => Date.now() - createTime > ttl * 1000
  return new Proxy(target, {
    get: (obj, props) => (isExpired() ? undefined : Reflect.get(obj, props)),
  })
}

let bankAccount = cookieControl({ money: 1000 }, 10)

console.log(bankAccount.money)

setTimeout(() => {
  console.log(bankAccount.money)
}, 10 * 1000)

```

4. 重载运算符

> 重载 has,in,new 方法

5. 实现观察者模式
