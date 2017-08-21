---
layout: post
title: COM组件弱引用的简单实现
date: 2010-08-23
categories: tech
tags: [com]
---

说明：我们知道boost用shared_ptr,weak_ptr实现了指针的智能化管理，使用它们可以防止C＋＋常见的内存泄露问题。COM组件的管理和指针类似却又不同，COM组件同样需要在使用的时候调用AddRef和Release来管理组件的引用计数，为了方便管理，ATL库提供了CComPtr等智能组件指针来简化COM组件的使用。可是我的问题是如何去观察一个com组件而不影响它的引用计数，并能在合适的时候将观察对象转换成真实的COM引用。

从COM组件的实现原理，我们可以知道COM组件的弱引用并不好实现。原因在于COM组件的引用计数是包含在COM对象中的，如此，当COM对象被销毁时，也就无法通过获取它的引用计数值来判断该COM对象是否存在了，那么弱引用也就无法判断何时可以将弱引用转换成强引用了。

那么boost的weak_ptr是如何实现的呢？很简单，boost中的shared_ptr,weak_ptr将引用计数和对象本身分离开，这样就算对象本身被销毁了，只要还有弱引用仍在观察该对象，那么弱引用都可以通过判断引用计数中的值来判断对象是否销毁，这样就可以在对象已经销毁后返回失败的强引用即可。

那么COM组件可以借鉴boost的这种实现方式。但是有几个问题：
1. 引入额外的引用计数对象后，如何处理内部引用计数和外部引用计数的一致性。
1. 由于需要保证内部引用计数和外部引用计数的一致性，线程安全就成了一个问题，因为需要保证对两个计数变量同时增减，单一变量可以用原子增减，多个变量似乎只能用锁来解决了。Linux下面用pthread_mutex_t来实现锁，windows下可以用关键代码段。

我把问题简化了，因此我只是简单实现了类似boost的CComSharedPtr,CComWeakPtr，对于上述两个问题，首先 CcomWeakPtr只能观察由CComSharedPtr封装的COM组件，因此，如果使用了CComPtr来引用一个组件，那么CComWeakPtr是观察不到的，也就是此时即使CComPtr还有效，如果CComSharedPtr管理的组件引用全部失效，那么CComWeakPtr仍然会返回失败。第二，CComSharedPtr中的计数只包含由CComSharedPtr管理的对象个数，而不一定是实际COM组件对象被引用的个数。这样，COM组件对象的引用就包括两个部分：CComSharedPtr管理的和其他指针直接管理的。

因此CComSharedPtr的使用需要注意：若是整个项目的COM对象全部由CComSharedPtr封装管理，那么CComSharedPtr和 CcomWeakPtr会工作的很好。如果项目代码中还有CComPtr等其他指针对COM进行引用的话，那么CComSharedPtr一切正常使用，但是CComWeakPtr在将COM弱引用转换成强引用时，即使失败了，也不代表COM对象被销毁了，只能代表已经没有对应的CComSharedPtr管理的COM引用了。

最后，该实现不是线程安全的，如果需要的话，可以修改代码的增减引用计数部分（具体修改可以参见代码注释的pthread_mutex_t部分，windows下可以用关键代码段）。另外，代码草率完成，定有不适之处，有什么问题可以留言告知。

PS:COM组件的弱引用的需求在实际项目中可能是设计不当的结果，在此实现完全是为了熟悉弱引用的实现原理。
以下是代码实现, 借鉴了boost的大量源码：

```C++
#include <stdio.h>
#include <memory>
#include <boost/smart_ptr.hpp>
#include <pthread.h>
#include <unistd.h>
 using namespace std;
//这个基类就是为了在com组件之外添加额外的引用计数对象，这个是因为当com对象被销毁时无法
//访问com内部的引用来探测com对象是否存在，而通过引入额外的引用计数对象，我们可以通过
//使用该com对象的弱引用来探测com对象是否存在，直到所有观察该com对象的弱引用全部销毁后，
//额外的引用计数对象才会销毁自身。这样就达到了观察com对象却不影响com对象引用计数的目的。
//缺点：需要额外维护两个引用计数的一致性，因此无法做到线程安全。而且只能保证维护
//CComSharedPtr所管理的com对象，其他如CComPtr不在管理范围内，因此相应的弱引用也只能观察
//由CComSharedPtr所管理的com对象，当所有的CComSharedPtr管理的com对象销毁后，弱引用返回
class com_counted_base
{
private:

    com_counted_base( com_counted_base const & );
    com_counted_base & operator= ( com_counted_base const & );

    long use_count_;        // #shared
    long weak_count_;       // #weak + (#shared != 0)

//    pthread_mutex_t refmutex;
public:

    com_counted_base(): use_count_( 1 ), weak_count_( 1 )
    {
//        pthread_mutex_init(&refmutex,NULL);
    }

    virtual ~com_counted_base() // nothrow
    {
//        pthread_mutex_destroy(&refmutex);
    }

    // dispose() is called when use_count_ drops to zero, to release
    // the resources managed by *this.

    virtual void dispose() = 0; // nothrow
    virtual ULONG com_add_ref() = 0;
    virtual ULONG com_release() = 0;

    // destroy() is called when weak_count_ drops to zero.

    virtual void destroy() // nothrow
    {
        delete this;
    }

    void add_ref_copy()
    {
//        pthread_mutex_lock(&refmutex);
        ++use_count_;
        //如果需要测试线程安全，可以在此加入sleep
//        usleep(1);
        com_add_ref();
//        pthread_mutex_unlock(&refmutex);
    }

    bool add_ref_lock() // true on success
    {

        if( use_count_ == 0 ) return false;
        add_ref_copy();
        return true;
    }

    void release() // nothrow
    {
//        pthread_mutex_lock(&refmutex);
        com_release();
        //如果需要测试线程安全，可以在此加入sleep
//        usleep(1);
        if( --use_count_ == 0 )
        {
            dispose();
            weak_release();
        }
//        pthread_mutex_unlock(&refmutex);
    }
    void weak_add_ref() // nothrow
    {
        ++weak_count_;
    }

    void weak_release() // nothrow
    {
        if( --weak_count_  == 0 )
        {
            destroy();
        }
    }

    long use_count() const // nothrow
    {
        return static_cast<long const volatile &>( use_count_ );
    }
};
template <typename T> class com_counted_impl: public com_counted_base
{
private:
    T * px_;

    com_counted_impl( com_counted_impl const & );
    com_counted_impl & operator= ( com_counted_impl const & );

    typedef com_counted_impl this_type;

public:
    explicit com_counted_impl( T * px, bool isaddrefneeded): px_( px )
    {
        if(px_!=NULL && isaddrefneeded)
            com_add_ref();
    }
    virtual ULONG com_add_ref()
    {
        return px_->AddRef();
    }
    virtual ULONG com_release()
    {
        return px_->Release();
    }
    virtual void dispose() // nothrow
    {
        //普通数据指针这里需要delete自己，但是com组件指针不用，因为当com组件引用降为0时会删除自己，和普通数据指针不同
    }
};
class com_weak_count;

class com_shared_count
{
private:

    com_counted_base* pi_;

    friend class com_weak_count;

public:

    com_shared_count(): pi_(0) // nothrow
    {
    }

    template <typename Y> explicit com_shared_count( Y * p, bool isaddrefneeded = true ): pi_( 0 )
    {
        pi_ = new com_counted_impl<Y>( p , isaddrefneeded);

        if( pi_ == 0 )
        {
            p->Release();
            boost::throw_exception( std::bad_alloc() );
        }
    }

    ~com_shared_count() // nothrow
    {
        if( pi_ != 0 ) pi_->release();
    }

    com_shared_count(com_shared_count const & r): pi_(r.pi_) // nothrow
    {
        if( pi_ != 0 ) pi_->add_ref_copy();
    }

    explicit com_shared_count(com_weak_count const & r); // throws bad_weak_ptr when r.use_count() == 0

    com_shared_count & operator= (com_shared_count const & r) // nothrow
    {
        com_counted_base * tmp = r.pi_;

        if( tmp != pi_ )
        {
            if( tmp != 0 ) tmp->add_ref_copy();
            if( pi_ != 0 ) pi_->release();
            pi_ = tmp;
        }
        return *this;
    }
    void swap(com_shared_count & r) // nothrow
    {
        com_counted_base * tmp = r.pi_;
        r.pi_ = pi_;
        pi_ = tmp;
    }

    long use_count() const // nothrow
    {
        return pi_ != 0? pi_->use_count(): 0;
    }

    bool unique() const // nothrow
    {
        return use_count() == 1;
    }

    friend inline bool operator==(com_shared_count const & a, com_shared_count const & b)
    {
        return a.pi_ == b.pi_;
    }

};

class com_weak_count
{
private:

    com_counted_base * pi_;
    friend class com_shared_count;

public:

    com_weak_count(): pi_(0) // nothrow
    {
    }

    com_weak_count(com_shared_count const & r): pi_(r.pi_) // nothrow
    {
        if(pi_ != 0) pi_->weak_add_ref();
    }

    com_weak_count(com_weak_count const & r): pi_(r.pi_) // nothrow
    {
        if(pi_ != 0) pi_->weak_add_ref();
    }

    ~com_weak_count() // nothrow
    {
        if(pi_ != 0) pi_->weak_release();
    }

    com_weak_count & operator= (com_shared_count const & r) // nothrow
    {
        com_counted_base * tmp = r.pi_;
        if(tmp != 0) tmp->weak_add_ref();
        if(pi_ != 0) pi_->weak_release();
        pi_ = tmp;

        return *this;
    }

    com_weak_count & operator= (com_weak_count const & r) // nothrow
    {
        com_counted_base * tmp = r.pi_;
        if(tmp != 0) tmp->weak_add_ref();
        if(pi_ != 0) pi_->weak_release();
        pi_ = tmp;

        return *this;
    }

    void swap(com_weak_count & r) // nothrow
    {
        com_counted_base * tmp = r.pi_;
        r.pi_ = pi_;
        pi_ = tmp;
    }

    long use_count() const // nothrow
    {
        return pi_ != 0? pi_->use_count(): 0;
    }

    friend inline bool operator==(com_weak_count const & a, com_weak_count const & b)
    {
        return a.pi_ == b.pi_;
    }
};
inline com_shared_count::com_shared_count( com_weak_count const & r ): pi_( r.pi_ )
{
    if( pi_ == 0 || !pi_->add_ref_lock() )
    {
        boost::throw_exception( boost::bad_weak_ptr() );
    }
}
template <typename T> class CComWeakPtr;
template <typename T> class CComSharedPtr
{
private:
    T* px;
    com_shared_count pn;
    typedef CComSharedPtr<T> this_type;
public:

    CComSharedPtr(): px(0), pn() // never throws in 1.30+
    {
    }
    ~CComSharedPtr()
    {

    }
    explicit CComSharedPtr( T* p ): px( p ), pn( p )
    {
    }
    //  generated copy constructor, assignment, destructor are fine...

    template <typename Y> CComSharedPtr & operator=(CComSharedPtr<Y> const & r) // never throws
    {
        px = r.px;
        pn = r.pn; // shared_count::op= doesn't throw
        return *this;
    }
    template <typename Y> explicit CComSharedPtr(CComWeakPtr<Y> const & r): pn(r.pn) // may throw
    {
        // it is now safe to copy r.px, as pn(r.pn) did not throw
        px = r.px;
    }

    template<typename Y> CComSharedPtr(CComSharedPtr<Y> const & r): px(r.px), pn(r.pn)
    {

    }
    template<class Y>
    CComSharedPtr(CComSharedPtr<Y> const & r, boost::detail::dynamic_cast_tag): px(dynamic_cast<T *>(r.px)), pn(r.pn)
    {
        if(px == 0) // need to allocate new counter -- the cast failed
        {
            pn = com_shared_count();
        }
    }

    operator T*() const
    {
        return px;
    }

    //使用该操作符返回的指针被修改指向新的com组件后，必须调用init_count函数进行计数初始化
    T** operator&() throw()
    {
        //     BOOST_ASSERT(px==NULL);
        return &px;
    }

    void init_com_count()
    {
        //初始化通过获取原始数据指针直接修改的com引用，由于直接修改原始com引用，跳过了
        //额外引用计数的初始化，因此需要调用该函数进行额外计数初始化。
        //由于直接修改的com引用必然是已经调用过AddRef的，因此传递false以表示不再需要AddRef了。
        com_shared_count temp(px,false);
        pn = temp;
    }

    void reset() // never throws in 1.30+
    {
        this_type().swap(*this);
    }

    void reset(T * p) // Y must be complete
    {
        BOOST_ASSERT(p == 0 || p != px); // catch self-reset errors
        this_type(p).swap(*this);
    }
    T& operator* () const // never throws
    {
        BOOST_ASSERT(px != 0);
        return *px;
    }

    T * operator-> () const // never throws
    {
        BOOST_ASSERT(px != 0);
        return px;
    }

    T * get() const // never throws
    {
        return px;
    }
    // implicit conversion to "bool"
    operator bool () const
    {
        return px != 0;
    }
    // operator! is redundant, but some compilers need it
    bool operator! () const // never throws
    {
        return px == 0;
    }

    bool unique() const // never throws
    {
        return pn.unique();
    }

    long use_count() const // never throws
    {
        return pn.use_count();
    }
    void swap(CComSharedPtr<T>& other)
    {
        std::swap(px, other.px);
        pn.swap(other.pn);
    }
private:
    template<class Y> friend class CComSharedPtr;
    template<class Y> friend class CComWeakPtr;
};
template<class T, class U> CComSharedPtr<T> dynamic_pointer_cast(CComSharedPtr<U> const & r)
{
    return CComSharedPtr<T>(r, boost::detail::dynamic_cast_tag());
}
template <typename T> class CComWeakPtr
{
private:

    typedef CComWeakPtr<T> this_type;

public:

    CComWeakPtr(): px(0), pn() // never throws in 1.30+
    {
    }

    //  generated copy constructor, assignment, destructor are fine


    //
    //  The "obvious" converting constructor implementation:
    //
    //  template<class Y>
    //  weak_ptr(weak_ptr<Y> const & r): px(r.px), pn(r.pn) // never throws
    //  {
    //  }
    //
    //  has a serious problem.
    //
    //  r.px may already have been invalidated. The px(r.px)
    //  conversion may require access to *r.px (virtual inheritance).
    //
    //  It is not possible to avoid spurious access violations since
    //  in multithreaded programs r.px may be invalidated at any point.
    //

    template <typename Y> CComWeakPtr(CComWeakPtr<Y> const & r): pn(r.pn) // never throws
    {
        px = r.lock().get();
    }

    template <typename Y> CComWeakPtr(CComSharedPtr<Y> const & r): px(r.px), pn(r.pn) // never throws
    {
    }

    template <typename Y> CComWeakPtr & operator=(CComWeakPtr<Y> const & r) // never throws
    {
        px = r.lock().get();
        pn = r.pn;
        return *this;
    }
    template <typename Y> CComWeakPtr & operator=(CComSharedPtr<Y> const & r) // never throws
    {
        px = r.px;
        pn = r.pn;
        return *this;
    }

    CComSharedPtr<T> lock() const // never throws
    {
#if defined(BOOST_HAS_THREADS)

        // optimization: avoid throw overhead
        if(expired())
        {
            return CComSharedPtr<T>();
        }

        try
        {
            return CComSharedPtr<T>(*this);
        }
        catch(boost::bad_weak_ptr const &)
        {
            // Q: how can we get here?
            // A: another thread may have invalidated r after the use_count test above.
            return CComSharedPtr<T>();
        }

#else

        // optimization: avoid try/catch overhead when single threaded
        return expired()? CComSharedPtr<T>(): CComSharedPtr<T>(*this);

#endif
    }

    long use_count() const // never throws
    {
        return pn.use_count();
    }

    bool expired() const // never throws
    {
        return pn.use_count() == 0;
    }

    void reset() // never throws in 1.30+
    {
        this_type().swap(*this);
    }

    void swap(this_type & other) // never throws
    {
        std::swap(px, other.px);
        pn.swap(other.pn);
    }

    // Tasteless as this may seem, making all members public allows member templates
    // to work in the absence of member template friends. (Matthew Langston)

#ifndef BOOST_NO_MEMBER_TEMPLATE_FRIENDS

private:

    template<class Y> friend class CComWeakPtr;
    template<class Y> friend class CComSharedPtr;

#endif

    T * px;                       // contained pointer
    com_weak_count pn; // reference counter

};
HRESULT  CreateComTestInstance(const IID& iid,void** ppv)
{
    ComTest *t = new ComTest;
    HRESULT hr = t->QueryInterface(iid,ppv);

    t->Release();
    return hr;
}
class SharedTest
{

public:
    CComSharedPtr<IPrintTest> spCom;
    CComWeakPtr<IUnknown> wkcom;
};
/*
 *测试多线程的方法，线程安全测试用
 * void* testpthread(void* voidCom)
{
    CComSharedPtr<IPrintTest> spCom = *((CComSharedPtr<IPrintTest>*)voidCom);
    int i;

    usleep(1);
    for(i=0;i<5;i++)
    {
        CComSharedPtr<IPrintTest> spComtest;
        spComtest = spCom;
        spComtest->printref();
    }

    CComSharedPtr<IPrintTest> spComtest2[5];
    for(i=0;i<5;i++)
    {
        spComtest2[i] = spCom;
        spComtest2[i]->printref();
    }
    return 0;
}
*/
int main()
{
    SharedTest mysharedtest;
    {

        CComSharedPtr<IPrintTest> spIPT;
        HRESULT hr = CreateComTestInstance(IID_IPrintTest,(void**)&spIPT);
        spIPT.init_com_count();
        if (SUCCEEDED(hr))
        {
        }
        mysharedtest.spCom = spIPT;
        mysharedtest.wkcom = spIPT;
        {
            CComSharedPtr<IPrintTest> spIPTfrom_wk(mysharedtest.wkcom.lock(),boost::detail::dynamic_cast_tag());
        }
    }
/* 测试多线程的线程安全
 * pthread_t pid[10];

    for(int i=0;i<10;i++)
    {
        pthread_create(&pid[i],NULL,testpthread,(void*)&mysharedtest.spCom);
    }
    usleep(1000);
    for(int i=0;i<10;i++)
    {
        pthread_join(pid[i],NULL);
    }
    */
    mysharedtest.spCom.reset();
    CComSharedPtr<IPrintTest> spIPTfrom_wk2(mysharedtest.wkcom.lock(),boost::detail::dynamic_cast_tag());
    if (spIPTfrom_wk2)
    {
    }
    else
    {

        printf("get strong ref failed. The ref has been deleted.\n");
        mysharedtest.wkcom.reset();
    }
    return 0;
}
```