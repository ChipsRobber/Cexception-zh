CException ![CI](https://github.com/ThrowTheSwitch/CException/workflows/CI/badge.svg)
==========

_该文档分发使用 Creative Commons 3.0 Attribution Share-Alike 许可证_

CException 是一个简单的C语言异常处理器，运行速度明显高于C++的异常处理，但代价是丢失了一些灵活性。
CException 可以被移植到任何支持 `setjump`/`longjump` 的平台。

快速开始
================

这是最简单的方法，你只需要拉取代码，合并到你的项目里：

```
git clone https://github.com/throwtheswitch/cexception.git
```

如果你想要为这个项目贡献，你需要安装 Ruby 和 Ceedling 用来进行单元测试。

用法
=====

### 它有什么用处？

大多数的错误处理。将错误传递给长长的函数调用链，一点都不优雅。
要是可以选择一个地方处理所有错误就好了，来看这个简单示例：

CException 内部使用C标准库中的 setjump 和 longjump 函数，因此只要你的系统定义了这两个函数，就可以在经过少量配置后在系统上使用。
CException 甚至支持使用多个程序流的环境，例如在实时操作系统上使用。
另外，虽然项目是为嵌入式系统设计的，但其也可被用于大型系统。

### 使用 CException 进行错误处理:

```
void functionC(void) {
  //假装这里写了一些代码
  if (there_was_a_problem)
    Throw(ERR_BAD_BREATH);
  //如果发生了错误，这里的代码就不会执行
}
```

在外界，大量的异常处理框架都使用类似 setjump/longjump 的方法，以后可能也是如此。
不幸的是，在我们开始我们最近的嵌入式项目时，那些框架都存在两个问题 (a) 不支持多任务（因此也不支持多个栈结构）(b) 过于复杂。所以我们编写了 CException。

为什么?
====

### 使用 ANSI C

...并且总比到处传递错误码要好。
_原文就这样写的，不知道在指啥_

### 如果想要事情更简单

CException 抛出一个单一的 ID。你可以将这些 ID 定义为你想要的任何含义。
甚至可以选择这个数字ID的类型。但是就目前而言，我们对传递一个对象或者结构或者字符串什么的不感兴趣... 简单的错误码足以，快速，易用且容易理解。

### 高性能

CException 可以被配置为单任务或多任务。在单任务中，除了 setjump/longjmp（这两个已经很快了） 外几乎没有开销。在多任务中，唯一增加的开销就是你决定一个独一无二的任务id（从 0 到 num_tasks）的时间。

怎么做?
====

包装在 `Try { }` 块中的代码是_被保护(protected)_的
在其中如果任何 Throw 被调用（即抛出错误），程序控制将被直接转移到 Catch 块，
Catch 块紧跟在 Try 块后面，如果没有调用 Throw，Catch 块将不会被执行。

一个数值类型的异常ID作为 Throw 调用的参数，并被传入 Catch 块。这将告诉你错误的类型并允许你用不同的方式处理。或者也可以告诉你抛出错误的位置以便你更方便的进行调试。

Throw 可以发生在 Try 块的任何位置，直接在你正在测试的函数甚至函数调用里面（随你喜欢嵌套多深）。你想 Throw 多少次都可以，只需要注意一旦第一个 Throw 被触发，整个 Try 块就会结束执行。一旦你 Throw，你就会被转移到 Catch 块，一个简单的实例：

```
void SillyExampleWhichPrintsZeroThroughFive(void) {
  volatile CEXCEPTION_T e;
  int i;
  while (i = 0; i < 6; i++) {
    Try {
      Throw(i);
      //这个位置的代码永远不会执行
    } 
    Catch(e) {
      printf(“%i “, e);
    }
  }
}
```

限制
===========

CException 被设计的尽可能快速，并提供基础的异常处理。但并不如 C++ 的异常处理成熟。因此，为了正常的使用 CException，你需要遵守一些限制：

### Return & Goto

不要在 `Try` 块中直接 `return`, 也不要使用 `goto` 进入或者退出 `Try` 块。
`Try` 宏定义会分配一些内存空间和修改一些全局指针。这些工作之后会在 `Catch` 宏中清理。goto 和 return 语句会导致跳过其中的一些步骤，导致内存泄漏或无法预测的行为。

### 局部变量

如果 (a) 你在 `Try` 块中更改了局部变量（也就是栈内的变量），并且 (b) 希望被更新后的变量值在异常抛出后依旧可用，那么这些变量声明需要使用 `volatile` 修饰。

注意这只适用于局部变量，并且你需要在 `Throw` 之后访问那些变量的情况。

由于编译器优化（当然这实际上是件好事）。没有方法确保是真实位置的内存被更新又或者只是寄存器被更新，除非变量被声明为 volatile，那么只会使用内存中的变量，而不是寄存器缓存的值。

### 内存管理

抛出异常时，`Try` 块中 `malloc` 的内存不会被自动释放。这有时是我们期望的，有时则不是。因此你需要在 `Catch` 块中编写代码来负责执行这类的清理工作。

没有很简单的方法可以跟踪 `malloc` 之类的函数所分配的内存，除非替换或封装 `malloc` 调用。但这是一个轻量级的框架，因此这些选项不可取。

CException API
==============

### `Try { ... }`

`Try` 是一个宏，用于开始一个被保护的块。在它后面必须紧跟由一对大括号包裹的被保护代码块，或者一个单行的被保护的代码（和 'if' 一样)，这被称为一个 `Try` 块。在 `Try` 块后面必须紧跟一个 `Catch` 块（不用担心，如果你忘记的话会有编译错误告诉你）

`Try` 块是你的被保护块。它包含了你的主程序流，在这里你可以忽略错误（除了快速调用 `Throw`）。你可以嵌套多个 `Try` 块如果你想要处理多个层级的错误，你甚至可以在 `Catch` 块中重新抛出错误。

_:鬼知道 quick `Throw` calls 啥意思_

### `Catch(e) { }`

`Catch` 是一个宏用于结束 `Try` 块并开始一个错误处理块，只有在 `Try` 块内抛出异常时，才会执行 `Catch` 块中的代码。这个错误是由 `Try` 块（或由 `Try` 块调用的某个函数，或者由该函数进一步调用的其他函数）内部某处的 `Throw` 调用引发的。

`Catch` 接受一个类型为 `CEXCEPTION_T` 的单一错误码标识符，你可以忽略它或以某种方式来处理错误。你可以在 `Catch` 块中抛出错误，但他们会被包裹该 `Catch` 块的 `Try` 块捕获，而不是紧挨在之前的 `Try` 块。

### `Throw(e)`

`Throw` 方法用于抛出错误。`Throw` 应该只能在被保护的(`Try`...`Catch`)块中调用，它可以容易的嵌套在深层的函数调用中且不影响性能和功能。`Throw` 接受单个错误码参数，并传递给 `Catch` 以提示错误发生的原因。如果你希望_重新抛出(re-throw)_一个错误，可以调用 `Throw(e)` 并传入刚刚你捕获的错误码。从 `Catch` 块中抛出错误是被允许的。

### `ExitTry()`

`ExitTry` 方法可以立刻退出你的当前 Try 块且不会看成是抛出了一个错误。退出后不会运行 Catch 块，而是从 Catch 块后面的位置继续运行，就好像什么都没有发生一样。

配置
=============

CException 是一个大部分可移植的库。它有一个通用的依赖，并且在多任务环境工作则需要一些宏定义。

C标准库 setjump 必须是可用的。但因为这是标准库的一部分，所以无需担心。

如果工作在多任务环境，则每个任务都需要一个栈帧。因此，你必须定义一个方法用于获取栈帧数组的索引和获取所需要的id的总数。如果操作系统支持检索任务id的方法，并且这些任务id是 0, 1, 2... 那么这是一个理想的情况。否则，你可能需要一个复杂的映射函数。注意这个函数在每个受保护的块中可能会被调用两次，一次是在 throw 的时候。这只会在系统中添加一些开销。

你可以用一些选项配置该库，如果默认配置对你来说不够好。你可以直接在命令行中添加define宏定义。你可以总是在 #include `CException.h` 之前 #include 一个配置文件。你可以定义 `CEXCEPTION_USE_CONFIG_FILE`，这将会强制让 CException 寻找 `CExceptionConfig.h`，你可以在其中定义任何你想要的内容。无论怎样，你都可以覆盖以下部分或全部的配置：

### `CEXCEPTION_T`

将其设置为一个你想要的异常id类型。默认值为 'unsigned int'。
Set this to the type you want your exception id's to be. Defaults to an 'unsigned int'.

### `CEXCEPTION_NONE`

将其设置为一个在你的系统中永远不会成为异常id的数字。默认为 `0x5a5a5a5a`。
Set this to a number which will never be an exception id in your system. Defaults to `0x5a5a5a5a`.

### `CEXCEPTION_GET_ID`

_鬼知道你的#2是哪个啊_
如果是多任务环境，则应当被设置为上面所描述的函数调用。
它的默认值函数在任何情况都只会返回 0（用于单任务环境，对其他情况不适用）
If in a multi-tasking environment, this should be set to be a call to the function described in #2 above. 
It defaults to just return 0 all the time (good for single tasking environments, not so good otherwise).

### `CEXCEPTION_NUM_ID`

如果在多任务环境，应当被设置为你所需要的ID的数量（通常是系统中任务的数量）。默认为 1（适用于单任务环境或你只需要使用一个任务的系统）
If in a multi-tasking environment, this should be set to the number of ID's required (usually the number 
of tasks in the system). Defaults to 1 (good for single tasking environments or systems where you will only 
use this from one task).

### `CEXCEPTION_NO_CATCH_HANDLER (id)`

该宏是可选的。它允许你指定当 Throw 发生在 Try...Catch 保护外时被调用的代码。当出现严重情况时，将此作为紧急的备用计划。
This macro can be optionally specified. It allows you to specify code to be called when a Throw is made 
outside of Try...Catch protection. Consider this the emergency fallback plan for when something has gone 
terribly wrong.

### 还有更多！

_什么玩意啊_
您可能还想在这里包含一些在应用程序的其余部分使用异常处理时通常会需要的头文件。例如，操作系统头文件或者异常处理代码将会很有用。

最后，这里有一些钩子宏，你可以实现并注入你的特定目标代码中。虽然你很少会用到，但只要你需要就会有。

* `CEXCEPTION_HOOK_START_TRY` - 在 Try 块之前被调用
* `CEXCEPTION_HOOK_HAPPY_TRY` - 如果 Try 块没有抛出任何错误，则在之后被调用
* `CEXCEPTION_HOOK_AFTER_TRY` - Try 块之后或捕获异常之前被调用
* `CEXCEPTION_HOOK_START_CATCH` - 在 Catch 块之前被调用

Finally, there are some hook macros which you can implement to inject your own target-specific code in 
particular places. It is a rare instance where you will need these, but they are here if you need them:

* `CEXCEPTION_HOOK_START_TRY` - called immediately before the Try block
* `CEXCEPTION_HOOK_HAPPY_TRY` - called immediately after the Try block if no exception was thrown
* `CEXCEPTION_HOOK_AFTER_TRY` - called immediately after the Try block OR before an exception is caught
* `CEXCEPTION_HOOK_START_CATCH` - called immediately before the catch

测试
=======

如果你想确认 CException 是否在你的工具或自定义配置下工作，你或许想要运行包含的测试套件。这是我们使用的测试套件（以及我们用在真实项目上面的）用来确保这些的确在按照我们所声明的那样工作。
测试套件使用Ceedling，使用 Unity 测试框架。它还需要一个本机的 C 编译器，在示例的 makefile 和 rakefile 里使用的是 gcc
If you want to validate that CException works with your tools or that it works with your custom 
configuration, you may want to run the included test suite. This is the test suite (along with real 
projects we've used it on) that we use to make sure that things actually work the way we claim.
The test suite makes use of Ceedling, which uses the Unity Test Framework. It will require a native C compiler. 
The example makefile and rakefile both use gcc. 
