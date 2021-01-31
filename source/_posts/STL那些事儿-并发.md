---
title: STL那些事儿-并发
date: 2020-02-29 11:28:19
tags:
    - "C++"
    - "并发"
cover: "/img/5e568d5c56a51.jpg"
---
# 关于STL的并发部分

C++里我们所能依赖的并发工具，大概就是这样几个部分
1. `std::thread` 
2. `std::mutex`
3. `std::future`
4. `std::async`/`std::promise`
rust所提供的编译器线程安全检查有多爽，C++里自己管理锁就有多麻烦。那就先从C++里提供的`std::mutex`开始看。

## mutex

mutex类的实现很短

```c++
class mutex
	: public _Mutex_base
	{	// class for mutual exclusion
public:
	/* constexpr */ mutex() _NOEXCEPT	// TRANSITION
		: _Mutex_base()
		{	// default construct
		}

	mutex(const mutex&) = delete;
	mutex& operator=(const mutex&) = delete;
	};
```

禁用了`copy constructor`与`copy assigning operator`，很合理，锁本身就是不可复制的。除此之外没有别的东西，大部分特性都是由`_Mutex_base`派生得到，那么接着看`_Mutex_base`

```c++
class _Mutex_base
	{	// base class for all mutex types
public:
	_Mutex_base(int _Flags = 0) _NOEXCEPT
		{	// construct with _Flags
		_Mtx_init_in_situ(_Mymtx(), _Flags | _Mtx_try);
		}

	~_Mutex_base() _NOEXCEPT
		{	// clean up
		_Mtx_destroy_in_situ(_Mymtx());
		}

	_Mutex_base(const _Mutex_base&) = delete;
	_Mutex_base& operator=(const _Mutex_base&) = delete;

	void lock()
		{	// lock the mutex
		_Mtx_lockX(_Mymtx());
		}

	_NODISCARD bool try_lock()
		{	// try to lock the mutex
		return (_Mtx_trylockX(_Mymtx()) == _Thrd_success);
		}

	void unlock()
		{	// unlock the mutex
		_Mtx_unlockX(_Mymtx());
		}

	typedef void *native_handle_type;

	_NODISCARD native_handle_type native_handle()
		{	// return Concurrency::critical_section * as void *
		return (_Mtx_getconcrtcs(_Mymtx()));
		}

private:
	friend condition_variable;
	friend condition_variable_any;

	aligned_storage_t<_Mtx_internal_imp_size,
		_Mtx_internal_imp_alignment> _Mtx_storage;

	_Mtx_t _Mymtx() _NOEXCEPT
		{	// get pointer to _Mtx_internal_imp_t inside _Mtx_storage
		return (reinterpret_cast<_Mtx_t>(&_Mtx_storage));
		}
	};
```

`_Mtx_storage`即是关键的互斥量，提供了lock，unlock，try_lock三个方法。`aligned_storage_t`比较关键，是个trait，经过一段不可直视的展开之后应该是这样一个东西

```c++
template<class _Ty,
	size_t _Len>
	union _Align_type
	{	// union with size _Len bytes and alignment of _Ty
	_Ty _Val;
	char _Pad[_Len];
	};
```

traits真心是个不能说好的设计，或者该说模板元编程就是一团乱麻。

其中`_Ty`是一个double,` _Len`的长度是80，被访问时cast成`struct _Mtx_internal_imp_t`类型，很遗憾，后者的定义一直没有翻到，估计是一个ABI接口。

同时除了无参的构造函数外，还有接受一个flag的构造函数，用以区分各种锁。

同样由`_Mtx_storage`派生出的还有另一种锁`recursive_mutex`，即递归锁，Java那边似乎习惯叫可重入锁？相比`mutex`仅有两处不同

1. 重写了`try_lock`方法，方法里面调用了基类的同名方法，不太明白为什么要这么实现
2. 使用`_Mtx_recursive`作为flag传给了基类的构造函数，如何实现递归特性有待研究，推测这得问操作系统

stl中的互斥锁就到这里，接下来是三个`identifier type`

```c++
struct adopt_lock_t
	{	// indicates adopt lock
	};

struct defer_lock_t
	{	// indicates defer lock
	};

struct try_to_lock_t
	{	// indicates try to lock
	};
_INLINE_VAR constexpr adopt_lock_t adopt_lock{};
_INLINE_VAR constexpr defer_lock_t defer_lock{};
_INLINE_VAR constexpr try_to_lock_t try_to_lock{};
```

**强类型牛逼！**以类型标识了三个构造参数，会在之后的几种锁中用到。

先说最常用也最简单的`lock_guard`

```c++
template<class _Mutex>
	class lock_guard
	{	// class with destructor that unlocks a mutex
		......
		......
	};	
```

唯一一个成员变量`_Mutex& _MyMutex`，不用多说什么，依赖RAII控制，构造即上锁，析构即释放。唯一可以说说的是第二个构造函数

```c++
lock_guard(_Mutex& _Mtx, adopt_lock_t)
		: _MyMutex(_Mtx)
		{	// construct but don't lock
		}
```

可以接管别的锁。因为是泛型，所以大概可以做出`lock_guard<lock_guard<lock_guard<lock_guard<....>>>>`这种邪恶东西来。

因为啥方法都无，所以设计上就不是用来做精细的粒度控制的，你应该拿它做的就是进函数锁上然后不去碰它。

相比之下，`unique_lock`就能提供更多功能。

```c++
		// CLASS TEMPLATE unique_lock
template<class _Mutex>
	class unique_lock
	{	// whizzy class with destructor that unlocks mutex
		......
        ......
	};
```

两个成员变量`_Mutex *_Pmtx` 和`bool _Owns`，前者保存互斥量的资源，后者标识互斥量的状态。

在保存互斥量的时候用了指针而非`lock_guard`中的引用，目的昭然。`unique_lock`相比`lock_guard`更像一个智能指针，如同`unique_ptr`一般，是能拥有所有权的对象。

```c++
unique_lock(_Mutex& _Mtx, adopt_lock_t)
		: _Pmtx(_STD addressof(_Mtx)), _Owns(true)
		{	// construct and assume already locked
		}

	unique_lock(_Mutex& _Mtx, defer_lock_t) _NOEXCEPT
		: _Pmtx(_STD addressof(_Mtx)), _Owns(false)
		{	// construct but don't lock
		}
	unique_lock(unique_lock&& _Other) _NOEXCEPT
		: _Pmtx(_Other._Pmtx), _Owns(_Other._Owns)
		{	// destructive copy
		_Other._Pmtx = 0;
		_Other._Owns = false;
		}
```

这三个构造函数就足够表达`unique_lock`的核心特点

1. 真正接管一个互斥量
2. 获取互斥量的所有权

以及接下来的若干函数

```c++
	unique_lock(_Mutex& _Mtx, try_to_lock_t)
		: _Pmtx(_STD addressof(_Mtx)), _Owns(_Pmtx->try_lock())
		{	// construct and try to lock
		}

	template<class _Rep,
		class _Period>
		unique_lock(_Mutex& _Mtx,
			const chrono::duration<_Rep, _Period>& _Rel_time)
		: _Pmtx(_STD addressof(_Mtx)), _Owns(_Pmtx->try_lock_for(_Rel_time))
		{	// construct and lock with timeout
		}

	template<class _Clock,
		class _Duration>
		unique_lock(_Mutex& _Mtx,
			const chrono::time_point<_Clock, _Duration>& _Abs_time)
		: _Pmtx(_STD addressof(_Mtx)), _Owns(_Pmtx->try_lock_until(_Abs_time))
		{	// construct and lock with timeout
		}

	unique_lock(_Mutex& _Mtx, const xtime *_Abs_time)
		: _Pmtx(_STD addressof(_Mtx)), _Owns(false)
		{	// try to lock until _Abs_time
		_Owns = _Pmtx->try_lock_until(_Abs_time);
		}
```

延时上锁以及疯狂尝试，同时提供的lock与unlock方法，都说明这是一个用来实现精细粒度控制的锁。

`scoped_lock` WIP

`timed_mutex`WIP

`recursive_timed_mutex` WIP

### condition_variable

神奇的条件变量

没有多少方法，最基本的方法长这样

```c++
	void wait(unique_lock<mutex>& _Lck)
		{	// wait for signal
		// Nothing to do to comply with LWG 2135 because std::mutex lock/unlock are nothrow
		_Cnd_waitX(_Mycnd(), _Lck.mutex()->_Mymtx());
		}
	void notify_one() _NOEXCEPT
		{	// wake up one waiter
		_Cnd_signalX(_Mycnd());
		}
```

基本上就是这么个用法吧，就跟量子纠缠一样，一边notify就会把另一边的wait唤醒。

一般来讲会使用wait的另一个重载版本，额外传入一个闭包直到闭包返回true，否则往往会导致死锁的场面。

## thread

方法就主要那么几个，join和detach的区别也不必多说。实现倒是值得考据，thread有一个来自`<xthread>`的`_LaunchPad`类，同时`_LaunchPad`派生自`_Pad`，这是线程启动的核心。成员变量有三个，一个条件变量，一个mutex还有一个应该是记录线程是否已启动的bool。不止如此，这个类还与`<xthrcommon>`中的`_Thrd_imp_t`结构体有着千丝万缕的联系。

```c++
typedef unsigned int _Thrd_id_t;
typedef struct
	{	/* thread identifier for Win32 */
	void *_Hnd;	/* Win32 HANDLE */
	_Thrd_id_t _Id;
	} _Thrd_imp_t;

```

我们在这里抵达了C++的边界，再进一步便是混乱而邪恶的操作系统。理所当然的命题得到了证明，C++中的线程不过只是对操作系统api的包装。

如果我们再退一步，还能看到许多`_EXTERN_C`，这无时无刻不提醒着我，冯诺依曼电子计算机的扩张是一场多么铁血冷酷而无情的战争。我在这里虔诚的呼唤至高神圣的主**λ**回归世间，洗净所有C/C++程序员的罪孽，将世界引回正途。

## 异步

WIP