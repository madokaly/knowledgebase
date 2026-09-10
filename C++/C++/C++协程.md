C++20 的协程（Coroutine）最容易让人困惑的一点是：

**它不是线程，也不是一种新的函数调用方式，而是“可以暂停并在之后继续执行的函数”。**

普通函数：

```
void Func()  
{  
    Step1();  
    Step2();  
    Step3();  
}
```

一旦进入 `Func()`，通常会一直执行到 `return`。

而协程可以变成：

```
Task Func()  
{  
    Step1();  
​  
    co_await Something();  
​  
    Step2();  
​  
    co_await SomethingElse();  
​  
    Step3();  
}
```

执行到 `co_await` 时，它可以：

```
Func()  
  ↓  
Step1  
  ↓  
暂停  
  ↓  
其他代码继续运行  
  ↓  
某个时刻恢复 Func()  
  ↓  
Step2  
  ↓  
再次暂停  
  ↓  
恢复  
  ↓  
Step3
```

这就是理解 C++20 协程最核心的一点。

---

# 1. C++20 协程解决什么问题

举个游戏开发里很典型的逻辑。

假设你想写：

播放攻击动画();  
​  
等待动画播放结束();  
​  
生成伤害判定();  
​  
等待0.3秒();  
​  
进入下一段攻击();

如果是普通 C++，你不能真的这么写：

```
void Attack()  
{  
    PlayAnimation();  
​  
    WaitAnimation();  
​  
    CreateHitBox();  
​  
    Wait(0.3f);  
​  
    NextAttack();  
}
```

因为 `WaitAnimation()` 如果阻塞线程，整个 GameThread 都停了。

所以传统游戏开发通常会拆成状态机：

```
enum class AttackState  
{  
    Start,  
    WaitingAnimation,  
    Hit,  
    Recovery,  
    Finish  
};
```

然后：

```
void Tick(float DeltaTime)  
{  
    switch (State)  
    {  
    case AttackState::Start:  
        PlayAnimation();  
        State = AttackState::WaitingAnimation;  
        break;  
​  
    case AttackState::WaitingAnimation:  
        if (AnimationFinished())  
        {  
            CreateHitBox();  
            State = AttackState::Recovery;  
        }  
        break;  
​  
    case AttackState::Recovery:  
        ...  
        break;  
    }  
}
```

问题是：

**代码的逻辑顺序被状态机打碎了。**

协程允许重新写回：

```
Task Attack()  
{  
    PlayAnimation();  
​  
    co_await WaitAnimation();  
​  
    CreateHitBox();  
​  
    co_await WaitSeconds(0.3f);  
​  
    co_await NextAttack();  
}
```

看起来几乎就是：

播放动画  
↓  
等动画结束  
↓  
打伤害  
↓  
等0.3秒  
↓  
下一段攻击

但是：

**等待的时候不会阻塞线程。**

这也是协程最大的价值。

---

# 2. 什么函数会变成协程

只要函数中出现下面任意一个关键字：

co_await  
co_yield  
co_return

编译器就会把它转换成协程。

例如：

Task Foo()  
{  
    co_return;  
}

虽然只有一句：

co_return;

但它已经不是普通函数了，而是协程。

三个关键字可以简单记成：

|关键字|含义|
|---|---|
|`co_await`|等待某件事情，并允许协程暂停|
|`co_yield`|产出一个值，然后暂停|
|`co_return`|结束协程|

其中最重要的是：

co_await

---

# 3. 普通函数和协程最根本的区别

普通函数调用：

A();

栈大概是：

main  
 ↓  
A  
 ↓  
B  
 ↓  
C

函数返回以后，它的局部变量通常也就没了。

例如：

void Foo()  
{  
    int x = 10;  
}

`Foo()` 返回后：

x

自然就不存在了。

但是协程有一个特殊需求：

Task Foo()  
{  
    int x = 10;  

    co_await Something();  
  
    std::cout << x;  

}

这里协程暂停以后：

Foo()

已经不再占用普通调用栈。

但是以后恢复的时候：

x

还必须是 `10`。

所以编译器必须专门保存：

x  
当前执行到了哪里  
promise 对象  
协程状态  
其他跨暂停点存活的局部变量

这些东西被保存到一个对象里。

这个对象叫：

# Coroutine Frame

协程帧。

---

# 4. Coroutine Frame 是理解协程的核心

例如：

Task Foo()  
{  
    int a = 10;  
    std::string name = "Alice";  

    co_await WaitSomething();  
  
    std::cout << a << name;  
  
    co_return;  

}

编译器在概念上可能生成：

Coroutine Frame  
┌──────────────────────┐  
│ promise │  
│ │  
│ a = 10 │  
│ name = "Alice" │  
│ │  
│ 当前执行位置 = await │  
│ │  
│ 其他协程内部状态 │  
└──────────────────────┘

因此暂停以后：

Foo()

的调用栈可以退出。

但是 Frame 仍然存在。

之后：

resume()

重新从暂停位置继续。

所以：

> 协程暂停 ≠ 保存线程栈。

更准确地说是：

> 编译器把需要跨越暂停点存活的状态保存到了 Coroutine Frame。

这点非常重要。

---

# 5. 协程其实是编译器生成的状态机

比如：

Task Foo()  
{  
    StepA();  

    co_await X;  
  
    StepB();  
  
    co_await Y;  
  
    StepC();  

}

你可以粗略理解为编译器改造成：

struct FooCoroutine  
{  
    int State = 0;  

    void Resume()  
    {  
        switch (State)  
        {  
        case 0:  
  
            StepA();  
  
            State = 1;  
  
            if (!X.ready())  
            {  
                X.suspend();  
                return;  
            }  
  
        case 1:  
  
            StepB();  
  
            State = 2;  
  
            if (!Y.ready())  
            {  
                Y.suspend();  
                return;  
            }  
  
        case 2:  
  
            StepC();  
  
            State = 3;  
            return;  
        }  
    }  

};

真实生成代码当然复杂很多。

但这个模型非常接近本质：

协程  
≈  
编译器自动帮你写状态机

因此如果你以前在 UE / Unity 中写过：

Idle  
AttackStartup  
AttackActive  
AttackRecovery

这种状态机，那么理解协程其实会很容易。

协程做的事情，本质上就是：

> 让你用顺序代码描述状态机。

---

# 6. `co_await` 到底做了什么

这是 C++20 协程最关键的部分。

假设：

co_await object;

这里的 `object` 最终会得到一个 Awaiter。

Awaiter 需要提供三个核心函数：

await_ready()  
await_suspend()  
await_resume()

可以记成：

co_await X  

     ↓  

X.await_ready()  

     ↓  

X.await_suspend()  

     ↓  

协程暂停  

     ↓  

以后 resume()  

     ↓  

X.await_resume()  

     ↓  

继续执行

三个函数分别负责：

---

# 7. `await_ready()`

定义：

bool await_ready();

意思：

> 这件事情现在已经完成了吗？

例如：

bool await_ready()  
{  
    return Finished;  
}

如果返回：

true

那么：

co_await

不会暂停。

直接继续执行。

例如：

co_await LoadTexture();

如果纹理已经在缓存里：

await_ready() = true

那么完全没必要暂停协程。

直接：

继续往下执行

如果：

await_ready() = false

则准备进入暂停流程。

---

# 8. `await_suspend()`

典型形式：

void await_suspend(std::coroutine_handle<> handle);

它负责：

> 协程即将暂停，我应该怎么安排它以后恢复？

这里会收到一个非常重要的对象：

std::coroutine_handle<>

它可以理解成：

> 当前协程的句柄。

通过它可以：

handle.resume();

恢复协程。

例如：

struct TimerAwaiter  
{  
    bool await_ready()  
    {  
        return false;  
    }  

    void await_suspend(std::coroutine_handle<> h)  
    {  
        TimerManager.AddTimer(  
            1.0f,  
            [h]()  
            {  
                h.resume();  
            });  
    }  
  
    void await_resume()  
    {  
    }  

};

然后：

Task Foo()  
{  
    std::cout << "A";  

    co_await TimerAwaiter{};  
  
    std::cout << "B";  

}

执行过程：

打印 A  

↓  

TimerAwaiter::await_ready()  
false  

↓  

await_suspend(handle)  

↓  

把 handle 保存到 TimerManager  

↓  

Foo 暂停  

↓  

1 秒以后 TimerManager 回调  

↓  

handle.resume()  

↓  

await_resume()  

↓  

打印 B

所以你会发现：

**C++ 协程本身甚至不知道 Timer 是什么。**

它只是给了你一个：

coroutine_handle

至于什么时候恢复：

Timer  
网络  
IO  
线程池  
动画事件  
UE Delegate  
TaskGraph  
GPU Fence  
文件加载

都可以由你决定。

---

# 9. `await_resume()`

它在协程恢复时调用。

并且它可以返回一个值。

例如：

struct NetworkAwaiter  
{  
    Response response;  

    bool await_ready()  
    {  
        return false;  
    }  
  
    void await_suspend(std::coroutine_handle<> handle)  
    {  
        RequestAsync(  
            [this, handle](Response result)  
            {  
                response = result;  
                handle.resume();  
            });  
    }  
  
    Response await_resume()  
    {  
        return response;  
    }  

};

于是你可以写：

Response result = co_await NetworkRequest();

整个逻辑实际上是：

NetworkRequest  
↓  
暂停协程  
↓  
网络请求完成  
↓  
resume  
↓  
await_resume()  
↓  
返回 Response  
↓  
赋值给 result

因此：

auto result = co_await X;

最终赋值给 `result` 的：

不是 `X` 本身。

而是：

awaiter.await_resume()

的返回值。

这是很重要的知识点。

---

# 10. 一个完整 Awaiter

最简单的：

#include <coroutine>  
#include <iostream>  

struct MyAwaiter  
{  
    bool await_ready() const noexcept  
    {  
        std::cout << "await_ready\n";  
        return false;  
    }  

    void await_suspend(std::coroutine_handle<> handle) const noexcept  
    {  
        std::cout << "await_suspend\n";  
  
        handle.resume();  
    }  
  
    int await_resume() const noexcept  
    {  
        std::cout << "await_resume\n";  
        return 100;  
    }  

};

使用：

Task Foo()  
{  
    int value = co_await MyAwaiter{};  

    std::cout << value << '\n';  

}

大致输出：

await_ready  
await_suspend  
await_resume  
100

注意：

await_suspend()

里面立即：

handle.resume();

只是为了演示。

实际异步系统通常是：

void await_suspend(std::coroutine_handle<> handle)  
{  
    SaveHandle(handle);  
}

然后某个未来事件：

handle.resume();

---

# 11. `std::coroutine_handle` 是什么

可以粗略理解为：

Coroutine Frame 的遥控器

例如：

std::coroutine_handle<> handle;

可以做：

handle.resume();

恢复：

暂停中的协程

还可以：

handle.done();

检查是否结束。

以及：

handle.destroy();

销毁协程帧。

概念上：

Coroutine Frame  
        ↑  
        │  
coroutine_handle

它不是协程的所有权模型本身。

更像：

一个指向协程状态的轻量句柄

因此生命周期管理尤其重要。

---

# 12. 那 `Task` 又是什么

可能你已经发现一个问题：

Task Foo()  
{  
    co_await X;  
}

为什么返回类型一定是什么 `Task`？

C++ 并没有内置：

Task

这一点和 C# 很不一样。

C# 有：

async Task Foo()

但 C++20 只提供了**协程语言机制**。

它没有给你：

Task  
Generator  
Scheduler  
TimerAwaiter  
IOAwaiter

这些高级组件。

所以：

Task

通常需要：

库  
框架  
或者你自己

实现。

这也是 C++20 协程第一次接触时显得很底层的原因。

---

# 13. `promise_type` 是整个 C++ 协程的中枢

假设：

Task Foo()  
{  
    co_return;  
}

编译器会查看：

Task::promise_type

例如：

struct Task  
{  
    struct promise_type  
    {  
    };  
};

为什么叫：

promise_type

这里不要和：

std::promise

混为一谈。

它们不是一回事。

你可以把 coroutine `promise_type` 理解为：

> 协程和外部世界之间的控制中心。

---

# 14. `promise_type` 典型结构

一个最小化版本大概长这样：

struct Task  
{  
    struct promise_type  
    {  
        Task get_return_object()  
        {  
            return {};  
        }  

        std::suspend_never initial_suspend() noexcept  
        {  
            return {};  
        }  
  
        std::suspend_never final_suspend() noexcept  
        {  
            return {};  
        }  
  
        void return_void()  
        {  
        }  
  
        void unhandled_exception()  
        {  
            std::terminate();  
        }  
    };  

};

这些函数分别负责不同阶段。

生命周期大致：

调用协程  
    ↓  
创建 Coroutine Frame  
    ↓  
创建 promise_type  
    ↓  
get_return_object()  
    ↓  
initial_suspend()  
    ↓  
执行协程正文  
    ↓  
co_return  
    ↓  
return_void / return_value  
    ↓  
final_suspend()  
    ↓  
销毁 Coroutine Frame

这是另一个需要牢牢记住的流程。

---

# 15. `get_return_object()`

比如：

Task Foo()  
{  
    ...  
}

调用：

auto task = Foo();

那么这里：

task

是怎么构造出来的？

答案是：

promise.get_return_object()

例如：

Task get_return_object()  
{  
    return Task{  
        std::coroutine_handle<promise_type>::from_promise(*this)  
    };  
}

它通常会：

1. 获取当前 Coroutine Frame 的 handle
    
2. 封装到 `Task`
    
3. 返回给调用方

所以：

Task

通常就是：

Coroutine Handle 的 RAII 包装器

加上一些额外状态。

---

# 16. `initial_suspend()`

这个函数决定：

> 调用协程以后，是立刻执行协程正文，还是先暂停？

有两个标准 Awaiter 经常用到：

std::suspend_never

以及：

std::suspend_always

---

## `std::suspend_never`

例如：

std::suspend_never initial_suspend()  
{  
    return {};  
}

意思：

不要暂停

所以：

Task Foo()  
{  
    std::cout << "Foo\n";  
    co_return;  
}  

Foo();

调用 `Foo()` 后立即进入正文。

这种叫：

eager coroutine

也就是：

**立即启动型协程。**

---

## `std::suspend_always`

std::suspend_always initial_suspend()  
{  
    return {};  
}

意思：

一创建就暂停

于是：

auto task = Foo();

只是创建：

Coroutine Frame  
Task

但正文还没运行。

必须：

task.Resume();

才开始。

这种叫：

lazy coroutine

也就是：

**惰性协程。**

很多 Task / Generator 都采用这种设计。

---

# 17. `co_return`

例如：

Task Foo()  
{  
    co_return;  
}

编译器会调用：

promise.return_void();

如果是：

Task<int> Foo()  
{  
    co_return 100;  
}

那么会调用：

promise.return_value(100);

例如：

void return_value(int value)  
{  
    result = value;  
}

之后调用方可能：

int x = co_await Foo();

最后：

100

就通过 Task/Awaiter 传出来。

---

# 18. `final_suspend()`

这个特别重要。

协程正文执行完以后：

co_return;

它不会简单地像普通函数一样自动消失。

它首先进入：

final_suspend()

例如：

std::suspend_always final_suspend() noexcept  
{  
    return {};  
}

意思：

正文已经结束  
但是 Coroutine Frame 先别销毁

为什么？

因为：

Task

可能还需要读取：

返回值  
异常  
状态

或者：

恢复正在等待这个 Task 的父协程

所以实际 Task 类型中：

final_suspend()

通常是一个非常关键的地方。

---

# 19. 父协程等待子协程

这是 Task 模型里最重要的事情之一。

比如：

Task Child()  
{  
    co_await WaitSeconds(1);  

    co_return;  

}

父任务：

Task Parent()  
{  
    StepA();  

    co_await Child();  
  
    StepB();  

}

预期流程是：

Parent  
 ↓  
StepA  
 ↓  
启动 Child  
 ↓  
Parent 暂停  

Child  
 ↓  
WaitSeconds  
 ↓  
暂停  

1秒后  
 ↓  
Child 恢复  
 ↓  
Child 完成  
 ↓  
恢复 Parent  

Parent  
 ↓  
StepB

所以：

co_await Child()

实际需要完成一个重要操作：

> Child 必须记住是谁在等待它。

通常叫：

continuation

---

# 20. continuation

例如 `promise_type` 中：

std::coroutine_handle<> continuation;

Parent：

co_await Child();

时，可以在 Child 的 awaiter 中：

child.promise().continuation = parentHandle;

于是：

Child  
┌─────────────────────┐  
│ continuation │────→ Parent  
└─────────────────────┘

Child 最后执行：

final_suspend()

时就可以：

return continuation;

把执行权还给 Parent。

这就是：

co_await Task

背后的基本原理。

---

# 21. 协程之间并不是“自动异步”

这是 C++ 协程最容易产生的误解之一。

写：

co_await Foo();

并不意味着：

Foo 在另一个线程执行

也不意味着：

自动加入线程池

甚至：

co_await

本身都不等于异步。

例如：

struct ReadyAwaiter  
{  
    bool await_ready()  
    {  
        return true;  
    }  

    void await_suspend(std::coroutine_handle<>)  
    {  
    }  
  
    int await_resume()  
    {  
        return 10;  
    }  

};

这里：

int result = co_await ReadyAwaiter{};

完全不会暂停。

本质就是：

int result = 10;

因此牢记：

> `co_await` 只是提供“暂停点”。

谁来恢复、在哪里恢复、什么时候恢复，完全由 Awaiter / Scheduler 决定。

---

# 22. 协程和线程是什么关系

例如：

Task Foo()  
{  
    co_await SomeAsyncOperation();  
}

可能全程都在主线程：

GameThread  

Foo  
↓  
暂停  
↓  
GameThread继续Tick  
↓  
下一帧  
↓  
恢复Foo

根本没有新线程。

也可以：

GameThread  
↓  
Foo  
↓  
暂停  
↓  
ThreadPool执行IO  
↓  
IO完成  
↓  
GameThread恢复Foo

甚至：

GameThread  
↓  
Foo  
↓  
暂停  
↓  
WorkerThread恢复Foo

这些都是可能的。

所以：

Coroutine ≠ Thread

更准确地说：

Thread  
= 代码在哪执行  

Coroutine  
= 代码如何暂停和继续

这是两个维度。

---

# 23. 游戏开发中可以把协程理解成“高级 Latent Action”

如果结合你熟悉的 Unreal 来理解，会非常直观。

UE 蓝图里：

Event  
 ↓  
Play Animation  
 ↓  
Delay 1s  
 ↓  
Spawn Actor

`Delay` 不会阻塞 GameThread。

蓝图内部会把：

执行状态  
Continuation  
Latent Action

保存起来。

然后一秒后继续。

C++20 Coroutine 本质上允许 C++ 写成类似：

Task Attack()  
{  
    PlayAnimation();  

    co_await Delay(1s);  
  
    SpawnActor();  

}

可以把它理解成：

蓝图 Latent Action  
        +  
编译器生成状态机  
        +  
C++ 类型系统

这一点对 UE 开发者特别好理解。

---

# 24. `co_yield`

`co_yield` 主要用于 Generator。

例如你想生成：

1  
2  
3

传统：

std::vector<int> GetNumbers()  
{  
    return {1, 2, 3};  
}

但如果数据量很大：

1000万个

就没有必要一次性全部构造。

可以：

Generator<int> Numbers()  
{  
    co_yield 1;  
    co_yield 2;  
    co_yield 3;  
}

调用方：

for (int value : Numbers())  
{  
    std::cout << value << '\n';  
}

执行：

第一次 next  
↓  
执行到 co_yield 1  
↓  
返回 1  
↓  
暂停  

第二次 next  
↓  
从上次继续  
↓  
co_yield 2  
↓  
返回 2  
↓  
暂停

本质仍然是：

暂停 + 恢复

---

# 25. `co_yield` 实际是什么

编译器大概会把：

co_yield value;

转换为类似：

co_await promise.yield_value(value);

所以 `promise_type` 可以写：

std::suspend_always yield_value(int value)  
{  
    current_value = value;  
    return {};  
}

即：

保存 value  
↓  
暂停协程

调用方读取：

current_value

然后：

resume()

获取下一个值。

---

# 26. 一个简化版 Generator

下面这个例子很值得理解：

#include <coroutine>  
#include <iostream>  

class Generator  
{  
public:  

    struct promise_type  
    {  
        int current_value;  
  
        Generator get_return_object()  
        {  
            return Generator{  
                std::coroutine_handle<promise_type>::from_promise(*this)  
            };  
        }  
  
        std::suspend_always initial_suspend()  
        {  
            return {};  
        }  
  
        std::suspend_always final_suspend() noexcept  
        {  
            return {};  
        }  
  
        std::suspend_always yield_value(int value)  
        {  
            current_value = value;  
            return {};  
        }  
  
        void return_void()  
        {  
        }  
  
        void unhandled_exception()  
        {  
            std::terminate();  
        }  
    };  

private:  

    std::coroutine_handle<promise_type> handle;  

public:  

    explicit Generator(std::coroutine_handle<promise_type> h)  
        : handle(h)  
    {  
    }  
  
    ~Generator()  
    {  
        if (handle)  
        {  
            handle.destroy();  
        }  
    }  
  
    bool Next()  
    {  
        handle.resume();  
  
        return !handle.done();  
    }  
  
    int Value() const  
    {  
        return handle.promise().current_value;  
    }  

};

然后：

Generator Numbers()  
{  
    co_yield 10;  
    co_yield 20;  
    co_yield 30;  
}

调用：

auto numbers = Numbers();  

while (numbers.Next())  
{  
    std::cout << numbers.Value() << '\n';  
}

结果：

10  
20  
30

这里你已经能看到完整关系：

Generator  
  │  
  │ handle  
  ▼  
Coroutine Frame  
  │  
  ├── promise_type  
  │ └── current_value  
  │  
  └── 执行状态

---

# 27. `promise_type` 和协程返回类型的关系

编译器怎么知道：

Task Foo()

应该用哪个 `promise_type`？

通常会查：

std::coroutine_traits<Task>::promise_type

最常见情况下：

Task::promise_type

所以：

struct Task  
{  
    struct promise_type  
    {  
        ...  
    };  
};

是一种标准写法。

---

# 28. 协程调用流程完整展开

假设：

Task Foo()  
{  
    A();  

    co_await X;  
  
    B();  
  
    co_return;  

}

调用：

auto task = Foo();

概念上会经历：

① 分配 Coroutine Frame  

② 在 Frame 中构造 promise_type  

③ 保存函数参数  

④ promise.get_return_object()  

⑤ promise.initial_suspend()  

⑥ 如果不暂停：  
       开始执行 Foo  

⑦ A()  

⑧ 处理 co_await X  

       ↓  

   获取 Awaiter  

       ↓  

   await_ready()  

       ↓ false  

   await_suspend(handle)  

       ↓  

   Foo 暂停  

⑨ 某事件发生  

   handle.resume()  

⑩ await_resume()  

⑪ B()  

⑫ co_return  

⑬ promise.return_void()  

⑭ promise.final_suspend()  

⑮ 最终 Coroutine Frame 被 destroy

如果你把这个流程理解了，C++20 Coroutine 大概已经理解了一半以上。

---

# 29. Coroutine Frame 通常在哪里

很多情况下是在：

堆

上分配。

类似：

operator new

但标准并没有简单规定：

> 所有 Coroutine Frame 必须在 heap。

编译器在满足条件时可以优化掉动态分配。

例如：

Task Child()  
{  
    ...  
}  

Task Parent()  
{  
    co_await Child();  
}

如果生命周期非常明确，编译器可能做：

allocation elision

把协程帧嵌入其他存储中。

不过在性能分析中，一般应该意识到：

创建协程  
可能产生一次动态内存分配

对于游戏开发尤其重要。

比如你每帧创建：

100000 个 coroutine

就很可能是个问题。

---

# 30. 局部变量什么时候进入 Coroutine Frame

不一定所有变量都会保存。

例如：

Task Foo()  
{  
    int a = 10;  

    Use(a);  
  
    co_await X;  
  
    int b = 20;  
  
    Use(b);  

}

`a` 如果暂停之后不再使用：

a

未必需要保存到 Frame。

而：

Task Foo()  
{  
    int a = 10;  

    co_await X;  
  
    Use(a);  

}

`a` 必须跨越 suspend point：

co_await X

所以通常必须存入 Coroutine Frame。

可以粗略记成：

生命周期跨过 co_await 的局部状态  
→ 通常需要保存在 Coroutine Frame

这会影响：

Frame大小  
缓存局部性  
性能

---

# 31. 一个很重要的危险：引用生命周期

例如：

Task Foo(const std::string& name)  
{  
    co_await Delay();  

    std::cout << name;  

}

调用：

Foo(std::string("Alice"));

这可能非常危险。

因为：

std::string("Alice")

这个临时对象可能早就销毁了。

但是协程中：

name

只是保存的引用。

恢复时可能变成：

悬空引用  
dangling reference

所以协程中特别要小心：

T&  
const T&  
T*  
this

跨越：

co_await

以后，它们指向的对象是否仍然存在。

游戏项目中最典型的是：

Task Character::Attack()  
{  
    co_await Delay(3s);  

    HP -= 10;  

}

问题：

3 秒以后 Character 还存在吗？

如果 Character 已经被销毁：

this

就悬空了。

在 UE 中尤其要考虑：

UObject GC  
Actor Destroy  
TWeakObjectPtr  
Object lifetime

所以：

协程生命周期

和：

对象生命周期

是实际工程中的核心问题之一。

---

# 32. 异常如何处理

普通协程中，如果内部：

throw

并且异常没有被捕获：

Task Foo()  
{  
    throw std::runtime_error("error");  

    co_return;  

}

编译器不会简单地让它直接穿出协程。

而是调用：

promise.unhandled_exception();

例如：

void unhandled_exception()  
{  
    exception = std::current_exception();  
}

然后父协程：

co_await Foo();

时：

await_resume()

可以：

std::rethrow_exception(exception);

所以 Task 通常会存：

std::exception_ptr

大致：

Child throw  
↓  
unhandled_exception()  
↓  
保存 exception_ptr  
↓  
Child结束  
↓  
Parent恢复  
↓  
await_resume()  
↓  
rethrow_exception

---

# 33. `std::suspend_always` 到底是什么

实际上它只是一个标准 Awaiter：

struct suspend_always  
{  
    constexpr bool await_ready() const noexcept  
    {  
        return false;  
    }  

    constexpr void await_suspend(  
        coroutine_handle<>  
    ) const noexcept  
    {  
    }  
  
    constexpr void await_resume() const noexcept  
    {  
    }  

};

因为：

await_ready() == false

所以：

永远暂停

---

# 34. `std::suspend_never`

大概是：

struct suspend_never  
{  
    constexpr bool await_ready() const noexcept  
    {  
        return true;  
    }  

    constexpr void await_suspend(  
        coroutine_handle<>  
    ) const noexcept  
    {  
    }  
  
    constexpr void await_resume() const noexcept  
    {  
    }  

};

因为：

await_ready() == true

所以：

永远不暂停

理解了这两个，你就会发现：

initial_suspend()  
final_suspend()  
co_await

其实全都建立在同一个 Awaiter 协议上。

---

# 35. `co_await` 并不一定直接使用对象自身

下面这句：

co_await expr;

实际上还有一层自定义机制。

通常可能经过：

promise.await_transform(expr)

以及：

operator co_await()

最终才得到 Awaiter。

大概理解成：

co_await expr  

↓  

promise.await_transform(expr)  
        可选  

↓  

expr.operator co_await()  
        或  
operator co_await(expr)  
        可选  

↓  

awaiter  

↓  

await_ready()  
await_suspend()  
await_resume()

新手阶段最重要的是先理解最后三个。

后面的：

await_transform  
operator co_await

属于自定义框架时的高级机制。

---

# 36. `await_suspend()` 的返回类型也有讲究

常见三种。

## 返回 `void`

void await_suspend(std::coroutine_handle<> h);

意思基本是：

当前协程已经暂停

---

返回 bool

bool await_suspend(std::coroutine_handle<> h);

返回：

true

表示：

确实暂停

返回：

false

表示：

不要暂停，继续执行当前协程

---

返回另一个 `coroutine_handle`

std::coroutine_handle<>  
await_suspend(std::coroutine_handle<> h);

表示：

> 当前协程暂停以后，直接继续执行另一个协程。

这叫：

symmetric transfer

在 Task 链中非常重要。

例如：

Parent  
 ↓  
await Child  

Parent暂停  
 ↓  
直接转移给Child

而不是：

Parent  
 ↓  
Scheduler  
 ↓  
Child

减少额外调度。

---

# 37. Symmetric Transfer

假设：

Task A()  
{  
    co_await B();  
}

如果 B 的 Awaiter：

std::coroutine_handle<>  
await_suspend(std::coroutine_handle<> parent)  
{  
    child.promise().continuation = parent;  

    return child;  

}

那么执行权：

A  
↓  
B

可以直接切过去。

B 执行完成：

final_suspend()

再：

B  
↓  
A

直接切回来。

概念上：

A Coroutine Frame  
        ↓  
B Coroutine Frame  
        ↓  
C Coroutine Frame

但并不是：

A函数栈  
 ↓  
B函数栈  
 ↓  
C函数栈

这也是协程和递归普通调用的重要区别。

---

# 38. 为什么协程适合大量异步任务

传统线程模型：

每个任务一个线程

会面临：

线程栈内存  
Context Switch  
线程数量限制  
调度成本

而协程：

一个线程  
   ↓  
Task A  
   ↓ suspend  

Task B  
   ↓ suspend  

Task C  
   ↓ suspend

暂停中的任务只保留：

Coroutine Frame

而不是完整线程。

所以可以存在：

成千上万个等待中的协程

这也是异步服务器大量采用 Coroutine / Fiber / async-await 模型的原因之一。

---

# 39. 但是 C++ Coroutine 不是 Fiber

也需要区分：

Stackful Coroutine

和：

Stackless Coroutine

C++20 Coroutine 是：

# Stackless Coroutine

也就是：

无独立调用栈协程

假设：

void A()  
{  
    B();  
}  

void B()  
{  
    co_await X;  
}

这是不允许简单实现成：

B暂停  
同时把A的普通函数调用栈一起暂停

因为 C++ Coroutine 的 suspend point 只存在于：

协程本身

所以调用链一般需要：

Task A()  
{  
    co_await B();  
}  

Task B()  
{  
    co_await X;  
}

即：

协程链

逐层组合。

而 Fiber 通常是：

有自己的栈

可以从调用栈很深的位置整体暂停。

---

# 40. Stackless 和 Stackful 对比

|特性|C++20 Coroutine|Fiber|
|---|---|---|
|模型|Stackless|Stackful|
|独立栈|没有|有|
|Suspend point|显式 `co_await`|可在调用栈深处切换|
|内存|通常较小|每个 Fiber 要栈|
|编译器支持|是|通常运行库|
|状态机|编译器生成|保存完整栈上下文|

C#：

async/await

也是偏 Stackless 的模型。

Unity：

IEnumerator  
yield return

同样是状态机思路。

---

# 41. C++20 协程和 Unity IEnumerator 非常像

你过去如果写过：

IEnumerator Attack()  
{  
    PlayAnimation();  

    yield return new WaitForSeconds(0.5f);  
  
    SpawnHitBox();  

}

那么 C++：

Task Attack()  
{  
    PlayAnimation();  

    co_await WaitSeconds(0.5f);  
  
    SpawnHitBox();  

}

两者思想非常接近。

Unity 编译器也会把：

yield return

转换成状态机。

区别是 C++20：

Awaiter机制  
Promise机制  
Continuation控制  
Scheduler控制

都更加底层和可定制。

---

# 42. 游戏里实际可以怎么设计

例如你做动作系统。

可以设计：

Task Attack()  
{  
    PlayMontage(AttackMontage);  

    co_await WaitNotify("Hit");  
  
    OpenHitBox();  
  
    co_await WaitNotify("HitEnd");  
  
    CloseHitBox();  
  
    co_await WaitMontageFinished();  
  
    FinishAttack();  

}

攻击逻辑变得非常线性。

传统版本可能是：

Montage Started  
    ↓  
AnimNotify Hit  
    ↓  
Gameplay Event  
    ↓  
Combat Component  
    ↓  
Open Hitbox  

AnimNotify HitEnd  
    ↓  
Close Hitbox  

Montage End Delegate  
    ↓  
Finish Attack

协程可以把这些异步事件重新组织成：

开始攻击  
↓  
等待Hit Notify  
↓  
开判定  
↓  
等待HitEnd  
↓  
关判定  
↓  
等待动画结束  
↓  
结束攻击

这就是协程对 Gameplay Code 最大的吸引力之一。

---

# 43. 例如做 UE 的 `WaitSeconds`

概念上：

struct FWaitSeconds  
{  
    float Duration;  

    bool await_ready() const noexcept  
    {  
        return Duration <= 0.f;  
    }  
  
    void await_suspend(std::coroutine_handle<> Handle)  
    {  
        // 交给 TimerManager  
    }  
  
    void await_resume() const noexcept  
    {  
    }  

};

协程：

Task Foo()  
{  
    UE_LOG(LogTemp, Log, TEXT("Start"));  

    co_await FWaitSeconds{1.0f};  
  
    UE_LOG(LogTemp, Log, TEXT("1 second later"));  

}

`await_suspend()` 可以把：

Handle

注册到：

FTimerManager

时间到：

Handle.resume();

这样根本不会：

Sleep(1000)

所以 GameThread 不受阻塞。

---

# 44. 等待动画事件也是同样原理

比如：

co_await WaitAnimNotify("ComboWindow");

内部：

await_ready  
↓  
Notify是否已经满足  

await_suspend  
↓  
注册AnimNotify回调  
↓  
保存coroutine_handle  

动画播放到Notify  
↓  
触发Delegate  
↓  
handle.resume()  

await_resume  
↓  
解除绑定 / 返回结果

所以你可以发现：

**Awaiter 本质是把“事件回调”包装成“看起来同步的代码”。**

这是非常关键的一句话。

传统回调：

PlayAnimation(  
    []()  
    {  
        LoadEffect(  
            []()  
            {  
                DoAttack();  
            }  
        );  
    }  
);

协程：

co_await PlayAnimation();  
co_await LoadEffect();  

DoAttack();

协程并没有消灭异步机制。

它只是把：

Callback  
Delegate  
Future  
Timer  
IO Completion

重新包装成：

co_await

---

# 45. 一个特别好的心智模型

你可以把：

co_await X;

脑补成：

if (!X.IsFinished())  
{  
    保存“执行到这里了”;  

    X完成时恢复我;  
  
    return;  

}  

继续执行;

比如：

Task Attack()  
{  
    PlayAttack();  

    co_await WaitHitFrame();  
  
    SpawnHitbox();  
  
    co_await WaitAttackEnd();  
  
    Finish();  

}

脑中直接翻译成：

PlayAttack  

如果 HitFrame 还没到：  
    记录状态  
    HitFrame到了恢复我  
    暂停  

SpawnHitbox  

如果 Attack 还没结束：  
    记录状态  
    Attack结束恢复我  
    暂停  

Finish

这基本就是 C++20 协程。

---

# 46. 协程真正重要的 5 个对象

建议你牢牢记住这五个概念：

Coroutine Function  
Coroutine Frame  
promise_type  
coroutine_handle  
Awaiter

它们之间大致是：

       Coroutine Function  
              │  
              ▼  
      编译器生成状态机  
              │  
              ▼  
       Coroutine Frame  
       ┌───────────────┐  
       │ promise_type  │  
       │ local vars    │  
       │ state         │  
       └───────────────┘  
              ▲  
              │  
      coroutine_handle  
              │  
              ▼  
           resume()  

co_await X  
     │  
     ▼  
   Awaiter  
┌────────────────┐  
│ await_ready │  
│ await_suspend │  
│ await_resume │  
└────────────────┘

---

# 47. `promise_type` 和 Awaiter 不要混淆

这是新手特别容易混的。

promise_type

负责：

整个协程

比如：

怎么创建返回对象  
刚创建时是否暂停  
协程结束时怎么办  
返回值怎么保存  
异常怎么保存

也就是：

get_return_object()  
initial_suspend()  
final_suspend()  
return_value()  
return_void()  
unhandled_exception()

---

Awaiter

负责：

某一次 co_await

例如：

co_await WaitSeconds(1);

这个等待行为由：

WaitSeconds Awaiter

负责：

await_ready  
await_suspend  
await_resume

可以记成：

promise_type  
= 协程级别  

Awaiter  
= suspend point级别

---

# 48. C++20 Coroutine 不是一个完整异步框架

这一点非常重要。

标准只给了：

<coroutine>  

std::coroutine_handle  
std::coroutine_traits  
std::suspend_always  
std::suspend_never

以及：

co_await  
co_yield  
co_return

但是它没有直接给你：

std::task  
std::generator // C++20没有  
std::async_timer  
std::io_context

所以 C++20 协程更像：

> 编译器级基础设施。

真正项目里一般要配合：

asio  
cppcoro  
folly  
UE自己的调度系统  
自定义Task  
自定义Scheduler

来使用。

后续标准才逐步补更多协程相关库设施。

---

# 49. 为什么设计得这么底层

因为 C++ 不想规定：

一定用线程池  
一定用事件循环  
一定动态分配  
一定抛异常  
一定引用计数

你可以设计：

游戏主线程 Scheduler

也可以：

服务器 IO Scheduler

也可以：

GPU Task Scheduler

甚至嵌入式环境：

不允许动态分配

也能设计对应 Coroutine 类型。

代价就是：

学习曲线高

但优势是：

控制力极强

---

# 50. 最后把整个 C++20 协程压缩成一句话

看到：

Task Attack()  
{  
    PlayAnimation();  

    co_await WaitHitFrame();  
  
    DealDamage();  
  
    co_return;  

}

不要想成：

启动一个线程

应该想成：

编译器创建一个 Attack 状态机  
        ↓  
把状态保存在 Coroutine Frame  
        ↓  
执行 PlayAnimation  
        ↓  
遇到 WaitHitFrame  
        ↓  
Awaiter 判断是否需要暂停  
        ↓  
需要暂停：  
保存 coroutine_handle  
        ↓  
当前函数退出CPU执行  
        ↓  
HitFrame事件发生  
        ↓  
handle.resume()  
        ↓  
恢复Coroutine Frame  
        ↓  
从 co_await 后继续  
        ↓  
DealDamage  
        ↓  
co_return

最值得记住的关系就是：

              C++20 Coroutine  
  
                  │  
          编译器自动生成  
                  ▼  
              状态机  
                  │  
                  ▼  
          Coroutine Frame  
        ┌──────────────────┐  
        │ promise_type     │  
        │ 局部变量         │  
        │ 当前执行位置     │  
        └──────────────────┘  
                  ▲  
                  │  
          coroutine_handle  
                  │  
               resume  
                  │  
                  ▼  
            继续执行代码  

co_await X  
    │  
    ▼  
 Awaiter  
    │  
    ├─ await_ready()  
    │  
    ├─ await_suspend(handle)  
    │  
    └─ await_resume()

如果你准备真正把协程用进 **UE 的 Gameplay 系统**，下一步最值得讲的是：自己从零实现一个真正可用的 `Task<T>`，然后实现 `co_await WaitSeconds()`、`co_await AnimNotify`、`co_await Delegate`。这样你会把 `promise_type → coroutine_handle → Awaiter → continuation` 整条链一次性彻底看懂。