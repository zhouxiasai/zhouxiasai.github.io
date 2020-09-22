# [Runtime](./Runtime.pdf)

### 概念

**Objective-C** 是基于 C 的，它为 C 添加了面向对象的特性。它将很多静态语言在编译和链接时期做的事放到了 runtime 运行时来处理，可以说 **runtime** 是我们 Objective-C 幕后工作者。

- **runtime**（简称运行时），是一套由C、C++、汇编编写的API。而 OC 就是 **运行时机制**，也就是在运行时候的一些机制，其中最主要的是 **消息机制**。
- 对于 C 语言，**函数的调用在编译的时候会决定调用哪个函数**。
- OC的函数调用成为消息发送，属于 **动态调用过程**。在编译的时候并不能决定真正调用哪个函数，只有在真正运行的时候才会根据函数的名称找到对应的函数来调用。
- 事实证明：在编译阶段，OC 可以 **调用任何函数**，即使这个函数并未实现，只要声明过就不会报错，只有当运行的时候才会报错，这是因为OC是运行时动态调用的。而 C 语言 **调用未实现的函数** 就会报错。

### 对象模型

![Runtime_0](./Runtime_0.png)

实例对象：`objc_object`结构体
类对象：`objc_class`结构体（单例，用于创建实例对象，储存对象方法、实例变量）
元类：`objc_class`结构体（单例，用于查找类方法，使其机制与对象查找方法一致）
分类：`category_t`结构体（运行时添加到类的结构体中）
协议：`protocol_t`结构体
扩展：不是结构体，在编译时直接加到类中

```c
typedef struct objc_class *Class;
typedef struct objc_object *id;

typedef struct method_t *Method;
typedef struct ivar_t *Ivar;
typedef struct category_t *Category; 
typedef struct property_t *objc_property_t;

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
}

struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};
```

#### 实例对象的结构

```c
struct objc_object {
private:
	isa_t isa;  // 用于查找对应的类
public:
    // 由于开启了指针优化后，isa不再是指针，要获取类指针就要用到 ISA() 方法。
    Class ISA(); // 非 tagged pointer 对象(比如类对象、元类对象)
    Class getIsa(); // 可以是 tagged pointer 对象(比如实例对象)
    void initIsa(Class cls /*nonpointer=false*/);
//省略剩余内容...
private:
    void initIsa(Class newCls, bool nonpointer, bool hasCxxDtor);
//省略剩余内容...
};

union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

	//二选一：指针优化时赋值bits，否则赋值 cls
    Class cls;
    uintptr_t bits;
    
#if SUPPORT_PACKED_ISA //是否支持在 isa 指针中插入除 Class 之外的信息

//-------------------arm64上的实现
# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL //获取 isa 指针的掩码
#   define ISA_MAGIC_MASK  0x000003f000000001ULL //获取 magic 值的掩码
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL //默认 magic 值
    struct {
        uintptr_t nonpointer        : 1;    //0：普通isa指针，1：优化的指针
        uintptr_t has_assoc         : 1;    //表示该对象是否包含associated object，如果没有，析构时会更快
        uintptr_t has_cxx_dtor      : 1;    //表示对象是否含有c++或者ARC的析构函数，如果没有，析构更快
        uintptr_t shiftcls          : 33;   // MACH_VM_MAX_ADDRESS 0x1000000000 类的指针(isa 真正的指向)
        uintptr_t magic             : 6;    //用于在调试时分辨对象是否未完成初始化（用于调试器判断当前对象是真的对象还是没有初始化的空间）
        uintptr_t weakly_referenced : 1;    //对象是否有过weak对象
        uintptr_t deallocating      : 1;    //是否正在析构
        uintptr_t has_sidetable_rc  : 1;    //该对象的引用计数值是否过大无法存储在isa指针
        uintptr_t extra_rc          : 19;   //存储引用计数值减一后的结果。对象的引用计数超过 1，会存在这个这个里面，如果引用计数为 10，extra_rc的值就为 9。
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

//省略剩余内容...
}
```

`SUPPORT_PACKED_ISA`：表示平台是否支持在`isa`指针中插入除`Class`之外的信息。如果支持就会将`Class`信息放入`isa_t`定义的`struct`内（`shiftcls`位域），并附上一些其他信息，比如引用计数`extra_rc`，析构状态；如果不支持，那么不会使用`isa_t`内定义的`struct`，这时`isa_t`只使用`cls`成员变量(`Class`指针，经常说的“`isa`指针”就是这个)。在 iOS 以及 MacOSX 上，`SUPPORT_PACKED_ISA` 定义为 1（支持）。

#### 类的结构

```c
struct objc_class : objc_object {
    // Class ISA;
    Class superclass; 		   // 父类
    cache_t cache;             // 方法缓存，用于优化方法调用(缓存已调用过的方法)
    class_data_bits_t bits;    // 用于(通过掩码)获取类的信息.

    class_rw_t *data() { 
        return bits.data();
    }
//省略剩余内容...
}

struct class_data_bits_t { // = class_rw_t 指针 + rr/alloc 标志位
    uintptr_t bits;

    class_rw_t* data() {
        return (class_rw_t *)(bits & FAST_DATA_MASK); //与掩码按位与获取数据结构体的指针
    }
//省略剩余内容...
};

//掩码值
// 当前类是swift类
#define FAST_IS_SWIFT           (1UL<<0)
//当前类或者父类含有默认的 retain/release 方法
#define FAST_HAS_DEFAULT_RR     (1UL<<1)
// 当前类的实例需要 raw isa
#define FAST_REQUIRES_RAW_ISA   (1UL<<2)
// 数据指针
#define FAST_DATA_MASK          0x00007ffffffffff8UL

struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods; //方法
    property_array_t properties;//属性
    protocol_array_t protocols;//协议

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;//是计算机语言用于解决实体名称唯一性的一种方法，做法是向名称中添加一些类型信息，用于从编译器中向连接器传递更多语义信息。
    
//省略剩余内容...
};

struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;//类名
    method_list_t * baseMethodList;//方法列表
    protocol_list_t * baseProtocols;//协议列表
    const ivar_list_t * ivars;//ivar列表

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;//属性列表

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};
```
在编译后`class_data_bits_t`指向的是一个`class_ro_t`的地址，这个结构体是不可变的(只读)。在运行时，才会通过`realizeClass`函数将`bits`指向`class_rw_t`。

在程序开始运行后会初始化`Class`，在这个过程中，会把编译器存储在`bits`中的`class_ro_t`取出，然后创建`class_rw_t`，并把`ro`赋值给`rw`，成为`rw`的一个成员变量，最后把`rw`设置给`bits`，替代之前`bits`中存储的`ro`。除了这些操作外，还会有一些其他赋值的操作，下面是初始化`Class`的精简版代码。

```c
ro = (const class_ro_t *)cls->data();//强制转换
if (ro->flags & RO_FUTURE) {
    // This was a future class. rw data is already allocated.
    rw = cls->data();
    ro = cls->data()->ro;
    cls->changeInfo(RW_REALIZED|RW_REALIZING, RW_FUTURE);
} else {
    // Normal class. Allocate writeable class data.
    rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);//初始化一个 class_rw_t 结构体,分配可读写数据空间
    rw->ro = ro;//设置结构体 ro 的值以及 flag
    rw->flags = RW_REALIZED|RW_REALIZING;
    cls->setData(rw);//最后设置正确的 data
}
```

初始化`rw`和`ro`之后，`rw`的`method list`、`protocol list`、`property list`都是空的，需要在下面`methodizeClass`函数中进行赋值。函数中会把`ro`的`list`都取出来，然后赋值给`rw`，如果在运行时动态修改，也是对`rw`做的操作。所以`ro`中存储的是编译时就已经决定的原数据，`rw`才是运行时动态修改的数据。

```c
static void methodizeClass(Class cls)
{
    bool isMeta = cls->isMetaClass();
    auto rw = cls->data();
    auto ro = rw->ro;

    method_list_t *list = ro->baseMethods();
    if (list) {
        prepareMethodLists(cls, &list, 1, YES, isBundleClass(cls));
        rw->methods.attachLists(&list, 1);
    }

    property_list_t *proplist = ro->baseProperties;
    if (proplist) {
        rw->properties.attachLists(&proplist, 1);
    }

    protocol_list_t *protolist = ro->baseProtocols;
    if (protolist) {
        rw->protocols.attachLists(&protolist, 1);
    }

    if (cls->isRootMetaclass()) {
        // root metaclass
        addMethod(cls, SEL_initialize, (IMP)&objc_noop_imp, "", NO);
    }

    // Attach categories.
    category_list *cats = unattachedCategoriesForClass(cls, true /*realizing*/);
    attachCategories(cls, cats, false /*don't flush caches*/);
}
```

总结：

- 编译期，类的相关方法、属性、实例变量、协议会被添加到class_ro_t这个只读结构体中。
- 运行期，类第一次被调用时，class_rw_t会被初始化。

#### Category 结构

```c
struct category_t {
    // 所属的类名，而不是Category的名字
    const char *name;
    // 所属的类，这个类型是编译期的类，这时候类还没有被重映射
    classref_t cls;
    // 实例方法列表
    struct method_list_t *instanceMethods;
    // 类方法列表
    struct method_list_t *classMethods;
    // 协议列表
    struct protocol_list_t *protocols;
    // 实例属性列表
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    // 类属性列表
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```

在`_read_images`函数中会执行一个循环嵌套，外部循环遍历所有类，并取出当前类对应`Category`数组。内部循环会遍历取出的`Category`数组，将每个`category_t`对象取出，最终执行`addUnattachedCategoryForClass`函数添加到`Category`哈希表中。

```c
// 将category_t添加到list中，并通过NXMapInsert函数，更新所属类的Category列表
static void addUnattachedCategoryForClass(category_t *cat, Class cls, 
                                          header_info *catHeader)
{
    // 获取到未添加的Category哈希表
    NXMapTable *cats = unattachedCategories();
    category_list *list;

    // 获取到buckets中的value，并向value对应的数组中添加category_t
    list = (category_list *)NXMapGet(cats, cls);
    if (!list) {
        list = (category_list *)
            calloc(sizeof(*list) + sizeof(list->list[0]), 1);
    } else {
        list = (category_list *)
            realloc(list, sizeof(*list) + sizeof(list->list[0]) * (list->count + 1));
    }
    // 替换之前的list字段
    list->list[list->count++] = (locstamped_category_t){cat, catHeader};
    NXMapInsert(cats, cls, list);
}
```

`Category`维护了一个名为`category_map`的哈希表，哈希表存储所有`category_t`对象。

```c
// 获取未添加到Class中的category哈希表
static NXMapTable *unattachedCategories(void)
{
    // 未添加到Class中的category哈希表
    static NXMapTable *category_map = nil;

    if (category_map) return category_map;

    // fixme initial map size
    category_map = NXCreateMapTable(NXPtrValueMapPrototype, 16);

    return category_map;
}
```

上面只是完成了向`Category`哈希表中添加的操作，这时候哈希表中存储了所有`category_t`对象。然后需要调用`remethodizeClass`函数，向对应的`Class`中添加`Category`的信息。

在`remethodizeClass`函数中会查找传入的`Class`参数对应的`Category`数组，然后将数组传给`attachCategories`函数，执行具体的添加操作。

```c
// 将Category的信息添加到Class，包含method、property、protocol
static void remethodizeClass(Class cls)
{
    category_list *cats;
    bool isMeta;
    isMeta = cls->isMetaClass();

    // 从Category哈希表中查找category_t对象，并将已找到的对象从哈希表中删除
    if ((cats = unattachedCategoriesForClass(cls, false/*not realizing*/))) {
        attachCategories(cls, cats, true /*flush caches*/);        
        free(cats);
    }
}
```

在`attachCategories`函数中，查找到`Category`的方法列表、属性列表、协议列表，然后通过对应的`attachLists`函数，添加到`Class`对应的`class_rw_t`结构体中。

```c
// 获取到Category的Protocol list、Property list、Method list，然后通过attachLists函数添加到所属的类中
static void attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    // 按照Category个数，分配对应的内存空间
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    
    // 循环查找出Protocol list、Property list、Method list
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    auto rw = cls->data();

    // 执行添加操作
    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
```

这个过程就是将`Category`中的信息，添加到对应的`Class`中，一个类的`Category`可能不只有一个，在这个过程中会将所有`Category`的信息都合并到`Class`中。

##### 方法覆盖

在有多个`Category`和原类的方法重复定义的时候，原类和所有`Category`的方法都会存在，并不会被后面的覆盖。假设有一个方法叫做`method`，`Category`和原类的方法都会被添加到方法列表中，只是存在的顺序不同。

在进行方法调用的时候，会优先遍历`Category`的方法，并且后面被添加到项目里的`Category`，会被优先调用。如果从方法列表中找到方法后，就不会继续向后查找，这就是类方法被`Category`”覆盖”的原因。

##### 关联对象

`objc_getAssociatedObject`函数的具体实现如下（`objc_setAssociatedObject`类似）：

```c
id _object_get_associative_reference(id object, void *key) {
    id value = nil;
    uintptr_t policy = OBJC_ASSOCIATION_ASSIGN;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            ObjectAssociationMap *refs = i->second;
            ObjectAssociationMap::iterator j = refs->find(key);
            if (j != refs->end()) {
                ObjcAssociation &entry = j->second;
                value = entry.value();
                policy = entry.policy();
                if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) {
                    objc_retain(value);
                }
            }
        }
    }
    if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
        objc_autorelease(value);
    }
    return value;
}
```

从源码可以看出，所有通过`associated`添加的属性，都被存在一个单独的哈希表`AssociationsHashMap`中。`objc_setAssociatedObject`和`objc_getAssociatedObject`函数本质上都是在操作这个哈希表，通过对哈希表进行映射来存取对象。

在`associated`的`API`中会设置一些内存管理的关键字，例如`OBJC_ASSOCIATION_ASSIGN`，这是用来指定对象的内存管理的，这些关键字在`Runtime`源码中也有对应的处理。

#### Tagged Pointer

64 位处理器系统中，指针长度为64位（8字节），寻址范围远超常规内存容量，为了优化存储存储空间，苹果推出了`Tagged Pointer`特性，将指针的部分位域用于直接储存内容信息，减少了内存占用、加快了代码执行效率（减少`malloc`和`free`过程）。如 `NSNumber`、对象结构体中的`isa_t`。

### 对象加载过程

#### 对象初始化

NSObject 对象初始化时，一般使用`alloc`+`init`或者`new`方法进行实例化（`new`方法本质上和`alloc`+`init`一致）

```objective-c
+ (id)alloc {
    return _objc_rootAlloc(self);
}
id _objc_rootAlloc(Class cls)  //相当于 [cls allocWithZone:nil]
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}

+ (id)new {
    return [callAlloc(self, false/*checkNil*/) init];
}

- (id)init {
    return _objc_rootInit(self);
}
 id _objc_rootInit(id obj)
{
    // In practice, it will be hard to rely on this function.
    // Many classes do not properly chain -init calls.
    return obj;
}

//在 OC 2.0 之后 alloc 和 allocWithZone: 方法最后调用的方法一致(会忽略zone参数)
id 
class_createInstance(Class cls, size_t extraBytes)
{
    return _class_createInstanceFromZone(cls, extraBytes, nil);
}

static __attribute__((always_inline)) 
id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone, 
                              bool cxxConstruct = true, 
                              size_t *outAllocatedSize = nil)
{
//省略上部分...
    size_t size = cls->instanceSize(extraBytes); //调用下面函数
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj = (id)calloc(1, size);
    obj->initIsa(cls); //初始化 isa 结构体
//省略部分...
    return obj;
}

    size_t instanceSize(size_t extraBytes) {
        //alignedInstanceSize()函数获取 class_ro_t 中的 instanceSize 变量值(未对齐)，经过对齐后的值，在编译器就已经决定的，不能在运行时进行动态改变。
        size_t size = alignedInstanceSize() + extraBytes;
        // CF框架需要所有对象大小最少是16字节
        // CF requires all objects be at least 16 bytes.
        if (size < 16) size = 16;
        return size;
    }
```

#### 对象销毁

在对象销毁时，运行时环境会调用`NSObject`的`dealloc`方法执行销毁代码，并不需要我们手动去调用。接着会调用到`Runtime`内部的`objc_object::rootDealloc`(C++命名空间)函数。

`rootDealloc`最终调用 `objc_destructInstance`，主要完成：

1. 调用 C++ 析构函数
2. 调用 ARC 中实例变量的清理
3. 移除相关对象引用
4. clear 操作

```objective-c
// Replaced by NSZombies
- (void)dealloc {
    _objc_rootDealloc(self); 
}
//省略调用链...
// dealloc方法的核心实现，内部会做判断和析构操作
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        // 判断是否有OC或C++的析构函数
        bool cxx = obj->hasCxxDtor();
        // 对象是否有相关联的引用
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        // 对当前对象进行析构
        if (cxx) object_cxxDestruct(obj);
        // 移除所有对象的关联，例如把weak指针置nil
        if (assoc) _object_remove_assocations(obj);
        obj->clearDeallocating();
    }

    return obj;
}
```

#### 动态添加方法

动态添加、替换方法本质上都是通过调用`addMethod`函数实现。

```c
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return NO;

    rwlock_writer_t lock(runtimeLock);
    return ! addMethod(cls, name, imp, types ?: "", NO);  //若已存在则不添加
}

IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return nil;

    rwlock_writer_t lock(runtimeLock);
    return addMethod(cls, name, imp, types ?: "", YES); 
}

static IMP 
addMethod(Class cls, SEL name, IMP imp, const char *types, bool replace)
{
    IMP result = nil;

    runtimeLock.assertWriting();

    assert(types);
    assert(cls->isRealized());

    method_t *m;
    if ((m = getMethodNoSuper_nolock(cls, name))) {
        // 已经存在实现
        if (!replace) { //如果是添加方法，则返回旧实现
            result = m->imp; 
        } else { //如果是替换方法，则返回被替换的旧实现
            result = _method_setImplementation(cls, m, imp);
        }
    } else {
        //未实现，则向 cls 的 class_rw_t* data() 中的 methods 添加新实现，并且返回 nil
        method_list_t *newlist;
        newlist = (method_list_t *)calloc(sizeof(*newlist), 1);
        newlist->entsizeAndFlags = 
            (uint32_t)sizeof(method_t) | fixed_up_method_list;
        newlist->count = 1;
        newlist->first.name = name;
        newlist->first.types = strdupIfMutable(types);
        newlist->first.imp = imp;

        prepareMethodLists(cls, &newlist, 1, NO, NO);
        cls->data()->methods.attachLists(&newlist, 1);
        flushCaches(cls);

        result = nil;
    }

    return result;
}
```

#### 动态添加实例变量 Ivar

上述可知，不能向已存在（编译得到）的类添加实例变量。只能向通过`Runtime API`创建的类动态添加实例变量`class_addIvar`。

动态创建类方法如下：（顺序不能有误）

1. `objc_allocateClassPair`函数创建类
2. `class_addIvar`添加实例变量、`class_addMethod`添加方法
3. `objc_registerClassPair`函数注册类

> 类在注册完成之后就可以使用了，其实例变量的内存布局也已经确定了(只读)，如果动态为类添加`Ivar`会导致在此之前创建的实例在访问新`Ivar`时导致地址越界

```c
//创建类
Class objc_allocateClassPair(Class superclass, const char *name, 
                             size_t extraBytes)
{
    Class cls, meta;
    rwlock_writer_t lock(runtimeLock);
    if (getClass(name)  ||  !verifySuperclass(superclass, true/*rootOK*/)) {
        return nil;
    }
    cls  = alloc_class_for_subclass(superclass, extraBytes); //创建类
    meta = alloc_class_for_subclass(superclass, extraBytes); //创建元类
	//通过类、元类信息初始化类结构体
    objc_initializeClassPair_internal(superclass, name, cls, meta); 
    return cls;
}

//注册类到map中
void objc_registerClassPair(Class cls)
{
    cls->ISA()->changeInfo(RW_CONSTRUCTED, RW_CONSTRUCTING | RW_REALIZING);
    cls->changeInfo(RW_CONSTRUCTED, RW_CONSTRUCTING | RW_REALIZING);

    addNamedClass(cls, cls->data()->ro->name);
}
```

### 程序加载过程

#### 动态库

在iOS程序中会用到很多系统的动态库（`.tbd`,  `.dylib`, `.framework`），这些动态库都是动态加载的。所有iOS程序共用一套系统动态库，在程序开始运行时才会开始链接动态库。

除了在项目设置里显式出现的动态库外，还会有一些隐式存在的动态库。例如`objc`和`Runtime`所属的`libobjc.dyld`和`libSystem.dyld`，在`libSystem`中包含常用的`libdispatch(GCD)`、`libsystem_c`(C语言基础库)、`libsystem_blocks(Block)`等。

使用动态库的优点：

1. 防止重复。iOS系统中所有`App`公用一套系统动态库，防止重复的内存占用。
2. 减少包体积。因为系统动态库被内置到iOS系统中，所以打包时不需要把这部分代码打进去，可以减小包体积。
3. 动态性。因为系统动态库是动态加载的，所以可以在更新系统后，将动态库换成新的动态库。

#### 静态库

在iOS程序中，所有非系统的库（`.a`, `.framwork`）都是静态库，静态库在链接时会被拷贝到工程的可执行文件中。

`.a`库为纯二进制文件（只有`.m`等的内容），需要配合`.h`文件使用。
`.framwork`库除了有二进制文件外，还有资源文件。

简单来说`.framwork = .a + .h + resource`

> 静态库使用注意点：
> 如果静态库中包含`category`文件时，需要在工程中配置`other linker flags`的值为`-ObjC`。
>
> `other linker flags`：
> 链接器`ld`的额外参数

#### Runtime 加载过程

在应用程序启动后，由`dyld(the dynamic link editor)`进行程序的初始化操作。大概流程就像下面列出的步骤，其中第3、4、5步会执行多次，在`ImageLoader`加载新的`image`进内存后就会执行一次。
`ImageLoader`是`image`的加载器，`image`可以理解为编译后的二进制。

1. 在应用程序启动后，由`dyld`将应用程序加载到二进制中，并完成一些文件的初始化操作。
2. `Runtime`向`dyld`中注册回调函数。
3. 通过`ImageLoader`将所有`image`加载到内存中。
4. `dyld`在`image`发生改变时，主动调用回调函数。
5. `Runtime`接收到`dyld`的函数回调，开始执行`map_images`、`load_images`、`unmap_image`等操作，并回调`+load`方法。
6. 调用`main()`函数，开始执行业务代码。

在`Runtime`加载时，会调用`_objc_init`函数，并在内部注册三个函数指针，同时完成`Runtime`环境的初始化。

```c
void _objc_init(void)
{
    // ....
    environ_init();
    tls_init();
    static_init();
    lock_init();
    exception_init();

    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
}
```
##### map images

`map_images`函数的核心功能是`_read_images`函数，主要完成以下功能：

1. 加载所有类到类的`gdb_objc_realized_classes`表中。
2. 对所有类做重映射。
3. 将所有`SEL`都注册到`namedSelectors`表中。
4. 修复函数指针遗留。
5. 将所有`Protocol`都添加到`protocol_map`表中。
6. 对所有`Protocol`做重映射。
7. 初始化所有非懒加载的类，进行`rw`、`ro`等操作。
8. 遍历已标记的懒加载的类，并做初始化操作。
9. 处理所有`Category`，包括`Class`和`Meta Class`。

##### load images

`load images`的主要功能是：

1. 检索本次回调的`image`中是否有`+load`方法
2. 如果有则准备好对应的`class`和`category`，未实现则退出
3. 调用`+load`方法（`class`优先，`category`最后）

> `+load`方法在`main`函数执行之前执行（且程序运行期间只执行一次），可以用于`Method Swizzling`

```c
void
load_images(const char *path __unused, const struct mach_header *mh)
{
    if (!hasLoadMethods((const headerType *)mh)) return;
    prepare_load_methods((const headerType *)mh);
    call_load_methods();
}
```

##### initialize

`initialize`方法也是由`Runtime`进行调用的，自己不可以直接调用。与`load`方法不同的是，`initialize`方法是在第一次调用类所属的方法时被调用。同时，子类调用`initialize`时，如果父类尚未调用`initialize`，会先调用父类的`initialize`。

```c
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, bool initialize, bool cache, bool resolver);
// ....
// 第一次调用当前类的话，执行initialize的代码
if (initialize  &&  !cls->isInitialized()) {
    _class_initialize (_class_getNonMetaClass(cls, inst));
}
// ....
```

```c
// 第一次调用类的方法，初始化类对象
void _class_initialize(Class cls)
{
    Class supercls;
    bool reallyInitialize = NO;

    // 递归初始化父类。initizlize不用显式的调用super，因为runtime已经在内部调用了
    supercls = cls->superclass;
    if (supercls  &&  !supercls->isInitialized()) {
        _class_initialize(supercls);
    }
    
    {
        monitor_locker_t lock(classInitLock);
        if (!cls->isInitialized() && !cls->isInitializing()) {
            cls->setInitializing();
            reallyInitialize = YES;
        }
    }
    
    if (reallyInitialize) {
        _setThisThreadIsInitializingClass(cls);

        if (MultithreadedForkChild) {
            performForkChildInitialize(cls, supercls);
            return;
        }
        @try {
            // 通过objc_msgSend()函数调用initialize方法
            callInitialize(cls);
        }
        @catch (...) {
            @throw;
        }
        @finally {
            // 执行initialize方法后，进行系统的initialize过程
            lockAndFinishInitializing(cls, supercls);
        }
        return;
    }
    
    else if (cls->isInitializing()) {
        if (_thisThreadIsInitializingClass(cls)) {
            return;
        } else if (!MultithreadedForkChild) {
            waitForInitializeToComplete(cls);
            return;
        } else {
            _setThisThreadIsInitializingClass(cls);
            performForkChildInitialize(cls, supercls);
        }
    }
}
```

### 消息发送

在OC中方法调用是通过`Runtime`实现的，`Runtime`进行方法调用本质上是发送消息，通过`objc_msgSend()`函数进行消息发送。

```c
OBJC_EXPORT void
objc_msgSend(void /* id self, SEL op, ... */ );

OBJC_EXPORT id _Nullable
3456789 
```

该函数有2个隐藏参数：`self`和`_cmd`，且不同类的同名方法中，`_cmd`是同一个对象。因为在`Runtime`中维护了一个`SEL`的表，这个表存储`SEL`不按照类来存储，只要相同的`SEL`就会被看做一个，并存储到表中。在项目加载时，会将所有方法都加载到这个表中，而动态生成的方法也会被加载到表中。

在子类调用父类的方法时，在运行时会创建一个`objc_super`结构体，对父类的调用会被转化为`objc_msgSendSuper()`的调用，其内部实现还是调用`objc_msgSend()`。

```c
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained _Nonnull id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained _Nonnull Class class;
#else
    __unsafe_unretained _Nonnull Class super_class;
#endif
    /* super_class is the first class to search */
};

OBJC_EXPORT void
objc_msgSendSuper(void /* struct objc_super *super, SEL op, ... */ );

objc_msgSend(objc_super->receiver, @selector(class))
```

此时`objc_msgSend()`的参数`objc_super->receiver`依旧是当前对象。因此**当前对象无论调用任何方法，receiver都是当前对象。**

#### 发送过程

在消息发送时，`Runtime`对`selector`查找的过程做了优化，在上文中可以知道，每个类的结构体都有一个`cache`字段，在一个`selector`被调用后就会加入到`cache`中。在下次调用时，优先搜索`cache`中是否存在，如果没有才调用方法列表`method_lis`，然后存入`cache`。

```c
struct cache_t {
    // 存储被缓存方法的哈希表
    struct bucket_t *_buckets;
    // 占用的总大小
    mask_t _mask;
    // 已使用大小
    mask_t _occupied;
}

struct bucket_t {
    cache_key_t _key;
    IMP _imp;
};
```

当调用一个对象的方法时，查找对象的方法，本质上就是遍历对象`isa`所指向类的方法列表，并用调用方法的`SEL`和遍历的`method_t`结构体的`name`字段做对比，如果相等则将`IMP`函数指针返回，如果无匹配则顺着继承链向父类查找，直到`NSObject`。

#### 动态决议

如果一直没有找到对应的方法则会进入动态方法决议步骤。在`if`语句中会判断传入的`resolver`参数是否为`YES`，并且会判断是否已经有过动态决议，因为下面是`goto retry`，所以这段代码可能会执行多次。

```c
if (resolver  &&  !triedResolver) {
    _class_resolveMethod(cls, sel, inst);
    triedResolver = YES;
    goto retry;
}
```

如果满足条件并且是第一次进行动态方法决议，则进入`if`语句中调用`_class_resolveMethod`函数。动态方法决议有两种，`_class_resolveClassMethod`类方法决议和`_class_resolveInstanceMethod`实例方法决议。

```c
BOOL (*msg)(Class, SEL, SEL) = (__typeof__(msg))objc_msgSend;
bool resolved = msg(cls, SEL_resolveInstanceMethod, sel);
```

在这两个动态方法决议的函数实现中，本质上都是通过`objc_msgSend`函数，调用`NSObject`中定义的`resolveInstanceMethod:`和`resolveClassMethod:`两个方法。(`receiver`还是`self`)

可以在这两个方法中动态添加方法，添加方法实现后，会在下面执行`goto retry`，然后再次进入方法查找的过程中。从`triedResolver`参数可以看出，动态方法决议的机会只有一次，如果这次再没有找到，则进入消息转发流程。

> 消息发送的流程：
>
> 1. 判断`SEL`是否需要被忽略，例如`Mac OS`中的垃圾处理机制启动的话，则忽略`retain`、`release`等方法，并返回一个`_objc_ignored_method`的`IMP`，用来标记忽略。
> 2. 判断接收消息的对象是否为`nil`，因为在OC中对`nil`发消息是无效的，这是因为在调用时就通过判断条件过滤掉了。
> 3. 从方法的缓存列表中查找，通过`cache_getImp`函数进行查找，如果找到缓存则直接返回`IMP`。
> 4. 查找当前类的`method list`，查找是否有对应的`SEL`，如果有则获取到`Method`对象，并从`Method`对象中获取`IMP`，并返回`IMP`(这步查找结果是`Method`对象)。
> 5. 如果在当前类中没有找到`SEL`，则去父类中查找。首先查找`cache list`，如果缓存中没有则查找`method list`，并以此类推直到查找到`NSObject`为止。
> 6. 如果在类的继承体系中，始终没有查找到对应的`SEL`，则进入动态方法解析中（只有一次机会）。可以在`resolveInstanceMethod`和`resolveClassMethod`两个方法中动态添加实现。
> 7. 动态消息解析如果没有做出响应，则进入动态消息转发阶段。此时可以在动态消息转发阶段做一些处理，否则就会`Crash`。

#### 消息转发

如果经过上面这些步骤，还是没有找到方法实现的话，则进入动态消息转发中。

```
imp = (IMP)_objc_msgForward_impcache;
cache_fill(cls, sel, imp, inst);
```

1. **fast forwarding** 快速转发

OC通过类的 `-(id)forwardingTargetForSelector:` 将消息转发给其它对象。注：方法返回非`nil`、非`self`则将消息转发给某个对象。返回`self`死锁，返回`nil`，则到下一步：`normal forwarding`。

2. **normal forwarding** 普通转发

OC通过类的 `-(NSMethodSignature *)methodSignatureForSelector:` 方法获得函数的参数和返回值类型的函数签名。方法返回`nil`则报错。否则返回函数签名，并且系统创建`NSInvocation`对象，调用类的` -(void)forwardInvocation: `方法，在此方法中将消息转发出去。