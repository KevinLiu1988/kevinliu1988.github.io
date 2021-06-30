# DartVM介绍

> 原文地址：https://mrale.ph/dartvm/

> 此文的书写目的
>
> 此文章用来作为DartVM团队新成员、潜在的外部贡献者或仅仅是对VM内部有兴趣的任何人提供参考。本文对DartVM进行了高度的概括，并对各种内部组件的细节进行了一些描述。

DartVM是用于在原生环境执行Dart代码的一系列组件的集合。其主要包含了一下内容：

- 运行时系统
  - 对象模型
  - 垃圾回收
  - 快照
- 核心库的native方法
- 通过服务协议形成的众多开发体验组件，如调试、性能测量、热加载
- JIT和AOT编译管线
- 解释器
- ARM模拟器

“Dart VM”是一个历史命名。Dart VM是一个为高级编程语言提供运行环境的虚拟机，然而Dart并不总是解释型或者说是JIT编译的。比如说，在使用通过AOT编译管线Dart代码将会被编译为机器码，然后运行在一个阉割版本的DartVM中，这个DartVM叫做预编译运行时，这个运行时中并不包含用来动态运行Dart源码的编译组件。

> 译者注
>
> 第一句话稍显诡异，我妄加揣测是作者想要说Dart的运行时环境与以往的VM有所不同，其在不同的环境中有着不一样的效率和目标，而不要一概而论的理解成为虚拟机的效能就会偏低，DartVM也是可以直接执行机器码的。

## DartVM如何运行你的代码？

Dart VM有很多种方式运行代码，比如：

- 用JIT模式运行源码或者是内核二进制文件（kernel binary）
- 运行快照（AOT和AppJIT）

两者的主要区别在于VM在何时、以何种方式把Dart源码转换成了可执行代码，但始终运行时环境是有利于执行的。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210506225620.png)

在DartVM中每一行代码都一定是运行在某个isolate中的，isolate——即一个独立的Dart宇宙，其拥有自己的内存（堆）和自己的线程控制（mutator线程）。Dart代码可以在多个isolate中并发的执行，但是他们之间并不能直接共享任何状态，只能通过端口之中传递消息的形式进行沟通（这里的端口并不是网络概念中的端口）。

关于系统线程和isolate之间的关系是有一点模糊的，高度取决于DartVM在一个应用中是如何被嵌入的，仅有以下两点可以确保：

- 同一时间一个系统线程只能进入到一个isolate中，如果要进入到另外一个，那么它必须先离开当前的isolate
- 同一时间有且仅有一个mutator线程和一个isolate有关联。mutator线程是一个执行Dart代码的线程，并且可以使用VM的C语言开放API接口

所以，可以是同一个原生系统线程先进入了一个isolate执行Dart代码，然后离开进入到另一个isolate。或者是许多不同的原生线程进入同一个isolate去执行Dart代码，但不是同时的。

除了每个isolate会关联的mutator线程，其也会关联其他的许多辅助线程，比如：

- 后台的JIT编译线程
- GC清理线程
- 并发的GC标记线程

> 译者注
>
> 一个Dart虚拟机中可能仅有一个用来执行Dart代码的线程，反复复用于isolate之间，当然也可能是多个。这和我们自己开辟线程池的逻辑类似，需要找到一个性价比最高的值去界定怎样才是性能较高的。在后续分析Flutter中DartVM的实现时可以着重关注这一点，以学习理解成熟的框架是如何决策多线程的最佳性能问题的。

在DartVM内部会使用线程池`dart::ThreadPool`去管理原生线程，相对于使用原生的线程概念，在VM中会使用`dart::ThreadPool::Task`这个概念。

> 译者注
>
> 我的理解是，DartVM的设计者为了抹平或者说消除Thread这么一个比较重的概念，转而创造了Task这个概念去抽象任务、过程。其他开发者只需要关注Task、发出Task就好，至于背后则会有更高效的线程利用机制来最大化运行时性能。这样开发者就只需要关注任务、过程本身，而不用总是考虑是否要开辟、复用乃至回收原生线程这种性能敏感型操作。

打个比方：在GC后会去发起一次后台清除动作，比起去产生一个专用的线程，DartVM的做法是发出一个`dart::ConcurrentSweeperTask`，将这个任务给到全局的虚拟机线程池。由这个线程池的来决定，是选择一个当前闲置的线程，还是因为没有线程可用而去构造一个新线程。同样的，isolate的消息处理机制EventLoop，在收到一个新消息时，默认的实现也并不是直接产生一个专用线程，而是发出一个`dart::MessageHandlerTask`任务到线程池中去处理。

> 源码导读：
>
> - [`dart::Isolate`](https://github.com/dart-lang/sdk/blob/2d064faf748d6c7700f08d223fb76c84c4335c5f/runtime/vm/isolate.h#L974)代表一个isolate
> - [`dart::Heap`](https://github.com/dart-lang/sdk/blob/2d064faf748d6c7700f08d223fb76c84c4335c5f/runtime/vm/heap/heap.h#L35)代表一个isolate的堆内存
> - [`dart::Thread`](https://github.com/dart-lang/sdk/blob/2d064faf748d6c7700f08d223fb76c84c4335c5f/runtime/vm/thread.h#L239)描述了依附在一个isolate上的线程的状态，这里的Thread并不是原生线程，不要混淆。因为所有依附在同一个isolate上用作mutator的线程都会复用同一个Thread实例。这一点，可以通过isolate的默认消息处理过程来了解：[`Dart_RunLoop`](https://github.com/dart-lang/sdk/blob/2d064faf748d6c7700f08d223fb76c84c4335c5f/runtime/vm/dart_api_impl.cc#L1931)和[`dart::MessageHandler`](https://github.com/dart-lang/sdk/blob/2d064faf748d6c7700f08d223fb76c84c4335c5f/runtime/vm/message_handler.h#L20)

### 通过JIT运行源码

这一节将要解释当你通过命令行执行Dart时候发生了什么：

```dart
// hello.dart
main() => print('Hello, World!');
```

```shell
$ dart hello.dart
Hello, World!
```

Dart 2之后，VM不再支持直接执行原始代码，转而支持去执行包含了序列化后的内核抽象语法树内容（[Kernel ASTs](https://github.com/dart-lang/sdk/blob/master/pkg/kernel/README.md)）的内核二进制（kernel binary，即dill格式的文件）文件。将源码编译成内核抽象语法树的任务是通过通用前端（[common front-end，即CFE](https://github.com/dart-lang/sdk/tree/master/pkg/front_end)）做到的。此工具是用Dart书写的，在各个不同的Dart工具之间共享（比如VM，dart2js，Dart Dev Compiler）。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210506235847.png)

为了保留可以直接从源码运行的便捷功能，`dart`这个命令行工具提供了一个叫做内核服务（kernel service）的辅助isolate，用来将Dart源码编译成为内核。然后VM就可以执行这个内核二进制文件了。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210507000217.png)

这并不是联合CFE和VM去执行Dart代码的唯一途径。比如说，Flutter就将两个过程（编译产生内核，运行内核）完全分开到不同的设备上了：

- 在开发者的机器（宿主）中进行编译
- 通过`flutter`命令行工具将内核二进制文件推送到目标手机设备中执行

> 译者注：当然，Flutter中也包括了将内核二进制文件打包到APP中，而不仅仅是通过命令行推送

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210507000624.png)

请注意，`flutter`这个工具并没有自己处理Dart的解析，而是产生了另一个常驻进程`frontend_server`，这大体上是一个CFE很薄的封装，还有一些Flutter特殊的对于内核转换过程中的处理。是此进程将Dart源码编译成为内核文件，然后`flutter`工具再将其推送到设备中。此进程之所以常驻是因为当开发者需要进行热重载时，其可以利用之前CFE的编译状态作对比，只是编译那些实际变化了的库以加速编译过程。

一旦内核二进制被加载到了VM中，那么它就会被解析进而创造不同的程序节点所代表的对象。整个过程是懒加载的，一开始只会有最基础的库和类的信息被加载。每一个节点都会持有一个指回内核二进制的指针，这样一来就可以在需要时候加载更多的信息了。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210508010434.png)

> 每当我们去代指在VM内部所申请的对象时候，我们都用Untagged作为前缀。这是一个VM自己的命名约定，那些定义在C++类中的内部VM对象总是要以此作为前缀，这些定义都在头文件`runtime/vm/raw_object.h`中。比如`dart::UntaggedClass`这种类型的VM对象就代表了一个Dart类，而`dart::UntaggedField`则代表了一个类中的变量，如此等等。但是在上图中，为了简洁，我们去除了Untagged这个前缀，希望读者能够理解到这一点。

只有当运行时在后边真正需要的时候类相关的信息才会被全量反序列化出来（如查找类中的成员、构造一个实例对象等）。这种场景下类的成员们才会从内核二进制中读取出来，而此时完整的方法体却还是没有被全量反序列化的，此时仅仅只有他们的签名信息而已。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210508012350.png)

此时此刻，运行时就已经从Kernel二进制中加载了足够的信息用于解释和执行方法。比如说，可以找到库中的main方法并执行它。

> 源码导读
>
> - `package:kernel/ast.dart`这个文件定义了从内核AST中读取出的类的结构。`package:front_end`是用来解析Dart源码并编译成内核AST结构数据
>
> - `dart::kernel::KernelLoader::LoadEntireProgram`是将KernelAST结构反序列化为VM对象的入口点
> - `pkg/vm/bin/kernel_service.dart`实现了KernelService Isolate，而`runtime/vm/kernel_isolate.cc`作为胶水将Dart的实现黏合给其他的VM部分使用
>
> > 译者注
> >
> > KernelService有两个作用：
> >
> > 1. 用于单独执行来编译Kernel文件
> > 2. 在JIT模式下用做性能辅助，提升JIT性能
>
> - `package:vm`包含了大多数基于Kernel的VM特殊方法，比如Kernel到Kernel的转化，但因为历史原因还有一些转换仍然存在于`package:kernel`包下。比如用来解析`async`，`async*`，`sync*`语法糖的操作就在`package:kernel/transformations/continuation.dart`中

> 小试身手
>
> 如果你对Kernel格式以及它在VM中特殊用法感兴趣，你可以使用`pkg/vm/bin/gen_kernel.dart`以Dart源文件作为输入产生一个Kernel二进制文件。所产生的二进制文件可以用`pkg/vm/bin/dump_kernel.dart`将信息再打印出来
>
> ```shell
> # 使用CFE将hello.dart编译成为hello.dill这样的kernel二进制
> $ dart pkg/vm/bin/gen_kernel.dart                        \
>        --platform out/ReleaseX64/vm_platform_strong.dill \
>        -o hello.dill                                     \
>        hello.dart
> 
> # 将文本格式的KernelAST数据打印出来
> $ dart pkg/vm/bin/dump_kernel.dart hello.dill hello.kernel.txt
> ```
>
> 在上边使用`gen_kernel.dart`时有个配置项叫做`--platform`，其所指的文件是包含了核心库的AST数据的Kernel二进制文件。如果你已经配置了DartSDK的编译环境，那么你就直接指向其out目录下的`out/ReleaseX64/vm_platform_strong.dill`文件即可。否则你要用`pkg/front_end/tool/_fasta/compiled_platform.dart`来生成：
>
> ```shell
> # 给定核心库列表来生成对应的平台文件和outline文件
> $ dart pkg/front_end/tool/_fasta/compile_platform.dart \
>        dart:core                                       \
>        sdk/lib/libraries.json                          \
>        vm_outline.dill vm_platform.dill vm_outline.dill
> ```

起初，所有方法都只是有一个容器，而并不是实际的方法体可执行代码，这个容器叫做`LazyCompileStub`，由这个容器去要求运行时系统生成当前方法的可执行代码，然后在最后执行这些新生成的代码。

> 真实场景中并不是所有方法都有实际的Dart/Kernel AST数据体。比如C++中定义的native方法或者DartVM所产生的人造无效方法，这种场景下IL（中间语言）只是凭空创造出来的，而并不是由AST生成

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210621192325.png)

当方法被首次编译时，他们是被“未优化编译器”所处理的：

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210630111105.png)

“未优化编译器”通过两轮处理来产生机器码：

1. 遍历方法体序列化后的AST以生成控制流图形数据（CFG）。CFG是由基础的块组成的，块中的数据是IL指令。此时所使用的IL指令集类似于基于栈的虚拟机指令：从栈中取数，处理后的数据也会压入栈
2. IL相对于机器码是一对多的关系：即每一个IL指令会扩展为多个机器指令码。CFG会被直接编译为对应的机器码

此时并未进行编译优化，“未优化编译器”的主要目的就是尽快产生可执行代码。

这意味着“未优化编译器”并不会对任何未曾在KernelBin中解析过的调用进行静态解析，所以像是`MethodInvocation`或`PropertyGet`这样的AST调用节点就像是完全动态的一样被编译的。当前，VM并不会使用像是基于虚拟表（virtual table）或接口表（interface table）的分发机制，而是实现了使用内联缓存（[inline caching](https://en.wikipedia.org/wiki/Inline_caching)）的动态调用机制。

> 摘自维基百科
>
> **Call site**
>
> - 表明某个方法在哪一行被调用了
> - 表明在哪里（哪一行）这个方法被传入了0个或N个参数，返回了0或N个值
>
> call site是一个关于方法的定位标识符。
>
> **Inline caching**
>
> 内联缓存首次出现在Smalltalk，是很多语言运行时都会使用的优化技术方案。其目标是通过记忆上一次在某个call site找到的方法结果来加速运行时方法绑定。此方案对动态类型语言尤其有效，因为其大多数的方法绑定都发生在运行时，并且v-table无法使用

内联缓存的核心理念是对单个call site的方法解析结果进行缓存。这种机制在VM中的应用包含两种：

- 通过`dart::UntaggedICData`对象作为一个call site的特殊缓存。其将接受者的类映射到一个方法，此方法会在接受者匹配这个类型时候被调用。缓存也存储了一些辅助信息，比如调用频率计数器，来跟踪在这个call site上给定的类有多频繁的出现过
- 通过共享的查找存根实现方法的快速调用。此存根会去查找给定缓存去看是否包含了符合接受者类型的节点。如果找到了这样的节点，那么存根表会去增加其频次计数，然后尾调用（？）该缓存方法。否则运行时的系统辅助器将会触发方法解析逻辑，如果方法成功解析出来那么就会更新缓存，然后串行调用就没有必要进入运行时系统了

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210628111403.png)

上图说明了对于`animal.toFace()`这个call site节点下相关的内联缓存结构及其状态。这个call site会在Dog实例下被调用一次，Cat实例下又被调用一次。

“未优化编译器”自己完全有能力执行任何Dart代码，但是它所生成的代码执行起来是相当慢的。这就是为什么VM还要实现一个自适应优化的编译管线。这个自适应优化的设计思路是根据运行中程序的执行测量数据来驱动优化决策。

未优化代码执行过程中会收集一下信息：

- 内联缓存会去收集在call site中相关的接受者类型
- 通过对方法和方法内的基础代码块的执行计数来跟踪代码热区

当某个方法的执行计数到达了某个确定的阈值后，此方法就会被提交到一个后台的“优化编译器”中进行优化处理。

“优化编译”和“未优化编译”采取了同样的启动方式：

遍历序列化后的KernelAST数据，然后先为待优化方法构建出未优化的IL。然后“优化编译器”并没有直接将IL翻译成为机器码，而是将未优化的IL翻译成了基于SSA（static single assignment 静态单赋值）结构的优化过的IL。SSA格式的IL数据会根据所收集的类型反馈进行专业化的推测，并且经过一系列传统的和Dart特殊的优化，比如内敛处理、范围分析、类型传播、表征选择、StoreLoad和LoadLoad转发、全局值编号、分配下沉等等（译者注：大多是编译器的名词，译者了解不深，如果有了解的看官敬请指教）。最终，经过了优化的IL会通过线性扫描寄存器分配器和简单的一对多的IL指令降级成为对应的机器码。

一旦编译工作在后台完成了，编译器就会请求mutator线程进行入安全点（safepoint），然后将编译过后的代码附着到对应的方法上去。

> 关于safepoint安全点
>
> 大体意思是说在虚拟中的某个线程，当它相关的状态（栈帧、堆内存状况等）一直没变，并且本线程不通过中断的方式就可以访问或者修改，那么就认为这个线程当前处于safepoint安全点了。通常这表明一个线程并不是暂停的也不是整在可控环境（虚拟机）外执行代码，比如执行管理范围外的native代码。

下一次方法被调用，就已经是使用了优化过的代码了。因为有些方法会包含较长的运行循环（运行较长时间），所以经常在运行的时候会产生从非优化代码到优化代码的切换动作。这个过程叫做栈上替换（on stack replacement - OSR）。如它的命名一样，实际上的操作就是将代表这个方法的某个版本的栈帧直接替换成为相同方法的另一个版本的栈帧。

> 译者注
>
> 结合新的理解，前边“未优化编译器”的图可以更新一个更完整的版本了

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210628153356.png)

> 源码导读
>
> - 编译器的源码在`runtime/vm/compiler`目录下
> - 编译管线的入口点在`dart::CompileParsedFunctionHelper::Compile`
> - `runtime/vm/compiler/backend/il.h`定义了IL
> - 内核到IL的翻译过程在`dart::kernel::StreamingFlowGraphBuilder::BuildGraph`开始，这个方法同样负责了为不同认为函数构造IL的任务
> - `dart::compiler::StubCodeCompiler::GenerateNArgsCheckInlineCacheStub`负责为内联存根生成机器码，`InlineCacheMissHandler`负责了内联缓存未命中的情况
> - `runtime/vm/compiler/compiler_pass.cc`定义了优化编译器的轮次以及他们的顺序
> - `dart::JitCallSpecializer`做了绝大多数基于专业推断的类型反馈

> 小试身手
>
> VM可以通过标志来控制JIT，把通过JIT编译出来的方法的机器码和IL都打印出来
>
> 标记参数与说明：
>
> - `--print-flow-graph[-optimized]`  打印所有（或仅为优化过的）编译出来的IL数据
> - `--disassemble[-optimized]` 拆解所有（或仅为优化过的）编译过的方法
> - `--print-flow-graph-filter=xyz,abc,...` 配合前边的标记参数，仅输出包含了子字符串的那些方法
> - `--compiler-passes=...` 对编译轮次进行进准控制：在每轮前后强制打印IL，通过命名禁用某轮次，输入`help`获取更多使用细节
> - `--no-background-compilation` 关闭后台编译，在主线程进行热函数的编译工作。仅为实验学习有效，否则可能很短的程序没等到后台编译器完成对热函数的处理就已经结束了
>
> 比如：
>
> ```shell
> # 执行test.dart，将包含“myFunction”字样的方法的优化后IL和机器码
> # 都打印出来
> # 并且关闭后台编译，使编译在主线程进行
> $ dart --print-flow-graph-optimized         \
>        --disassemble-optimized              \
>        --print-flow-graph-filter=myFunction \
>        --no-background-compilation          \
>        test.dart
> ```

需要格外强调的是，通过优化编译器所产生的代码是基于应用程序的运行时测量数据进行的推断强相关的。比如说，某个动态call site被观察到是仅仅是通过C这个类的实例进行调用的，那么就会被转换为一个直接调用，这个直接调用会进行期望是类型C的类型校验工作。而这种假设可能在后边的程序执行过程中被打破：

```dart
void printAnimal(obj) {
  print('Animal {');
  print('  ${obj.toString()}');
  print('}');
}

// 使用Cat的梳理对printAnimal进行数次调用，结果是printAnimal这个方法
// 就会根据认为obj始终是Cat类型这个假设进行优化
for (var i = 0; i < 50000; i++)
  printAnimal(Cat());

// 现在，用Dog实例进行printAnimal调用。已经优化的版本就不能正确处理
// 这种类型，因为它之前是在假设类型始终为Cat的前提下被优化的
// 此时，就会引发“反优化”
printAnimal(Dog());
```

每当优化过的代码做出乐观推断时，在执行过程中还是有可能被违反、被破坏。所以需要这样一个机制，能够在这种情况发生时候做出有效恢复，让程序继续执行下去。

这个恢复的机制就是“反优化”：当优化过的版本命中了不能处理的情况，就会把执行过程转向到其非优化版本上去继续执行。方法的非优化版本并不做任何的假设推断，这样就能够处理得来所有可能的输入。

> 进入未优化的方法的实际至关重要，因为代码可能产生副作用（以上边的例子来说，我们已经执行了第一个print，“反优化”是在后续才发生的）
>
> VM中按位置匹配指令集反优化到未优化代码是通过“反优化ID”做到的（这一点，在上边的图中有所提及）

VM通常会在“反优化”之后废弃掉优化过的版本，并且在之后使用更新过的类型反馈数据对其重新进行优化处理。

VM使用两种方式来保护编译器所生成的乐观推断/假设：

- 内立案检查（如CheckSmi、CheckClass IL指令）进行校验。举例说明：在将动态调用转换为直接调用时，会在直接调用之前加入这些检查指令。在检查点上发生的“反优化”被称之为“紧急反优化”或“及时反优化”，因为他们在检查到来时就迫切的进行了
- 全局保护会在优化代码所依赖的内容发生变化时告知运行时废弃这段优化后代码。举例说明：优化编译器可能观察到某个类C从未被继承过，就在后续的类型传播过程中携带着这样的信息。但是后续有可能通过动态代码加载或者类的确定会产生C的子类，这个动作就会使得之前的假设失效。此时，运行时就需要找到所有基于此假设所编译的优化代码，并且将其废弃。有一种可能是运行时会在当前的执行栈上发现无效的优化后代码，这种情况下所影响到的栈帧会被标记为要做反优化，然后会在执行到它们的时候进行反优化。这种反优化类型会被称之为“懒反优化”：因为它会延迟到执行控制权返回到优化过代码时才执行

> 代码导读
>
> 反优化器在`runtime/vm/deopt_instructions.cc`中。它本质上是一个小型解释器，用来将对应状态的优化过代码反优化成为对应的非优化代码指令数据。`dart::CompilerDeoptInfo::CreateDeoptInfo`在编译期为每个优化后代码潜在的反优化位置进行反优化指令的生成

> 小试身手
>
> - `--trace-deoptimization`标记可以让VM在每个反优化发生时打印出原因和位置的相关信息
> - `--trace-deoptimization-verbose`会在反优化过程中打印出执行的每一行反优化指令

###  通过快照运行

VM有能力将Isolate的堆数据或更清晰的堆中的对象图数据序列化成为二进制快照文件。然后再利用快照文件就能在启动VM Isolate时重新创造出相同的状态。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210628192103.png)

快照是一个为了快速启动而产生的低级别、优化过的格式，它本质上是一系列用来创建和指导相互连接关系的对象列表。快照背后的原始思想是：与其通过解析Dart源码然后逐渐创建VM内部的数据结构，VM有能力通过解包快照来快速的启动一个携带着全部必要数据结构的Isolate。

> 快照的理念来自于Smalltalk的image，此灵感受[Alan Kay](https://www.mprove.de/visionreality/media/kay68.html)影响。DartVM使用集群序列化格式，与[快速富特性二进制开发技术](http://scg.unibe.ch/archive/papers/Mira05aParcels.pdf)和[使用Fuel进行集群序列化](https://rmod.inria.fr/archives/workshops/Dia11a-IWST11-Fuel.pdf)文章中描述的技术类似

起初快照并不包含机器码，这种能力是随着AOT编译器被开发出来后加入的。有些平台会因为平台本身的限制，导致JIT无法使用，而为了让VM依然能在这些平台正常运转，才有了开发AOT编译器以及携带代码的快照的必要。

携带代码的快照基本上和普通快照的工作方式类似，只有很小的区别：携带代码的快照会包含一个不需要反序列化的代码段。这个代码段的特殊之处在于，它会在完成内存映射后直接变成堆数据的一部分。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210628194036.png)

> 译者注
>
> 如上图，原本黑色部分数据在经过快照格式的转换后，最终在堆中依然保持原有的样子。机器码被执行了，而这部分数据以快照为中介，完成了到新的运行时的保留目标

> 源码导读
>
> `runtime/vm/clustered_snapshot.cc`处理了快照的序列化和反序列化。如`Dart_CreateXyzSnapshot[AsAssembly]`系列的API负责将堆数据写入到快照中（如`Dart_CreateAppJITSnapshotAsBlobs`和`Dart_CreateAppAOTSnapshotAssembly`）。
>
> 另外，`Dart_CreateIsolateGroup`有选项可以接收快照数据来启动一个Isolate

#### 通过AppJIT快照运行

AppJIT快照的引入是在大型的Dart应用中为如`dartanalyzer`或`dart2js`减少JIT的预热时间。对于小型工程来说，这些工具在VM中所做工作需要耗费的时间基本等同于VM对这些App进行JIT编译的时间。

AppJIT快照能解决这样的一个问题：一个应用可以使用伪造的训练数据在VM中运行，然后所有生成的代码以及VM内部的数据结构就能序列化生成一个AppJIT快照文件。这个快照文件代替以源码或者内核二进制的格式进行分发，VM通过此快照启动。一旦发生了真实的运行状况与训练期间的运行状况不匹配时，其依然可以进行JIT。

> 小试身手
>
> `dart`如果传入了`--snapshot-kind=app-jit --snapshot=path-to-snapshot`参数的话就会在运行了应用程序后产生AppJIT快照文件。下边就是一个`dart2js`产生AppJIT快照的例子：
>
> ```shell
> # 采用JIT模式运行源码
> $ dart pkg/compiler/lib/src/dart2js.dart -o hello.js hello.dart
> # 编译产生hello.js文件
> Compiled 7,359,592 characters Dart to 10,620 characters JavaScript in 2.07 seconds
> Dart file (hello.dart) compiled to JavaScript: hello.js
> 
> # 训练式运行，产生AppJIT快照文件
> $ dart --snapshot-kind=app-jit --snapshot=dart2js.snapshot \
>        pkg/compiler/lib/src/dart2js.dart -o hello.js hello.dart
> # 产生了快照文件dart2js.snapshot
> Compiled 7,359,592 characters Dart to 10,620 characters JavaScript in 2.05 seconds
> Dart file (hello.dart) compiled to JavaScript: hello.js
> 
> # 从快照启动
> $ dart dart2js.snapshot -o hello.js hello.dart
> Compiled 7,359,592 characters Dart to 10,620 characters JavaScript in 0.73 seconds
> Dart file (hello.dart) compiled to JavaScript: hello.js
> ```

#### 通过AppAOT快照运行

AOT快照起初是因为一些无法使用JIT功能的平台引入的，但是在快速启动，以及在潜在的、有性能损伤的情况下追求一致性的性能的场景时AOT同样适用。

无法使用JIT意味着：

1. AOT快照必须包含每一个在应用执行期间会被调用的方法的可执行代码
2. 这些可执行代码不可以依赖于任何可能在运行期间被打破的乐观假设

为了满足以上要求，AOT的编译过程会做全局的静态分析（称为类型流分析 type flow analysis 简称TFA），其依据是入口点、哪些类的实例会被分配出来以及后续的数据流是怎样的，来判断程序中哪些部分会被触达。所有这些分析会很谨慎：意味着他们会在遇到正确性问题时出错。这与JIT形成了鲜明的对比，JIT会在性能上表现不佳，因为它会经常为了正确的执行动作去做反优化。

所有会被触达的方法都会在不含任何推断优化的情况下被编译成为native代码。但其类型流数据依然会被保留用于做专业分析这段代码（比如反虚拟化调用）

> 译者注
>
> Devirtualized——去虚拟化、反虚拟化。对比着前边提到的内联缓存去理解。指的是那些在代码中所书写的代码调用是通过抽象类或父类进行，但实际在运行时会是由具体的或派生类型调用。能够将这样抽象的、不实际的解析成具体的这个过程叫做反虚拟化。

当所有方法都被编译好了，堆的快照就可以取走了。

最终产生的快照可以使用预编译的运行时运行，这是一个不包含JIT、动态代码加载器等组件的特殊的DartVM变体。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210629121011.png)

> 源码导读
>
> - `package:vm/transformations/type_flow/transformer.dart`是进行类型流分析以及基于其结果做转化的入口点
> - `dart::Precompiler::DoCompileAll`是虚拟机中AOT编译的切入点

> 小试身手
>
> AOT编译管线目前已经被打包到DartSDK中名为`dart2native`脚本里：
>
> ```shell
> $ dart2native hello.dart -o hello
> $ ./hello
> Hello, World!
> ```
>
> 像是`--print-flow-graph-optimized`和`--disassemble-optimized`这样的标记参数是无法传递给`dart2native`脚本使用的，所以如果你想了解所生成的AOT代码你需要从源码开始编译编译器：
>
> ```shell
> # 编译可运行AOT代码的可执行运行时
> $ tool/build.py -m release -a x64 runtime dart_precompiled_runtime
> 
> # 然后使用AOT格式编译应用
> $ pkg/vm/tool/precompiler2 hello.dart hello.aot
> 
> # 使用运行时运行AOT快照
> $ out/ReleaseX64/dart_precompiled_runtime hello.aot
> Hello, World!
> ```

尽管通过全局的以及本地的分析，所产生的AOT编译代码仍有可能包含一些不能被反虚拟化的（不能被静态解析）call site。为了补偿这部分的AOT编译代码，运行时会使用在JIT中所集成的内联缓存的一个扩展版本。这个扩展版本就叫做“切换式调用”。

JIT章节中已经描述过，每一个call site所关联的内联缓存中包含两块数据：

1. 缓存对象，`dart::UntaggedICData`类型的实例对象
2. 一块用于调用的native代码，比如`InlineCacheStub`类型

在JIT模式的运行时中会仅仅对缓存部分数据进行更新，但在AOT模式的运行时中可以根据内联缓存的状态有选择性的将两个数据都替换掉。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210629142855.png)

最开始，所有的动态调用都会处于`unliked`的状态。当这些call site首次被触达时，`SwitchableCallMissStub`方法会被调用，这个方法会通过调用运行时辅助方法`DRT_SwitchableCallMiss`来连接这个call site。

有可能的话，`DRT_SwitchableCallMiss`会尝试将此call site的状态过渡到`monomorphic`单态的状态。在这种状态下call site会变成直接调用，调用的方法会通过一个特殊的入口点，这个入口点会校验接收者是否是预期的类型。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210629144359.png)

如上图中的示例，当`obj.method()`被首次调用的时候，`obj`是C的实例，所以`obj.method`就会被解析成为`C.method`。

下一次我们执行同样的call site的时候它就会跳过任何的方法查询过程直接调用`C.method`方法。但这个“直接”也并不是真正的直接，如上所说，它对`C.method`的调用会经过一个特殊的入口点，这个入口点会去校验此时`obj`实例依然是C类型的。如果不是，那么`DRT_SwitchableCallMiss`就会被调用，将去尝试切换到下一个call site状态。

`C.method`依然会是一个有效的调用目标，比如说此时`obj`是D类型，而D是C的派生类，但却为复写`C.method`方法。在这种情况下，会去检查此call site能否过渡到`single target`单目标状态，其实现是`SingleTargetCallStub`（代码在`dart::UntaggedSingleTargetCache`）。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210629145313.png)

经过以上方式进行AOT编译出来的大多数类都被赋予了一个整型ID，这个整型ID是在继承关系中首次遍历到的深度。比如，C有D0到Dn个子类，但是无一复写了C的method方法，那么就可以表示为：

`C.:cid <= classId(obj) <= max(D0.:cid, ..., Dn.:cid)`

这表示在调用`obj.method`方法时就会直接解析到`C.method`。在这种情况下，并不需要对比一个处于单态状态的类，仅仅通过检查所有C的子类的类ID范围就可以了。

另外的情况是，call site可能会切换到去对内联缓存做线性搜索的方式，就像JIT模式中用到的那样（相关代码在`ICCallThroughCodeStub`、`dart::UntaggedICData`和`dart::PatchableCallHandler::DoMegamorphicMiss`）。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210630095725.png)

最后，如果检查所在的线性数组的增长超过了某个阈值，call site就会切换去使用一个类字典式的数据结构（相关代码`MegamorphicCallStub`、`dart::UntaggedMegamorphicCache`以及`dart::PatchableCallHandler::DoMegamorphicMiss`）。

![](https://blog-imgs-1301146282.cos.ap-chengdu.myqcloud.com/20210630100001.png)

## 运行时系统

> 译者注
>
> 原文中，作者表示这一章节接下来会写，但此文章上一次的更新时间是2020.1.29，译者曾邮件联系过作者，希望能够获取相关继续书写的信息，目前尚未得到回复。
>
> 原文中只是列出了“对象模型”的标题，并留下了两个TODO项：
>
> 1. 关于解释AppJIT和CoreJIT快照区别的文档（那么CoreJIT好像是个新的概念，在上文中所提及的就只有AppJIT）
> 2. 在未优化代码中的切换式调用是如何工作的

