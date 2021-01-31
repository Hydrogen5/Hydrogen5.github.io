---
title: Web框架新秀Drogon源码分析
date: 2020-05-29 22:50:41
cover: /img/IMG_20200304_144858_229.jpg
tags: 
  - "C++"
  - "Web"
hide: true
---
近日发现[节奏榜](http://www.techempower.com/benchmarks/)更新了，[`actix`](https://github.com/actix/actix)痛失榜一，而罪魁祸首正是标题中的[`drogon`](https://github.com/an-tao/drogon)。  
这是个名不见经传的框架，但是是**C++**写的，作者还是个国人，这不由让我产生了几分兴趣。  
总之先占个坑。

# 从example开始看起

首先是个最小实践

```cpp
#include <drogon/drogon.h>
using namespace drogon;
int main()
{
    app().setLogPath("./")
         .setLogLevel(trantor::Logger::kWarn)
         .addListener("0.0.0.0", 80)
         .setThreadNum(16)
         .enableRunAsDaemon()
         .run();
}
```

看起来非常中规中矩。

```cpp
app.registerHandler("/test?username={name}",
                    [](const HttpRequestPtr& req,
                       std::function<void (const HttpResponsePtr &)> &&callback,
                       const std::string &name)
                    {
                        Json::Value json;
                        json["result"]="ok";
                        json["message"]=std::string("hello,")+name;
                        auto resp=HttpResponse::newHttpJsonResponse(json);
                        callback(resp);
                    },
                    {Get,"LoginFilter"});
```

但是注册接口这里开始变得吓人起来。

同样也可以封装成控制器

```cpp
/// The TestCtrl.h file
#pragma once
#include <drogon/HttpSimpleController.h>
using namespace drogon;
class TestCtrl:public drogon::HttpSimpleController<TestCtrl>
{
public:
    virtual void asyncHandleHttpRequest(const HttpRequestPtr& req, std::function<void (const HttpResponsePtr &)> &&callback) override;
    PATH_LIST_BEGIN
    PATH_ADD("/test",Get);
    PATH_LIST_END
};

/// The TestCtrl.cc file
#include "TestCtrl.h"
void TestCtrl::asyncHandleHttpRequest(const HttpRequestPtr& req,
                                      std::function<void (const HttpResponsePtr &)> &&callback)
{
    //write your application logic here
    auto resp = HttpResponse::newHttpResponse();
    resp->setBody("<p>Hello, world!</p>");
    resp->setExpiredTime(0);
    callback(resp);
}
```

……也许PHP真的是最好的语言。

^ 但是`drogon`提供了一套工具自动生成代码中的大部分，看起来还不差。

注意到这只是`HttpSimpleController`，当然也有功能更多的`HttpController`，当然也需要写更多东西……

---

好吧一个web框架的介绍到这里就足够了，只是我们依然好奇这个屠榜的运行速度从何而来？

不妨再看看这个介绍

- 网络层使用基于epoll(macOS/FreeBSD下是kqueue)的非阻塞IO框架，提供高并发、高性能的网络IO。详细请见[性能测试](https://github.com/an-tao/drogon/wiki/13-性能测试)和[TFB Live Results](https://tfb-status.techempower.com/)；
- 全异步编程模式；
- 基于非阻塞IO实现的异步数据库读写，目前支持PostgreSQL和MySQL(MariaDB)数据库；

当然吸引我的还有一点

- 基于template实现了简单的反射机制，使主程序框架、控制器(controller)和视图(view)完全解耦；

那么接下来不妨就这几点看看性能是如何被压榨出来的。

# 反射剑法

`drogon`提供了非常简陋的反射机制，你可以通过类名获取到堆上的实例，也可以以单例模式共享实例。没了。

## 实现

一个哈希表走天下。

```cpp
void DrClassMap::registerClass(const std::string &className,
                               const DrAllocFunc &func)
{
    LOG_TRACE << "Register class:" << className;
    getMap().insert(std::make_pair(className, func));
}
DrObjectBase *DrClassMap::newObject(const std::string &className)
{
    auto iter = getMap().find(className);
    if (iter != getMap().end())
    {
        return iter->second();
    }
    else
        return nullptr;
}
```

提供了对应的注册函数以及使用函数。

对应的单例模式则提供了两个函数

```cpp
    template <typename T>
    static std::shared_ptr<T> getSingleInstance()
    {
        static_assert(std::is_base_of<DrObjectBase, T>::value,
                      "T must be a sub-class of DrObjectBase");
        static auto const singleton =
            std::dynamic_pointer_cast<T>(getSingleInstance(T::classTypeName()));
        assert(singleton);
        return singleton;
    }

const std::shared_ptr<DrObjectBase> &DrClassMap::getSingleInstance(
    const std::string &className)
{
    auto &mtx = internal::getMapMutex();
    auto &singleInstanceMap = internal::getObjsMap();
    {
        std::lock_guard<std::mutex> lock(mtx);
        auto iter = singleInstanceMap.find(className);
        if (iter != singleInstanceMap.end())
            return iter->second;
    }
    auto newObj = std::shared_ptr<DrObjectBase>(newObject(className));
    {
        std::lock_guard<std::mutex> lock(mtx);
        auto ret = singleInstanceMap.insert(
            std::make_pair(className, std::move(newObj)));
        return ret.first->second;
    }
}
```

为了提高效率锁的粒度还挺细的。

## 基类

对应的`DrObject`实现同样简单，`DrObjct`是一个泛型类，自动为对象实现`className`,`isClass`方法，并注册对应的反射。

```cpp
        const std::string &className() const
        {
            static std::string className =
                DrClassMap::demangle(typeid(T).name());
            return className;
        }
        template <typename D>
        typename std::enable_if<std::is_default_constructible<D>::value,
                                void>::type
        registerClass()
        {
            DrClassMap::registerClass(className(),
                                      []() -> DrObjectBase * { return new T; });
        }
        template <typename D>
        typename std::enable_if<!std::is_default_constructible<D>::value,
                                void>::type
        registerClass()
        {
            LOG_ERROR << "Can't register classes without a default constructor";
        }
```

在cpp里做类型计算真是心累的事。