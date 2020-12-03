---
title: iOS调度队列
tags: [iOS, concurrent, GCD]
categories: [iOS]
date: 2016-12-15 09:24:00
layout: post
---

GCD调度队列是执行任务的强大工具。调度队列允许您相对于调度者异步或者同步的执行任意代码块。您能够使用调度队列来执行几乎所有在单独线程上执行的任务。调度队列的优点是它们比线程代码更简单且更高效。

下面提供了调度队列的简介，以及在应用程序中怎么使用调度队列执行一般的任务。如果您想用使用调度队列替换已经存在的线程代码，请参阅[线程迁移](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW1)。



### 关于调度队列

调度队列是在应用程序中异步并发执行任务的一种简单方法。任务通常是应用程序需要执行的一些工作。例如，您可能定义一个任务来执行一些计算，创建和修改数据结构，处理从文件中读取的数据，或者任意数量的事情。通过放置相应的代码到函数或者块对象中来定义任务，并把他们放到调度队列中。

调度队列是一个类似对象的结构体，它管理您提交给它的任务。所有的调度队列都是先进先出的数据结构。因此，添加到队列的任务始终以添加他们的相同顺序开始执行。GCD自动为您提供一些调度队列，但您可以为特定目的创建其他队列。下表列出了应用程序中可以使用的调度队列以及怎么使用它们。

| 类型    | 描述                                       |
| ----- | ---------------------------------------- |
| 串行    | 串行队列（也称为私有调度队列）以添加它们到队列的顺序每次执行一个任务。当前执行的任务运行在一个被调度队列管理的不同线程上（可以随任务变化）。串行队列经常用于同步访问特殊资源。 您可以根据需要创建尽可能多的串行队列，每个队列相对于其他队列并行运行。也就是说，如果你创建四个串行队列，每个队列同一时间只执行一个任务，但是仍然可以有多达四个任务同时执行，每个队列一个。有关怎么创建串行队列，请参阅[创建串行调度队列](#创建串行调度队列)。 |
| 并行    | 并发队列（也称为一类全局调度队列）同时执行一个或者多个任务，但任务仍然以被添加到队列的顺序开始执行。当前执行的任务运行在被调度队列管理的不同线程上。在给定的时间点执行的任务数量是可变的，并且取决于系统调节。在iOS5及以后，你可以通过指定队列类型为DISPATCH_QUEUE_CONCURRENT来自己创建并发队列。此外，还有四个预定义的全局并发队列供应用程序使用。有关怎么回去全局并发队列，请参[阅获取全局并发队列](#阅获取全局并发队列)。 |
| 主调度队列 | 主调队队列是一个全局可用的串行队列，它在应用程序的主线程上执行任务。这个队列与应用程序的RunLoop（如果存在）交错处理排队的任务以及添加到RunLoop的其他事件源。因为它运行在程序的主线程上，所以主队列经常作为应用程序的关键同步点。虽然您不需要创建主调度队列，但您需要确保您的应用程序适当的释放它。有关如何管理此队列，请参阅[在主线程中执行任务](#在主线程中执行任务)。 |



当向应用程序添加并发时，调度队列提供了优于线程的几个优点。最直接的优点是工作队列编程的简单性。使用线程，您必须编写执行的工作以及创建和管理线程的代码。调度队列使您专注于您实际想要执行的工作，而不用担心线程的创建和管理。相反，系统会为您处理所有的线程创建和管理。优点是，系统能够比任何单个应用更高效的管理线程。系统可以根据可用资源和当前系统的情况动态调整线程数量。另外，系统通常能够比您自己创线程更快的开始运行您的任务。

虽然您可能认为编写调度队列代码可能是困难的，但是通常编写调度队列比编写线程更简单。编码的关键是设计独立的且可以异步运行的任务。（这实际上对线程和调度队列都是真的。）但是调度队列有可预见性的优点。如果您有两个任务来访问相同的共享资源，但是运行在不同的线程上，每个线程都可以首先修改资源，您可能需要使用锁，以确保这两个任务不能同时修改该资源。使用调度队列，您可以添加两个任务到一个串行队列，以确保在任何给定时间只有一个任务修改资源。这种基于队列的同步比锁更高效，因为锁在有竞争和无竞争的情况下总是需要一个昂贵的内核陷阱，而调度队列主要在应用程序的进程空间中工作，只有在绝对必要时才调用内核。

虽然您可能指出，串行队列中的两个任务不是并发运行，但您必须记住，如果两个线程同时使用锁，线程提供的任何并发都会丢失或者显著减少。更重要的，线程模型需要创建两个线程，这两个线程都占用内核和用户内存空间。调度队列不需要为他们的线程支付相同的内存损失，并且使用的线程保持忙碌且不被阻塞。

谨记以下关于调度队列的一些其他关键点：

- 调度队列相对于其他调度队列并发执行任务。任务的串行限于单个调度队列中的任务。
- 在任何时候系统决定执行任务的数量。因此，在100个不同队列中有100个任务的应用程序可能不会同时执行这些任务（除非它具有100个或者更多有效的内核）。
- 在选择要启动的新任务时，系统会考虑队列优先级。有关如何设置串行队列的优先级，请参阅为[队列提供清理功能](#队列提供清理功能)。
- 当任务被添加到队列时，任务必须准备好执行。（如果您之前使用过Cocoa操作对象，请注意此行为与操作使用的模型不同）。
- 私有调度队列是引用计数对象。除了在您自己的代码中保留队列之外，请注意调度源也可以附加到队列，并且增加其保留计数。因此，您必须确保所有调度源都被取消，并且所有的保留调用（retain call）都通过适当的释放调用（release call）进行平衡。有关保留和释放队列，请参阅[调度队列的内存管理](#调度队列的内存管理)。有关调度源的更多信息，请参阅[关于调度源](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW12)。

有关操作调度队列的接口，请参阅[大中央调度（GCD）参考](https://developer.apple.com/reference/dispatch#//apple_ref/doc/uid/TP40008079)。



### 队列相关技术

除了调度队列，GCD提供了几种使用队列来帮助管理代码的技术。下表列出了这些技术，并提供了找到关于它们更多信息的链接。

| 技术                  | 描述                                       |
| ------------------- | ---------------------------------------- |
| Dispatch Group      | 调度组是一种用来监视一组块对象完成的方法（您可以根据需要同步或者异步监视）。组为依赖于其他任务完成的代码提供一种有用的同步机制。更多有关使用组的信息，请参阅[等待排队任务组](#等待排队任务组)。 |
| Dispatch semaphores | 调度信号量类似于传统的信号量，但通常更高效。只有当调用线程需要被阻塞时，调度信号量才调用内核，因为信号量不可用。如果信号量可用，则不进行内核调用。有关如何使用调度信号量的例子，请参阅[使用调度信号量来调节有限资源的使用](#使用调度信号量来调节有限资源的使用)。 |
| Dispatch sources    | 调度源生成通知以响应特定类型的系统事件。您可以使用调度源来监视事件，例如进程通信，信号和描述符事件等。当事件发生时，调度源异步的将您的任务代码提交到指定的调度队列进行处理。有关创建和使用调度源的更多信息，请参阅[调度源](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW1)。 |



### 使用块实现任务

块对象（Block Object）是基于C语言的功能，可以使用C，Objective-C和C++代码。块使定义独立的工作单元变的简单。虽然他们可能看起来类似函数指针，但块实际上是底层数据结构的表现，类似于对象，由编译器创建和管理。编译器将您提供的代码（以及任何相关数据）打包，并将其封装成可以存在于堆中并传递给应用程序的形式。

块的一个关键优点是它们能够使用自己的词汇作用域之外的变量。当您在函数或者方法中定义块时，块在某些方法充当传统代码块。例如，块可以读取定义在父作用域的变量值。由块访问的变量将被复制到堆上的块数据结构中，因此块可以稍后访问它们。当块被添加到调度队列时，这些值通常必须以只读格式保留。然而，被同步执行的的块也可以使用具有`__block`关键字的变量来返回数据到父作用域。

使用类似于函数指针语法的代码声明内联块。块和函数指针最大的不同是，在块名字之前使用脱字符(^)代替星号(*)。像函数指针一样，可以传递参数给块，从其接收返回值。下面代码展示如何声明和同步执行块。变量`aBlock`被声明为块，接收一个整型参数，没返回值。然后将与该原型匹配的实际块分配给`aBlock`并声明为内联。最后一行立即执行块，将指定证书打印到标准输出。

```objc
int x = 123;
int y = 456;
 
// Block declaration and assignment
void (^aBlock)(int) = ^(int z) {
    printf("%d %d %d\n", x, y, z);
};
 
// Execute the block
aBlock(789);   // prints: 123 456 789
```

下面是设计块时需要注意的一些主要指南的摘要：

- 对于打算使用调度队列异步执行的块，可以安全的从父函数或者方法中获取标量变量并在块中使用它们。然而，不应该试图获取由调用上下文分配和删除的大型结构或者其他基于指针的变量。当块执行时，被该指针引用的内存可能消失。当然，可以自己分配内存（或者对象）并明确的将该内存的所有权交给块。
- 调度队列复制添加给它们的块，并且当它们结束执行时释放块。换句话说，在添加它们到队列之前，您不需要显式的复制块。
- 虽然队列在执行小任务时比原始线程更高效，但仍然有创建块和在队列上执行它们的开销。如果块的工作太少，内联的执行可能比调度到队列成本更低。判断块是否工作太少的方法是使用性能工具收集每个路径的指标，然后进行比较。
- 不要缓存和底层线程相关的数据，并希望从不同的块访问数据。如果同一队列中的任务需要共享数据，使用调度队列的上下文指针来存储数据。 有关如何访问调度队列的上下文数据，请参阅[使用队列存储自定义上下文信息](#使用队列存储自定义上下文信息) 。
- 如果队列创建多个Objective-C对象，则可能需要将块代码的一部分包含在@autorelease块中，以处理这些对象的内存管理。 虽然GCD调度队列具有自己的自动释放池，但它们不能保证何时drain这些池。 如果您的应用程序受内存限制，创建自己的自动释放池允许您以定期的时间间隔释放自动释放对象的内存。


有关块的更多信息，包括如何声明和使用它们，请参阅[块编程](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html#//apple_ref/doc/uid/TP40007502)。有关怎么添加块到调度队列，请参阅[添加任务到队列](#添加任务到队列)。



### 创建和管理调度队列

在将任务添加到队列之前，必须确定要使用的队列类型以及如何使用它。调度队列可以串行或并发执行任务。此外，如果您对队列有特殊用途，您可以相应地配置队列属性。 以下各节介绍如何创建调度队列并对其进行配置。



#### 获得全局并发调度队列

当有多个任务并行运行时，并发调度队列很有用。并发队列仍然是一个队列，它以先进先出的顺序对任务进行出队，然而，在前面任何任务结束之前并发队列可能出队另外的任务。在任何给定时刻，并发队列执行任务的实际数量是可变的，并且随应用程序情况的变化而变化。许多因素影响并发队列执行的任务数，包括可用核心数，其他进程正在完成的工作量，其他串行调度队列中任务数量和优先级。

系统为每个应用程序提供四个并发调度队列。这些队列对应用程序是全局的，并且仅通过优先级来区分。因为它们是全局的，所以不需要显式的创建它们。相反，使用`dispatch_get_global_queue`函数来获取其中一个队列，如下所示：

```objc
dispatch_queue_t aQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```

除了获取默认并发队列，您可以通过传入`DISPATCH_QUEUE_PRIORITY_HIGH`和`DISPATCH_QUEUE_PRIORITY_LOW`常量到函数来获取高优先级和低优先级的队列，或者传入`DISPATCH_QUEUE_PRIORITY_BACKGROUND`常量来获取后台队列。正如您所期望的，高优先级并发队列中的任务在默认优先级和低优先级队列中的任务之前执行。类似的，默认队列中的任务在低优先级队列中的任务之前执行。

> **重要提示：**传入`dispatch_get_global_queue`函数的第二个参数是为将来扩展保留的。现在，您应该总是为此参数传0.

虽然调度队列是引用计数对象，但您不需要保留和释放全局并发队列。因为它们对于应用程序来说是全局的，所以忽略这些队列的保留和释放调用。因此，您不需要保存对这些队列的引用。任何时候您需要引用他们中的一个，只需要调用`dispatch_get_global_queue`函数。



#### 创建串行调度队列

当想要任务按照特定的顺序执行时，串行队列非常有用。串行队列每次只执行一个任务，并且总是从队列首获取任务。您可以使用串行队列代替锁来保护共享资源或者可变数据结构。与锁不同的是，串行队列能够确保任务按照可预见的顺序执行。只要以异步方式提交任务到串行队列，队列就永远不会死锁。

与已经为您创建好的并发队列不同，您必须显式的创建和管理任何您想要使用的串行队列。您可以为您的应用程序创建任意数量的串行队列，但应避免创建大量的串行队列来尽可能多的同时执行任务。如果您想同时执行大量任务，提交他们到全局并发队列。当创建串行队列时，请确定每个队列的用途，例如保护资源或者同步应用程序的某些关键行为。

下面代码显示了创建自定义串行队列所需的步骤。` dispatch_queue_create`函数有两个参数：队列名称和一组队列属性。调试器和性能工具显示队列名称，帮助您跟踪任务如何执行。队列属性是为将来使用预留的，应该总是NULL。

```objc
dispatch_queue_t queue;
queue = dispatch_queue_create("com.example.MyQueue", NULL);
```

除了您创建的自定义队列，系统自动创建串行队列并将其绑定到您应用程序的主线程。有关获取主线程队列的更多信息，请参阅[在运行时获取常见队列](#在运行时获取常见队列)。

> 注意：`dispatch_queue_create`函数第二个参数队列属性，传入参数来决定队列类型：
>
> - DISPATCH_QUEUE_SERIAL，串行队列，也就是NULL，
> - DISPATCH_QUEUE_SERIAL_INACTIVE，也是串行队列，不活跃的（调度队列可能以一种不活跃的状态被创建，在这种状态下的队列，队列中blocks 被调用之前，队列必须被激活。调用dispatch_activate函数是队列活跃。）
> - DISPATCH_QUEUE_CONCURRENT，并发队列
> - DISPATCH_QUEUE_CONCURRENT_INACTIVE，并发队列，不活跃的



#### 在运行时获取常见队列

GCD提供函数允许您从应用程序中访问几个常见的调度队列：

- 使用`dispatch_get_current_queue`函数调试或者测试当前队列的标示。在块对象内部调用这个函数，返回块被提交到的队列（并且现在可能正在运行）。在块外部调用此函数，将返回应用程序的默认并发队列。
- 使用`dispatch_get_main_queue`函数获取关联到应用程序主线程的串行调度队列。对于Cocoa应用程序和调用`dispatch_main`函数或在主线程上配置RunLoop（使用`CFRunLoopRef`类型或者`NSRunLoop`对象）的应用程序，此队列自动被创建。
- 使用`dispatch_get_global_queue`函数获取任意共享的并发队列。更多信息，请参阅[获得全局并发调度队列](#获得全局并发调度队列)。



#### 调度队列的内存管理

调度队列和其他调度对象是引用计数的数据类型。当创建串行调度队列时，他初始引用计数为1。可以根据需要使用`dispatch_retain`和`dispatch_release`函数来增加和减少引用计数。当队列的引用计数为0时，系统异步的释放（dealloc）队列。

保留（retain）和释放（release）调度对象（如队列）非常重要，以确保它们在被使用时保留在内存中。与Cocoa对象的内存管理一样，基本规则是，如果您打算使用传递给您代码的队列，在使用之前应当保留队列，在不再需要时释放队列。这个基本模式确保只要您使用队列，它就在内存中。

> **重要提示：**您不需要保留或释放任何全局调度队列，包括并发调度队列或主调度队列。任何保留和释放这些队列的试图都将被忽略。

即使您实现一个垃圾回收的应用程序，您也必须保留和释放您的调度队列和其他调度对象。GCD不支持用于回收内存的垃圾回收模型。



#### 使用队列存储自定义上下文信息

所有调度对象（包括调度队列）允许您将自定义上下文数据与调度对象关联。要在调度对象上设置和获取这些数据，可以使用`dispatch_set_context`和`dispatch_get_context`函数。系统不会以任何方式使用您的自定义数据，并且由您决定在适当的时候分配和释放数据。

对于队列，您可以使用上下文数据存储指向Objective-C对象的指针或者其他数据结构，用来帮助标示队列或者对代码的预期用途。您可以在队列释放之前使用队列的finalizer（终结器/清理器）函数将上下文数据从队列中释放（或者取消关联）。有关如何写finalizer函数来清理队列的上下文数据，请参阅[为队列提供清理功能](#为队列提供清理功能)。



#### 为队列提供清理功能

创建串行调度队列后，您可以附加一个finalizer函数，当队列释放时执行任意自定义清理。调度队列是引用计数对象，您可以使用`dispatch_set_finalizer_f`函数来指定一个函数，当队列的引用计数为0时执行。可以使用这个函数来清理关联到队列的上下文数据，只要上下文指针不为NULL，这个函数就被调用。

下面代码展示了一个自定义finalizer函数和一个创建队列并设置finalizer的函数。队列使用finalizer函数释放存储在队列上下文指针中的数据。（代码中的`myInitializeDataContextFunction`和`myCleanUpDataContextFunction`函数是自定义函数，提供初始化和清理数据结构内容功能。）传递给finalizer函数的上下文指针包含关联到队列的数据对象。

```objc
void myFinalizerFunction(void *context)
{
    MyDataContext* theData = (MyDataContext*)context;
 
    // Clean up the contents of the structure
    myCleanUpDataContextFunction(theData);
 
    // Now release the structure itself.
    free(theData);
}
 
dispatch_queue_t createMyQueue()
{
    MyDataContext*  data = (MyDataContext*) malloc(sizeof(MyDataContext));
    myInitializeDataContextFunction(data);
 
    // Create the queue and set the context data.
    dispatch_queue_t serialQueue = dispatch_queue_create("com.example.CriticalTaskQueue", NULL);
    if (serialQueue)
    {
        dispatch_set_context(serialQueue, data);
        dispatch_set_finalizer_f(serialQueue, &myFinalizerFunction);
    }
 
    return serialQueue;
}
```



### 添加任务到队列

为了执行任务，必须将其分配到适当的调度队列。可以同步或者异步调度任务，而且可以逐一或分组的调度它们。一旦进入队列，队列负责尽快执行任务，考虑它们的约束和队列中已经存在的任务。下面介绍一些将任务分配到队列的技术和它们的优点。



#### 添加单个任务到队列

有两种方式添加任务到队列：异步和同步。如果有可能的话，使用`dispatch_async`和`dispatch_async_f`函数异步执行优先于同步执行。当添加一个块对象或函数到队列，没有办法知道代码什么时候执行。因此，异步添加块或函数允许您调度代码执行并且继续在调用线程中做其他工作。如果从应用程序的主线程调度任务（可能响应一些用户事件），这一点尤其重要。

虽然应该尽可能异步添加任务，但可能仍需要同步添加任务以防止竞争条件或者其他同步错误。在这些情况下，可以使用`dispatch_sync`和`dispatch_sync_f`函数添加任务到队列。这些函数阻塞当前线程执行，直到指定的任务执行结束。

> **重要提示：**永远不要从队列中执行的任务里调用`dispatch_sync`或`dispatch_sync_f`函数 ，且传递给函数同一个队列。这对的串行队列非常重要，它产生了死锁，对于并发队列也应该避免。

下面示例展示如何使用基于块的变量异步和同步调度任务：

```objc
dispatch_queue_t myCustomQueue;
myCustomQueue = dispatch_queue_create("com.example.MyCustomQueue", NULL);
 
dispatch_async(myCustomQueue, ^{
    printf("Do some work here.\n");
});
 
printf("The first block may or may not have run.\n");
 
dispatch_sync(myCustomQueue, ^{
    printf("Do some more work here.\n");
});
printf("Both blocks have completed.\n");
```



#### 任务完成时执行完成块

根据其性质，被调度到队列的任务独立于创建他们的代码运行。然而，当任务完成时，应用程序可能希望被通知该情况，以便它可以合并结果。对于传统异步编程，可能使用回调机制来做，但对于调度队列，可以使用完成块。

完成块是在原始任务结束时调度到队列的另外一段代码。当任务开始时，调用代码通常提供完成块作为参数。任务代码需要做的是，当它结束时，提交指定块或者函数到指定队列。

下面代码展示一个使用块实现求平均值的函数。函数的最后两个参数允许调用者指定队列和当汇报结果时用的块。求平均值函数计算其结果后，传递结果到指定的块并调度块到队列。为了防止队列过早的被释放，在最开始保留队列并且在完成块被调度后释放队列是至关重要的。

```objc
void average_async(int *data, size_t len, dispatch_queue_t queue, void (^block)(int)) {
   // Retain the queue provided by the user to make
   // sure it does not disappear before the completion
   // block can be called.
   dispatch_retain(queue);
 
   // Do the work on the default concurrent queue and then
   // call the user-provided block with the results.
   dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
      int avg = average(data, len);
      dispatch_async(queue, ^{ block(avg);});
 
      // Release the user-provided queue when done
      dispatch_release(queue);
   });
}
```



#### 并发执行循环迭代

在循环执行固定数量迭代的地方，并发调度队列可能提高其性能。例如，假设有个for循环，通过每个循环迭代做一些工作：

```objc
for (i = 0; i < count; i++) {
   printf("%u\n",i);
}
```

如果每个迭代中执行的工作与所有其他迭代中执行的工作不同，且循环完成的顺序不重要。可以使用调用`dispatch_apply`或`dispatch_apply_f`函数来替换循环。这个函数为每次循环迭代提交指定块或函数到队列。当被调度到并发队列时，因此可以同时执行多个循环迭代。

当调用`dispatch_apply`或`dispatch_apply_f`函数时，可以指定一个串行队列或者并发队列。传入并发队列允许您同时执行多个循环迭代，是使用这个函数最常见的方法。虽然使用串行队列是允许的，并为您的代码做正确的事情，但使用这样的队列代替循环并没有真正的性能优势。

> **重要提示：**像普通循环一样，`dispatch_apply`或`dispatch_apply_f`函数不返回，直到所有循环迭代结束。因此，当从队列上下文已经执行的代码中调用它们时，应当小心。如果作为参数传递给函数的队列是串行队列，且与执行当前代码的队列是同一个队列，调用这个方法将会使队列死锁。
>
> 因为它们直接阻塞当前线程，所以当从主线程调用这些函数时也应当小心，它们可能阻止事件处理循环及时响应事件。如果您的代码需要大量的处理时间，您可能需要从不同的线程调用这些函数。

下面代码显示如何使用`dispath_apply`语法替换前面描述的for循环。传入到`dispath_apply`函数的块必须包含一个标示当前循环迭代的参数。当块执行时，第一次迭代参数为0，第二次为1，等等。最后一次迭代，参数的值为count - 1，count代表迭代的总次数。

```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
 
dispatch_apply(count, queue, ^(size_t i) {
   printf("%u\n",i);
});
```

您应当确保每次迭代的代码做合理数量的工作。和任何块或函数调度到队列一样，调度代码执行有开销。如果每次循环迭代只执行很少量的工作，调度代码的开销可能超过调度到队列带来的性能优势。如果在测试过程中发现这是真的，您可以使用跨步来增加每次循环迭代执行的工作量。随着跨步，将原来循环的多次迭代组成一个单独的块，减少迭代次数的比例。例如，如果最初执行100次迭代，但决定使用步幅4，现在每个块执行4次循环迭代，迭代次数是25。有关如何实现跨步，请参阅[完善循环代码](https://developer.apple.com/library/prerelease/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW2)



#### 在主线程中执行任务

GCD提供特殊的串行调度队列，可以使用它在应用程序的主线程上执行任务。这个队列被自动的提供给所有应用程序，并由在主线程上设置的运行循环（被`CFRunLoopRef`类型或者`NSRunLoop`对象管理）**drain**（*官方文档上写的drain，翻译成销毁感觉不太恰当，因为drain的意思是"使...流尽","耗尽"的意思，可以理解为使队列中的任务全部出队，NSAutoreleasePool也有drain方法，意思相近*）。如果您创建的不是Cocoa应用程序，不要想着显式的设置RunLoop，您必须显式的调用`dispatch_main`函数来drain主调度队列。您仍然可以添加任务到队列，但是如果您不调用此方法，这些任务永远不会执行。

可以通过调用`dispatch_get_main_queue`函数获取应用程序主线程的调度队列。添加到这个队列的任务在主线程上被串行执行。因此，可以使用这个队列作为应用程序其他部分工作执行完成的同步点。



#### 在任务中使用Objective-C对象

GCD提供内置的支持Cocoa内存管理技术，所以，您可以自由的在提交到队列的块中使用Objective-C对象。每个调度队列维护自己的自动释放池来确保自动释放对象在一些点被释放；队列不保证这些对象何时真正释放。

如果您的应用程序内存不足，且您的块创建超过几个自动释放对象，创建自己的自动释放池是唯一的方法来确保您的对象被及时释放。如果您的块创建上百个对象，您可能希望创建多个自动释放池或定期drain自动释放池。

关于自动释放池和Objective-C内存管理的更多信息，请参阅[高级内存管理编程指南](https://developer.apple.com/library/prerelease/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011i)



### 暂停和恢复队列

您可以通过暂停队列来临时阻止队列执行块对象。使用`dispatch_suspend`函数来暂停调度队列，使用`dispatch_resume`函数来恢复调度队列。调用`dispatch_suspend`增加队列的暂停引用计数，调用`dispatch_resume`减少引用计数。当引用计数大于0时，队列保持挂起。因此，为了恢复处理块，您必须使用一个配对的恢复调用平衡所有暂停调用。

> **重要提示：**暂停和恢复调用是异步的，仅在执行块之间生效。暂停一个队列不会导致已经执行的块停止。



### 使用调度信号量来调节有限资源的使用

如果提交到调度队列的任务访问一些有限的资源，您可能需要使用调度信号来调节同时访问资源的任务数量。调度信号像普通信号一样工作，但有一个例外。当资源可用时，它获取调度信号量消耗的时间比获取传统系统信号量消耗的时间少。这是因为GCD在这种特殊情况下不调用内核。只有当资源不可用且系统需要停驻线程直到向信号量发出信号时才调用内核。

使用信号量语义如下：

1. 当创建信号量时（使用`dispatch_semaphore_create`函数），您可以指定一个正数，表示可用资源的数量。
2. 在每个任务中，调用`dispatch_semaphore_wait`函数等待信号。
3. 当等待返回时，获取资源，执行工作。
4. 当资源使用完毕时，释放资源并调用`dispatch_semaphore_signal`函数向信号量发出信号。

有关这些步骤如何工作，例如，考虑在系统上使用文件描述符，每个应用程序被给予有限数量的文件描述符来使用。如果您有一个处理大量文件的任务，您不想一次打开这么多的文件，这样会耗尽文件描述符。您可以在文件处理代码中使用信号量限制任何时候文件描述符一次使用的数量。可能在您任务中添加的代码基本片段如下：

```objc
// Create the semaphore, specifying the initial pool size
dispatch_semaphore_t fd_sema = dispatch_semaphore_create(getdtablesize() / 2);
 
// Wait for a free file descriptor
dispatch_semaphore_wait(fd_sema, DISPATCH_TIME_FOREVER);
fd = open("/etc/services", O_RDONLY);
 
// Release the file descriptor when done
close(fd);
dispatch_semaphore_signal(fd_sema);
```

当创建一个信号量时，指定可用资源数量。这个值将成为信号量计数的初始值。每次等待信号，`dispatch_semaphore_wait`函数将计数变量减1。如果结果值为负数，函数告诉内核阻塞线程。另外一边，`dispatch_semaphore_signal`函数将计数变量增加1，指示资源已经被释放。如果有被阻塞且等待资源的任务，他们其中的一个随后变为非阻塞并允许工作。



### 等待排队任务组

调度组是阻塞线程直到一个或者多个任务结束执行的方法。您可以在不能够获取进度直到所有指定任务结束的地方使用这种行为。例如，调度几个任务来计算一些数据，您可能使用一个组来等待这些任务，然后当它们结束时处理结果。使用调度组的另外一种方法是替代线程连接。您可能添加相应的任务到调度组且等待整个组，而不是开启几个子线程然后连接它们。

下面代码显示创建一个组，调度任务给它，并等待结果。使用`dispatch_group_async`函数，而不是使用`dispatch_async`函数调度任务到队列。这个函数关联任务到组，将它们排队执行。为了等待任务组结束，稍后使用`dispatch_group_wait`函数，传递相应的组进去。

```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
 
// Add a task to the group
dispatch_group_async(group, queue, ^{
   // Some asynchronous work
});
 
// Do some other work while the tasks execute.
 
// When you cannot make any more forward progress,
// wait on the group to block the current thread.
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
 
// Release the group when it is no longer needed.
dispatch_release(group);
```

> **注意：**可以使用`dispatch_group_notify`或者`dispatch_group_notify_f`函数来通知关联到组的调度队列执行完毕。也就是说当调度到队列的块都执行完毕的时候，会执行`dispatch_group_notify`或者`dispatch_group_notify_f`函数。



### 调度队列和线程安全

在调度队列的内容中讨论线程安全可能看起来很奇怪，但线程安全仍然是相关联的话题。任何时候在应用程序中实现并发，有几件事情都应该知道：

- 调度队列自身是线程安全的。换句话说，您可以从系统的任何线程提交任务到调度队列，而不用先使用锁或者同步访问队列。
- 不要从队列中执行的任务里调用`dispatch_sync`函数 ，且传递给函数同一个队列。这么做会导致队列死锁。如果您需要调度到当前队列，异步使用`dispatch_async`函数。
- 避免在提交给调度队列的任务中使用锁。虽然在任务中使用锁是安全的，当您获取锁时，如果锁不可用，可能阻塞整个串行队列。相同的，对于并发队列，等待锁可能阻止其他线程执行。如果您需要同步部分代码，使用串行调度队列代替锁。
- 虽然您可以获取关于底层线程运行任务的信息，最好避免这么做。有关调度队列和线程兼容性的更多信息，请参阅[POSIX线程的兼容性](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW18)

有关如何更改现有线程代码到使用调度队列的更多提示，请参阅[线程迁移](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW1)。



参考：

[https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW1](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW1)