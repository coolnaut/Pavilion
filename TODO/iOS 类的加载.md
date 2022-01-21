## 1. 基础

### 1.1 概述

这篇文档将从`isa `开始，探索iOS runtime。包含类的结构，类的加载，runtime方法的使用等等。开始吧。

### 1.2 objc-private.h

在 `objc-private.h`中定义了如下的编译条件：

```c
#ifdef _OBJC_OBJC_H_
#error include objc-private.h before other headers
#endif
```

也就是说`objc-private.h`该头文件必须在其它头文件之前导入。因为这个头文件中定义了一些最新的结构体，同时其还定义了如下的宏，这些宏在后续都会遇到。

```c++
#define OBJC_TYPES_DEFINED 1
#undef OBJC_OLD_DISPATCH_PROTOTYPES
#define OBJC_OLD_DISPATCH_PROTOTYPES 0
```

避免与其它地方定义的 `id` 和 `Class` 产生冲突（因为在 Objective-C 1.0 和 2.0 中，定义类和对象的结构体是不同的，objc4-750.1 源码有多处分别定义了 `objc_class` 和 `objc_object`，他们通过相关宏来区分）：

### 1.3 宏基础

为什么要说宏呢，因为网上大部分资料都不注意版本信息，有的已经被苹果废弃了，却依旧津津乐道，下面说几个常见地方的宏。

> \__OBJC2__

这个宏定义如下：

```c++
#ifndef __OBJC2__
#   if TARGET_OS_OSX  &&  !TARGET_OS_IOSMAC  &&  __i386__
        // old ABI
#   else
#       define __OBJC2__ 1
#   endif
#endif
```

> OBJC2_UNAVAILABLE
>
> /* OBJC2_UNAVAILABLE: unavailable in objc 2.0, deprecated in Leopard */

这个代表着，OBJC 2.0版本已经不可用了。Objective-C 2.0发布于2006年7月WWDC，有很多的语法改进、runtime改进、垃圾回收机制（已废弃）、支持64 等。同样的`!OBJC2` 之间的代码是Objective-C 2.0之前1.0版本的东西。2.0已经不支持了。

> OBJC_TYPES_DEFINED
>
> \#define OBJC_TYPES_DEFINED 1

在[objc-private.h](https://opensource.apple.com/source/objc4/objc4-750.1/runtime/objc-private.h.auto.html)中定义了`OBJC_TYPES_DEFINED`，因此如果遇到 `if !OBJC_TYPES_DEFINED`表示其中的代码块也是废弃的。

> OBJC_ISA_AVAILABILITY

宏 `OBJC_ISA_AVAILABILITY` 在 `objc-api.h` 文件中定义。旧版本中，类型为 `Class` 的 `isa` 指针在 Objective-C 2.0 中被废弃了。`__OBJC2__`被定义为1。

```c++
/* OBJC_ISA_AVAILABILITY: `isa` will be deprecated or unavailable 
 * in the future */
#if !defined(OBJC_ISA_AVAILABILITY)
#   if __OBJC2__
#       define OBJC_ISA_AVAILABILITY  __attribute__((deprecated))
#   else
#       define OBJC_ISA_AVAILABILITY  /* still available */
#   endif
#endif
```

## 2. 实例对象

### 2.1 isa

先拆解OC的对象模型，从底层的数据结构出发。首先回顾一下，OBJC的对象模型，OBJC对象中有一个`isa`指针，这个指针是一个Class类型。这个指针并不是在结构体定义的，而是IDE方便开发者而产生的。具体需要抽丝剥茧的看看。s

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019112110494749.png)

下面这一块代码其实都是废弃的，具体里面宏都参考【1.2】和【1.3】节。 

```c
typedef struct objc_class *Class;
// objc_class 继承自 objc_object
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};
// 如果objc_object的成员是这个的话，说明又被忽悠了
// 这一块完整的代码是这样，也就是说这是被废弃的
#if !OBJC_TYPES_DEFINED
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;
/// Represents an instance of a class.
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};
/// A pointer to an instance of a class.
typedef struct objc_object *id;
#endif
```

真正的结构体定义在 `objc-private.h`

```c++
struct objc_object {
private:
    isa_t isa; // 可以看到isa不是Class
};
typedef struct objc_class *Class;
// 这个结构体在 objc-runtime-new.
struct objc_class : objc_object {
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
};
typedef struct objc_class *Class;
```

### 2.2 isa_t

```c++
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }
    uintptr_t bits;
private:
    Class cls;
public:
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
#endif
};

```

上面是`isa_t`的关键代码，其余代码都已隐藏。`isa_t`是一个联合体（联合体的基础在这里也不是重点）

`isa_t`联合体中有一个位域`ISA_BITFIELD`，其定义在`isa.h`（位域和联合体都是节省内存的手段）。其中关键的是`nonpointer`和`shiftcls_and_sig`。

```
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL

# if __arm64__
// ARM64 simulators have a larger address space, so use the ARM64e
// scheme even when simulators build for ARM64-not-e.
#   if __has_feature(ptrauth_calls) || TARGET_OS_SIMULATOR
#     define ISA_BITFIELD                                                      \
// ……省略……
#   else
#     define ISA_BITFIELD                                                      \
        uintptr_t nonpointer        : 1;                                       \
        uintptr_t has_assoc         : 1;                                       \
        uintptr_t has_cxx_dtor      : 1;                                       \
        uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/ \
        uintptr_t magic             : 6;                                       \
        uintptr_t weakly_referenced : 1;                                       \
        uintptr_t unused            : 1;                                       \
        uintptr_t has_sidetable_rc  : 1;                                       \
        uintptr_t extra_rc          : 19
        //省略代码
# elif __x86_64__
#   define ISA_BITFIELD                                                        \
// ……省略……

```

> **nonpointer**：表示是否对 **isa** 指针开启指针优化 **0**:纯**isa**指针，**1**:不止是类对象地址**,isa** 中包含了类信息、对象的引用计数等。基本都开启了优化
>
> **has_assoc**：关联对象标志位，**0**没有，**1**存在
>
> **has_cxx_dtor**：是否有 **C++** 或者 **Objc** 的析构器**,**如果有析构函数**,**则需要做析构逻辑**,** 如果没有**,**则可以更快的释放对象
>
> **shiftcls**：存储类指针的值。开启指针优化的情况下，在 **arm64** 架构中有 **33** 位用来存储类指针。
>
> **magic**：用于调试器判断当前对象是真的对象还是没有初始化的空间
>
> **weakly_referenced**：对象是否被指向或者曾经指向一个 **ARC** 的弱变量，没有弱引用的对象可以更快释放。
>
> **deallocating**：对象是否正在释放内存
>
> **has_sidetable_rc**：当对象引用技术大于 **10** 时，则需要借用该变量存储进位
>
> **extra_rc**：对象的引用计数值，实际上是引用计数值减 **1**， 例如，如果对象的引用计数为 **10**，那么 **extra_rc** 为 **9**。如果引用计数大于 **10**， 则需要使用到上面的 **has_sidetable_rc**。
>
> **MACH_VM_MAX_ADDRESS**：这是链接中的知识

### 2.4 isa_t的初始化

在我们`alloc`的底层就会调用到一个方法：`initIsa`。(`initInstanceIsa`，`initProtocolIsa`，`initIsa`等都是重载或封装)

```c++
inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    assert(!isTaggedPointer()); 
    
    if (!nonpointer) { // 当不是优化时，isa的cls会直接执行类对象
        isa.cls = cls;
    } else {
        isa_t newisa(0);
        newisa.bits = ISA_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor; // 是否有 C++ 或者 ARC 的析构函数
        newisa.shiftcls = (uintptr_t)cls >> 3;
        isa = newisa;
    }
}
```



到这里就知道可以通过`shiftcls`拿到类的地址，那么如何拿到类中的数据呢？通过`class_data_bits_t`

```c++
struct objc_class : objc_object {
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
};
```

### 2.3 class_data_bits_t

根据结构体的内存布局和内存对齐，可以很轻易的拿到 `class_data_bits_t bits`的地址。（`cache_t`这个类中也很容易确定大小，可以查看其类结构，最终可得到其大小为 16 字节）。

节省篇幅，留下`class_data_bits_t class_data_bits_t `中关键的地方。`class_rw_t* data() `，`const class_ro_t *safe_ro() const`。

```c++
struct class_data_bits_t {
    friend objc_class;

    // Values are the FAST_ flags above.
    uintptr_t bits;
public:

    class_rw_t* data() const {
        return (class_rw_t *)(bits & FAST_DATA_MASK);
    }
    // Get the class's ro data, even in the presence of concurrent realization.
    // fixme this isn't really safe without a compiler barrier at least
    // and probably a memory barrier when realizeClass changes the data field
    const class_ro_t *safe_ro() const {
        class_rw_t *maybe_rw = data();
        if (maybe_rw->flags & RW_REALIZED) {
            // maybe_rw is rw
            return maybe_rw->ro();
        } else {
            // maybe_rw is actually ro
            return (class_ro_t *)maybe_rw;
        }
    }

};
```

`class_rw_t`如下，这里面就有iOS开发同学熟悉的字段了。

```c++
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint16_t witness;
#if SUPPORT_INDEXED_ISA
    uint16_t index;
#endif
    explicit_atomic<uintptr_t> ro_or_rw_ext;
    Class firstSubclass;
    Class nextSiblingClass;
public:
    void setFlags(uint32_t set)
    {
        __c11_atomic_fetch_or((_Atomic(uint32_t) *)&flags, set, __ATOMIC_RELAXED);
    }

    void clearFlags(uint32_t clear) 
    {
        __c11_atomic_fetch_and((_Atomic(uint32_t) *)&flags, ~clear, __ATOMIC_RELAXED);
    }

    // set and clear must not overlap
    void changeFlags(uint32_t set, uint32_t clear) 
    {
        ASSERT((set & clear) == 0);

        uint32_t oldf, newf;
        do {
            oldf = flags;
            newf = (oldf | set) & ~clear;
        } while (!OSAtomicCompareAndSwap32Barrier(oldf, newf, (volatile int32_t *)&flags));
    }

    class_rw_ext_t *ext() const {
        return get_ro_or_rwe().dyn_cast<class_rw_ext_t *>(&ro_or_rw_ext);
    }

    class_rw_ext_t *extAllocIfNeeded() {
        auto v = get_ro_or_rwe();
        if (fastpath(v.is<class_rw_ext_t *>())) {
            return v.get<class_rw_ext_t *>(&ro_or_rw_ext);
        } else {
            return extAlloc(v.get<const class_ro_t *>(&ro_or_rw_ext));
        }
    }

    class_rw_ext_t *deepCopy(const class_ro_t *ro) {
        return extAlloc(ro, true);
    }
    // 这里是只读一些数据存储，移步查看 class_ro_t 
    const class_ro_t *ro() const {
        auto v = get_ro_or_rwe();
        if (slowpath(v.is<class_rw_ext_t *>())) {
            return v.get<class_rw_ext_t *>(&ro_or_rw_ext)->ro;
        }
        return v.get<const class_ro_t *>(&ro_or_rw_ext);
    }

    void set_ro(const class_ro_t *ro) {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            v.get<class_rw_ext_t *>(&ro_or_rw_ext)->ro = ro;
        } else {
            set_ro_or_rwe(ro);
        }
    }
   // 下面都是动态添加的方法、属性、协议等
   // 方法列表
    const method_array_t methods() const {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            return v.get<class_rw_ext_t *>(&ro_or_rw_ext)->methods;
        } else {
            return method_array_t{v.get<const class_ro_t *>(&ro_or_rw_ext)->baseMethods()};
        }
    }
  // 属性列表
    const property_array_t properties() const {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            return v.get<class_rw_ext_t *>(&ro_or_rw_ext)->properties;
        } else {
            return property_array_t{v.get<const class_ro_t *>(&ro_or_rw_ext)->baseProperties};
        }
    }
  // 协议列表
    const protocol_array_t protocols() const {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            return v.get<class_rw_ext_t *>(&ro_or_rw_ext)->protocols;
        } else {
            return protocol_array_t{v.get<const class_ro_t *>(&ro_or_rw_ext)->baseProtocols};
        }
    }
};
```

最终都归纳到`class_rw_ext_t`中。

```c++
struct class_rw_ext_t {
    DECLARE_AUTHED_PTR_TEMPLATE(class_ro_t)
    class_ro_t_authed_ptr<const class_ro_t> ro;
    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
    char *demangledName;
    uint32_t version;
};c
```

下面看看 `class_ro_t`这个只读的类，这里面都是确定的成员，成员方法等，不是通过runtime方式加入的。

```c++
struct class_ro_t {
// 方法列表获取
    void *baseMethodList;
// 确定的协议列表
    protocol_list_t * baseProtocols;
// 确定的成员列表
    const ivar_list_t * ivars;
    const uint8_t * weakIvarLayout;
// 确定的属性列表
    property_list_t *baseProperties;

};
```



## 附

### objc_class 完整版

```c++

```

https://juejin.cn/post/6973315947574591502

https://juejin.cn/post/6975878547265208356

https://juejin.cn/post/6975781940519501861

https://www.jianshu.com/p/6c21d4e8b04b

https://juejin.cn/post/6844903587965861896

https://juejin.cn/post/6921348387501834254

https://juejin.cn/post/6844903815028670477



