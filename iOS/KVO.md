## 一、KVO 是什么？

KVO 是 Objective-C 对**观察者设计模式**的一种实现。
KVO 提供一种机制，指定一个被观察对象(例如 A 类)，当对象某个属性(例如 A 中的字符串 name)发生更改时，对象会获得通知，并作出相应处理；【且不需要给被观察的对象添加任何额外代码，就能使用 KVO 机制】

在 MVC 设计架构下的项目，KVO 机制很适合实现 mode 模型和 view 视图之间的通讯。
例如：代码中，在模型类A创建属性数据，在控制器中创建观察者，一旦属性数据发生改变就收到观察者收到通知，通过 KVO 再在控制器使用回调方法处理实现视图 B 的更新；

## 二、实现原理

KVO 的实现依赖于 Objective-C 强大的 Runtime。

### 基本实现原理

当观察某对象 A 时，KVO 机制动态创建一个对象A当前类的子类，并为这个新的子类重写了被观察属性 keyPath 的 setter 方法。setter 方法随后负责通知观察对象属性的改变状况。

##### **深入剖析**：

Apple 使用了 isa 混写（isa-swizzling）来实现 KVO 。当观察对象A时，KVO机制动态创建一个新的名为：**NSKVONotifying_A** 的新类，该类继承自对象A的本类，且 KVO 为 NSKVONotifying_A 重写观察属性的 setter 方法，setter 方法会负责在调用原 setter 方法之前和之后，通知所有观察对象属性值的更改情况。
（备注： isa 混写（isa-swizzling）isa：is a kind of ； swizzling：混合，搅合；）

**①NSKVONotifying_A 类剖析：**在这个过程，被观察对象的 isa 指针从指向原来的 A 类，被 KVO 机制修改为指向**系统新创建的子类 NSKVONotifying_A 类，来实现当前类属性值改变的监听**；
所以当我们从应用层面上看来，完全没有意识到有新的类出现，这是系统“隐瞒”了对 KVO 的底层实现过程，让我们误以为还是原来的类。但是此时如果我们创建一个新的名为“NSKVONotifying_A”的类，就会发现系统运行到注册 KVO 的那段代码时程序就崩溃，因为系统在注册监听的时候**动态**创建了名为 NSKVONotifying_A 的中间类，并指向这个中间类了。
（**isa** 指针的作用：每个对象都有 isa 指针，指向该对象的类，它告诉 Runtime 系统这个对象的类是什么。所以对象注册为观察者时，isa 指针指向新子类，那么**这个被观察的对象就神奇地变成新子类的对象（或实例）了。**） 因而在该对象上对 setter 的调用就会调用已重写的 setter，从而激活键值通知机制。
—>我猜，这也是 KVO 回调机制，为什么都俗称KVO技术为黑魔法的原因之一吧：内部神秘、外观简洁。
**②子类setter方法剖析：**KVO 的键值观察通知依赖于 NSObject 的两个方法:`willChangeValueForKey:`和 `didChangevlueForKey:`，在存取数值的前后分别调用 2 个方法：
被观察属性发生**改变之前**，`willChangeValueForKey:`被调用，通知系统该 keyPath 的属性值即将变更；当**改变发生后**， `didChangeValueForKey: `被调用，通知系统该 keyPath 的属性值已经变更；**之后**， `observeValueForKey:ofObject:change:context:` 也会被调用。且重写观察属性的 setter 方法这种继承方式的注入是在**运行时**而不是**编译时**实现的。
KVO 为子类的观察者属性重写调用存取方法的工作原理在代码中相当于：

```objective-c
-(void)setName:(NSString *)newName{ 
[self willChangeValueForKey:@"name"];    //KVO 在调用存取方法之前总调用 
[super setValue:newName forKey:@"name"]; //调用父类的存取方法 
[self didChangeValueForKey:@"name"];     //KVO 在调用存取方法之后总调用
}
```

### 手动实现

`+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key `

方法返回false，并且手动调用：

`- (void)willChangeValueForKey:(NSString *)key; `

`- (void)didChangeValueForKey:(NSString *)key;`

## **三、特点：**

观察者观察的是属性，只有遵循 KVO 变更属性值的方式才会执行 KVO 的回调方法，例如是否执行了 setter 方法、或者是否使用了 KVC 赋值。
如果赋值没有通过 setter 方法或者 KVC，而是直接修改属性对应的成员变量，例如：仅调用 _name = @"newName"，这时是不会触发 KVO 机制，更加不会调用回调方法的。
所以使用 KVO 机制的前提是遵循 KVO 的属性设置方式来变更属性值。