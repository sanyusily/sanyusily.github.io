---
layout: article
title: 'Android 源码 RefBase，sp,wp'
tags: Android RefBase sp wp
---

# Android P源码分析之RefBase & sp,wp 

# 一﹑RefBase类

## 简介

  **RefBase类引入了计数概念,当类的引用计数为0的时候会自动释放类对象.该方式代替传统手动释放类对象**

## 示例

**类定义**  

```
class LocalRefBase : public RefBase{
public:
	LocalRefBase() {
		printf("LocalRefBase create\n");
	}
	virtual ~LocalRefBase() {
		printf("LocalRefBase destory\n");
	}
	void printfString() {
		printf("LocalRefBase printfString\n");
	}
protected:
	virtual void onFirstRef() {
		printf("LocalRefBase onFirstRef\n");
	}
	virtual void onLastStrongRef(const void* id __unused) {
		printf("LocalRefBase onLastStrongRef\n");
	}
};
```

**传统方式**
```
LocalRefBase *pLocalRb = new LocalRefBase(); 
delete pLocalRb;
```

**运行结果**
```
LocalRefBase create
LocalRefBase destory
```
**RefBase类计数方式**
```
LocalRefBase *pLocalRb = new LocalRefBase(); /* 见1.1 */
pLocalRb -> incStrong(pLocalRb); /* 强引用计数+1,当前引用计数为1,见1.2 */
pLocalRb -> decStrong(pLocalRb); /* 强引用计数-1,当前引用计数为0,释放对象,见1.3 */
```

**运行结果**
```
LocalRefBase create
LocalRefBase onFirstRef	     /* 强引用计数首次增加时,回调该函数 */ 
LocalRefBase onLastStrongRef /* 强引用计数减少到0时,回调该函数 */
LocalRefBase destory
```
**相同点**:构建对象时,都是用的new方式构建对象  
**差异点**:释放对象时,**传统方式**需**手动调用delete方法**,而**计数方式**则是**当强引用计数达到0时释放对象(默认生命周期)**

## 源码分析

**1.1 构造函数**
```
RefBase::RefBase()
    : mRefs(new weakref_impl(this))  /* 构建weakref_impl,该类用于记录强弱引用计数 */
{
}

weakref_impl(RefBase* base)
	: mStrong(INITIAL_STRONG_VALUE)	/* 初始化强引用计数 */
	, mWeak(0)	  /* 初始化弱引用计数 */ 
	, mBase(base) /* 记录RefBase类对象 */
	, mFlags(0)	  /* flag = OBJECT_LIFETIME_STRONG(0),代表强引用生命周期 */
{
}
```
**总结**  
  **1.RefBase构建weakref_impl类,用于记录计数情况**  
  **2.初始化强引用计数为INITIAL_STRONG_VALUE,弱引用计数为0,默认强引用生命周期**

**1.2 incStrong**

```
void RefBase::incStrong(const void* id) const
{
......
    weakref_impl* const refs = mRefs;
    refs->incWeak(id);	/* 弱引用计数mWeak加1 */ 

    /* 原子操作,强引用计数mStrong加1,返回旧值 */
    const int32_t c = refs->mStrong.fetch_add(1, std::memory_order_relaxed); 

    if (c != INITIAL_STRONG_VALUE)  {	
    /* 当旧值不为INITIAL_STRONG_VALUE(非首次增加强引用)时,直接返回 */
        return;
    }
    /* 当首次增加强引用时,原子操作 mStrong = mStrong - INITIAL_STRONG_VALUE = 1
     * 并调用成员函数onFirstRef
     */
    int32_t old __unused = refs->mStrong.fetch_sub(INITIAL_STRONG_VALUE, std::memory_order_relaxed);
    refs->mBase->onFirstRef();
......
}
```

**总结**  
  **1.强引用计数变量mStrong加1,如果是首次增加强引用,则回调成员函数onFirstRef**  
  **2.弱引用计数变量mWeak加1**

**1.3 decStrong**  

```
void RefBase::decStrong(const void* id) const
{
......
    weakref_impl* const refs = mRefs;

    /* 原子操作,强引用计数mStrong减1,返回旧值 */
    const int32_t c = refs->mStrong.fetch_sub(1, std::memory_order_release); 

    if (c == 1) {
    /* 当强引用计数旧值为1(当前强引用计数为0) 
     * 1. 调用成员函数onLastStrongRef
     * 2. 强引用生命周期(默认)时,释放RefBase对象(pLocalRb)
     */
        std::atomic_thread_fence(std::memory_order_acquire);
        refs->mBase->onLastStrongRef(id);
        int32_t flags = refs->mFlags.load(std::memory_order_relaxed);
        if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
            delete this;
            // The destructor does not delete refs in this case.
        }
    }

    refs->decWeak(id); /* 弱引用计数减1,见1.3.1 */
......
}
```
**1.3.1 decWeak**

```
void RefBase::weakref_type::decWeak(const void* id)
{
......
    weakref_impl* const impl = static_cast<weakref_impl*>(this);

    /* 原子操作,弱引用计数mWeak减1,返回旧值 */
    const int32_t c = impl->mWeak.fetch_sub(1, std::memory_order_release); 

    if (c != 1) return; /* 当旧值不为1(当前弱引用计数不为0)时,返回 */

    /* 弱引用计数为0流程 */
    int32_t flags = impl->mFlags.load(std::memory_order_relaxed);
    if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
    /* 1. 强引用生命周期(默认) 
     * 2. 强引用计数mStrong非初始值INITIAL_STRONG_VALUE
     * 当同时满足以上2个条件时,释放记录引用计数的对象mRefs(pLocalRb -> mRefs)
     */
        if (impl->mStrong.load(std::memory_order_relaxed)
                == INITIAL_STRONG_VALUE) {
            ALOGW("RefBase: Object at %p lost last weak reference "
                    "before it had a strong reference", impl->mBase);
        } else {
            delete impl;
        }
    } else {
     /* 弱引用生命周期
      * 回调成员函数onLastWeakRef,并释放RefBase对象(pLocalRb)
      */
        impl->mBase->onLastWeakRef(id);
        delete impl->mBase;
    }
......
}
```
**总结**  
  **1.强引用计数mStrong减1,如果强引用计数为0,则回调成员函数onLastStrongRef**  
  **2.弱引用计数mWeak减1**  
  **3.**

|                    | 强引用生命周期                      | 弱引用生命周期             |
| ------------------ | ---------------------------- | ------------------- |
| **强引用计数mStrong为0** | 释放RefBase对象                  | 无动作                 |
| **弱引用计数mWeak为0**   | 当强引用计数非初始值时,释放记录引用计数的对象mRefs | 释放RefBase对象和mRefs对象 |

## 小结

![class_lifetime.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d64435f16c8b4e2a9f2ea42808e17ef8~tplv-k3u1fbpfcp-watermark.image?)

**从上面的分析中,可以总结出如上图所示的生命周期关系**  
**这里值得一提的是,强引用生命周期时**  
**mRefs的生命周期 ≥ RefBase的生命周期的原因:**  
**1.强引用计数的增加函数,除了强引用计数mStrong加1外,弱引用计数也会mWeak也会加1**  
**此时mStrong == mWeak**  
**2.弱引用计数的增加函数,只会单独对弱引用计数mWeak加1.此时mWeak > mStrong**  
**综上可得结论 mWeak ≥ mStrong即mRefs的生命周期 ≥ RefBase的生命周期**

# 二﹑普通变量和指针变量

  **我们在讲述sp和wp前,先来看看指针变量和普通变量的生命周期对类对象生命周期的影响**

## 普通变量
```
int main(int argc __unused, char **argv __unused) {
	LocalRefBase pLocalRb;
	return 0;
}
```
**运行结果**  
LocalRefBase create  
LocalRefBase destory

**结论**  
  **普通变量在生命周期开始时,自动调用类的构造函数.生命周期结束时,自动调用类的析构函数并释放类对象**

## 指针变量
```
int main(int argc __unused, char **argv __unused) {
	LocalRefBase *pLocalRb = new LocalRefBase();
	return 0;
}
```
**运行结果**  
LocalRefBase create

**结论**  
  **当从堆中申请类对象所需的内存成功后,调用类的构造函数,指针变量生命周期结束时,不会自动调用类的析构函数,需要手动执行delete方法释放对象内存**

## 小结

  **1.普通变量在其生命周期结束时,会自动调用类的析构函数并释放类对象,而指针变量则需要手动调用delete方法.**  
  **2.sp和wp类就是利用了普通变量的特性,在生命周期开始时关联一个类对象,在构造函数中对关联的类对象引用计数加1.生命周期结束时,在析构函数中对关联的类对象引用计数减1**

# 三﹑sp模板类

## 简介

  **sp全称为strong point:强引用指针,该类的模板类型必须为RefBase类的子类.该类被设计用于智能释放RefBase类对象,解决内存泄露问题**

## 示例

```
int main(int argc __unused, char **argv __unused) {
    sp<LocalRefBase> spLocalRb(new LocalRefBase()); /* 见下源码分析 */
    return 0;
}
```
**运行结果**  
LocalRefBase create  
LocalRefBase onFirstRef  
LocalRefBase onLastStrongRef  
LocalRefBase destory

## 源码分析

构造函数

```
template<typename T>
sp<T>::sp(T* other)
: m_ptr(other) {
    if (other) {
    /* 调用RefBase类的incStrong函数:强引用计数mStrong加1,弱引用计数mWeak加1
     * 此时mStrong = 1,mWeak = 1.首次增加强引用计数,回调RefBase类成员函数onFirstRef
     */
   	    other->incStrong(this);
     }
}
```
析构函数
```
template<typename T>
sp<T>::~sp() {
    if (m_ptr) {
    /* 调用RefBase类的decStrong函数:强引用计数mStrong减1,弱引用计数mWeak减1
     * 此时mStrong = 0,mWeak = 0.强引用计数为0,回调RefBase类成员函数onLastStrongRef并释放RefBase对象.弱引用计数为0,释放mRefs对象
     */
   	    m_ptr->decStrong(this);
    }
}
```

## 小结

  **1.sp类利用普通变量的特性:**  
  **生命周期开始时:构造函数调用关联的RefBase对象的incStrong函数(强弱引用计数加1)**  
  **生命周期结束时:析构函数调用关联的RefBase对象的decStrong函数(强弱引用计数减1)**  
  **2.由于sp类的生命周期关联着强弱引用计数,因此sp类间接关联RefBase和mRefs的生命周期**

# 四﹑wp模板类

## 简介

  **wp全称为weak point:弱引用指针,该类的模板类型必须为RefBase类的子类**  
  **由于wp仅关联弱引用计数mWeak,因此在强引用生命周期(默认)时,wp不会关联RefBase的生命周期**

## 示例

```
int main(int argc __unused, char **argv __unused) {	
    /* 构建wp类对象,构造参数为关联对象LocalRefBase */
    wp<LocalRefBase> wpLocalRb(new LocalRefBase());
    for (;;) {
        printf("for scope begin\n");
        /* 默认强生命周期下,wp由于不持有强引用计数mStrong 
         * 因此无法保证关联对象LocalRefBase的生存情况,
         * 所以我们不能直接操作关联对象LocalRefBase,防止出现段错误crash,
         * wp类提供了个安全方法promote,该方法会检测LocalRefBase的生存情况
         * 如果关联对象LocalRefBase未被释放,则会返回非空sp,否则返回空sp.
         */
        sp<LocalRefBase> spLocalRb = wpLocalRb.promote();	
        if (spLocalRb == NULL) {  /* 这里需要做非空判断,确保关联对象存在 */
            printf("spLocalRb was null\n");
            break;
        }
        spLocalRb -> printfString();
        printf("for scope end\n");
    }
    return 0;
}
```

**运行结果**  
LocalRefBase create  
for scope begin  
LocalRefBase printfString  
for scope end  
LocalRefBase onLastStrongRef  
LocalRefBase destory  
for scope begin  
spLocalRb was null

**spLocalRb为空sp分析**  
  **1. 第一次for循环结束时,在作用域内的spLocalRb对象生命周期也跟随结束,因此会调用sp析构函数对**  
**关联对象LocalRefBase强弱引用计数减1,此时由于强引用计数归0,因此会释放关联对象LocalRefBase**  
  **2. 第二次for循环到来时,由于关联对象LocalRefBase已被释放因此wp类成员函数promote返回空的sp**

## 源码分析

构造函数
```
template<typename T>
wp<T>::wp(T* other)
    : m_ptr(other)
{
    if (other) m_refs = other->createWeak(this); /* 见下 */
}

RefBase::weakref_type* RefBase::createWeak(const void* id) const
{
    mRefs->incWeak(id);	/* 弱引用计数加1,此时mWeak = 1 */
    return mRefs;	/* 返回记录引用计数对象mRefs */
}
```
promote函数
```
template<typename T>
sp<T> wp<T>::promote() const
{
    sp<T> result;
if (m_ptr && m_refs->attemptIncStrong(&result)) {
/* 调用mRefs的attemptIncStrong方法,该方法返回关联对象生存情况 见下 */
        result.set_pointer(m_ptr); /* sp类关联对象赋值 */
    }
    return result;
}

bool RefBase::weakref_type::attemptIncStrong(const void* id)
{
......
    incWeak(id); /* 弱引用计数mWeak加1 */
    
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    int32_t curCount = impl->mStrong.load(std::memory_order_relaxed);

    while (curCount > 0 && curCount != INITIAL_STRONG_VALUE) {
    /* 强引用计数mStrong大于0且不为初始值时
     * 强引用计数mStrong加1,并返回true: 代表关联对象可用
     */
        if (impl->mStrong.compare_exchange_weak(curCount, curCount+1,
                std::memory_order_relaxed)) {
            break;
        }
    }
    
    if (curCount <= 0 || curCount == INITIAL_STRONG_VALUE) {
        int32_t flags = impl->mFlags.load(std::memory_order_relaxed);
        if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
        /* 强生命周期时
          *--------------------------------------------------------------------------------------------------------------------------
          * 如果强引用计数mStrong小于等于0
          * 说明关联对象生命周期已结束
          * 因此对弱引用计数mWeak减1(对应开头加1)并返回false
          * --------------------------------------------------------------------------------------------------------------------------
          * 如果强引用计数mStrong为初始值
          * 则对强引用计数mStrong加1减INITIAL_STRONG_VALUE,返回true: 代表关联对象可用
          * --------------------------------------------------------------------------------------------------------------------------
        */
            if (curCount <= 0) {
                // the last strong-reference got released, the object cannot
                // be revived.
                decWeak(id);
                return false;
            }
            while (curCount > 0) {
                if (impl->mStrong.compare_exchange_weak(curCount, curCount+1,
                        std::memory_order_relaxed)) {
                    break;
                }
            }

        } else {
        /* 弱生命周期时
         * 对强引用计数mStrong加1并返回true:代表关联对象可用
         */
            curCount = impl->mStrong.fetch_add(1, std::memory_order_relaxed);
        }
    }
    if (curCount == INITIAL_STRONG_VALUE) {
        impl->mStrong.fetch_sub(INITIAL_STRONG_VALUE,
                std::memory_order_relaxed);
    }

    return true;
......
}
```
## 小结

**1.wp类利用普通变量的特性:**  
**生命周期开始时:构造函数调用关联的RefBase对象的createWeak函数(弱引用计数加1并返回对象mRefs).**  
**生命周期结束时:析构函数调用关联的mRefs对象的decWeak函数(弱引用计数减1)**  
**2.由于wp类的生命周期仅关联着弱引用计数mWeak,因此**  
**强生命周期时,wp间接关联mRefs的生命周期**  
**弱生命周期时,wp间接关联RefBase和mRefs的生命周期**  
**3.想使用关联对象时,需调用wp成员函数promote,得到sp对象,并且要对该sp对象进行非空判断.**

# 五﹑全文总结

![UML_RefBase&wp&sp.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4aaf8bbdeaaa4f57bdc03a94b82e4bba~tplv-k3u1fbpfcp-watermark.image?)

![class_lifetime_sp_wp.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99959ce5b36944c7be88b2f7d50ef223~tplv-k3u1fbpfcp-watermark.image?)
**强引用生命周期(默认)**  
**1.强引用计数mStrong关联RefBase类对象的生命周期**  
**2.弱引用计数mWeak关联RefBase类成员mRefs的生命周期,该类成员mRefs用于记录强弱引用计数情况**  
**3.sp类的生命周期内持有关联对象的强弱引用计数,因此sp类间接关联RefBase对象和mRefs对象的生命周期**  
**4.wp类的生命周期内持有关联对象的弱引用计数,因此wp类间接关联mRefs对象的生命周期**

**弱引用生命周期**  
**1.强引用计数mStrong不关联任何对象的生命周期**  
**2.弱引用计数mWeak关联RefBase类对象和mRefs对象的生命周期**  
**3.sp类的生命周期内持有关联对象的强弱引用计数,因此sp类间接关联RefBase对象和mRefs对象的生命周期**  

