## AutoreleasePool

转载自[sunny](http://blog.sunnyxx.com/)的[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

### 对象释放时机

在 ARC 中，`autorelease`对象的释放时机并不一定是在当前作用域大括号结束时释放，如果当前对象包裹创建在一个手动添加的`autoreleasepool`中，那么是在这个`autoreleasepool`的大括号作用域结束时进行释放，如果没有手动添加`autoreleasepool`，则在当前`runloop`迭代结束时释放。原因为**系统在每个`runloop`迭代中都加入了`autoreleasepool`的`push`和`pop`操作**

### 原理

在 ARC 中，我们使用`@autoreleasepool{}`来使用一个`AutoreleasePool`，随后编译器将其改写成下面的样子： 

```c
void *context = objc_autoreleasePoolPush();
// {}中的代码
objc_autoreleasePoolPop(context);
```

而这两个函数都是对`AutoreleasePoolPage`的简单封装，所以自动释放机制的核心就在于这个类。 

![AutoreleasePool_0](./AutoreleasePool_0.jpg)

- `AutoreleasePool`并没有单独的数据结构，而是由若干个`AutoReleasePoolPage`以双向链表的形式组合而成
- `AutoreleasePool`是按线程一一对应的
- `AutoreleasePoolPage`每个对象会开辟4096字节(虚拟内存一页的大小)的内存，除了图中`page`基本信息的外的空间用于存储`autorelease`对象的地址
- 上面的`id *next`指针作为游标指向栈顶最新add进来的autorelease对象的下一个位置
- 一个`AutoreleasePoolPage`的空间被占满时，会新建一个`AutoreleasePoolPage`对象，连接链表，后来的`autorelease`对象在新的page加入

### 释放时机

每当进行一次`objc_autoreleasePoolPush`调用时，`runtime`向当前的`AutoreleasePoolPage`中添加一个`哨兵对象`，值为0（也就是个`nil`），那么这一个`page`就变成了下面的样子： 

![AutoreleasePool_1](./AutoreleasePool_1.jpg)

`objc_autoreleasePoolPush`的返回值正是这个哨兵对象的地址，被`objc_autoreleasePoolPop(哨兵对象)`作为入参，于是：

1. 根据传入的哨兵对象地址找到哨兵对象所处的page
2. 将晚于哨兵对象插入的所有`autorelease`对象都发送一次`- release`消息，并向回移动`next`指针到正确位置(可跨越若干个`page`)

知道了上面的原理，嵌套的`AutoreleasePool`就非常简单了，pop的时候总会释放到上次`push`的位置为止，多层的`pool`就是多个哨兵对象而已，就像剥洋葱一样，每次一层，互不影响。

### 遍历

使用容器的block版本的枚举器时，内部会自动添加一个AutoreleasePool： 

```objective-c
[array enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
    // 这里被一个局部@autoreleasepool包围着
}];
```

当然，在普通for循环和for in循环中没有，所以，还是新版的block版本枚举器更加方便。for循环中遍历产生大量autorelease变量时，就需要手加局部AutoreleasePool咯。