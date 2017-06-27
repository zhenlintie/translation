[什么是函数式反应型编程？](#什么是函数式反应型编程)

# ReactiveCocoa vs RxSwift

原文：[_Rui Peres on April 26, 2016_](https://www.raywenderlich.com/126522/reactivecocoa-vs-rxswift)

![RxSwift vs ReactiveCocoa](https://koenig-media.raywenderlich.com/uploads/2016/04/RxSwiftReactiveCocoa-feature.png)

函数式反应型编程（Functional Reactive Programming）是一种变得越来越流行的编程范式，尤其是在 Swift 开发者之中。它将复杂的异步过程，变得容易编写和理解。 
 
在这篇文章里，你将可以对比函数式反应型编程中最流行的两个框架：RxSwift 和 ReactiveCocoa。  

下面来简单的了解一下，什么是函数式反应型编程？然后详细的比较一下这两个框架，了解了这些之后，你就可以选择一个适合你的框架来使用。

那么开始吧！


什么是函数式反应型编程？
==============

> 如果你已经熟悉了函数式反应型编程的概念，可以跳过这一章节，直接阅读下一章节 [**ReactiveCocoa vs RxSwift**](#RACvsRxS)。

甚至在 Swift 出现之前，函数式反应型编程（FRP）在近些年来欢受迎程度就大幅增长，与面向对象编程形成鲜明对比。从 Haskell 到 Go，再到 Javascript，都有 FRP 的实现方式。为什么呢？FRP 到底有什么特异功能？
最重要的问题是，你如何将这种编程范式应用到 Swift 上呢？

函数式反应型编程是由 [Conal Elliott](https://twitter.com/conal) 创建的一种编程范式。他给了一个严谨详细的语义定义，你可以从[这里](https://stackoverflow.com/questions/1028250/what-is-functional-reactive-programming)去进一步了解。简单定义的话，FRP 是由两种概念组合而成的：  

1. **反应型编程（Reactive Programming）**，它关注的是异步数据流，你需要监听并根据其中的数据做出反应。想要了解更多，看看这篇[不错的介绍](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)。
2. **函数式编程（Functional Programming）**，他的函数定义具有数学风格，计算过程中尽量避免使用变量和状态值，代码更加灵活无副作用。想要了解更多，请看我们的另一篇 ["Swift functional programming tutorial"](https://www.raywenderlich.com/82599/swift-functional-programming-tutorial)。  

>[André Staltz](https://twitter.com/andrestaltz) 在他的文章 ["Why I cannot say FRP but I just did"](https://medium.com/@andrestaltz/why-i-cannot-say-frp-but-i-just-did-d5ffaa23973b#.62dnhk32p) 中，阐述了标准的 FRP 范式和它的可实现方式之间的差别。  

### 一个简单的例子

想要理解这个概念最简单的方式，就是通过例子来说明。想象有一款应用，需要关注用户的位置变化，并且在发现他的位置靠近一个咖啡店的时候提示他。

通过 FRP 的方式需要这样实现：  

1. 需要创建一个对象，它发出你需要响应的位置变化事件的数据流。
2. 然后通过筛选这些位置信息，把那些靠近咖啡店的位置信息显示出来。

在 ReactiveCocoa 中，代码大概长这样：

```
locationProducer // 1  
  .filter(ifLocationNearCoffeeShops) // 2  
  .startWithNext {[weak self] location in // 3  
    self?.alertUser(location)
}
```

说明一下这部分代码：

1. `locationProducer` 在位置信息每次发生变化的时候，都会抛出一个事件（event）。ReactiveCocoa 把它称作 `signal`，RxSwift 中称作 `sequence`。
2. 然后利用函数式编程（Functional Programming）技术去处理这些变化事件的数据。`filter` 函数的用法和数组（array）中的相同，将数据流中的每个值作为参数传给 `ifLocationNearCoffeeShops`处理，如果返回 `true`，这个事件会继续传递到下一步。
3. 最后，`startWithNext` 就是一个订阅方法，当有（过滤过的）事件传递到这里的时候，你传入的闭包表达式将会执行，并将那些数据作为参数提供给你。

上面的代码看起来很像是将数组中的数据进行转换。但是这个更高级...它是异步的；那个过滤方法和闭包表达式只有在位置变化事件发生的时候才会被执行。

语法看起来也许会怪一些，不过希望你可以明白这段代码所表达的意思。它既体现了函数式编程的精髓，又十分贴切 `values over time（直译：时间轴上的数据）`的概念：这也就是它的定义。你需要关心的是_数据来了_，而不是_产生这些数据的细节_。

>如果你想了解更多的 ReactiveCocoa 语法，去看看我写的一些例子吧：[Github 地址](https://github.com/RACCommunity/RACNest)。  


### 事件转换

在上面的例子中，仅仅是**关注**了位置变化的数据流，除了过滤一些靠近咖啡店的位置，并没有对这些事件做更多的处理。  
而 FRP 范式的另一个基本要素，就是能够将这些事件数据进行组合、转换，使其变得更有意义一些。怎么做呢？你可以利用（但不限于）一些高阶函数。  
你可以在我们的教程 [Swift functional programming tutorial](https://www.raywenderlich.com/82599/swift-functional-programming-tutorial) 中，找到一些常见的函数：`map`、`filter`、`reduce`、`combine`、`zip`。  
我们来优化一下这段代码，过滤掉那些重复的位置信息，并且将传过来的位置数据（`CLLocation`）转换成一段用户可识别的文本信息。

```
locationProducer
  .skipRepeats() // 1
  .filter(ifLocationNearCoffeeShops) 
  .map(toHumanReadableLocation) // 2
  .startWithNext {[weak self] readableLocation in
    self?.alertUser(readableLocation)
}
```

我们来看看新加的两行代码：  
1. 首先在 `locationProducer` 发出数据流之后，加一步 `skipRepeats` 操作，这个操作并不是 `array` 所具有的；它是 ReactiveCocoa 所特有的。这个方法的意图很明显：过滤掉那些相等的数据事件（这些事件的数据需要具有可比性）。  
2. 在 `filter` 方法执行过后，`map` 函数的作用就是将一种事件数据转换成另一种，比如把 `CLLocation` 类型转换成 `String` 类型。  

现在，你已经体会到了 FRP 的一些非凡之处了吧：  

- 它使用简单，却功能强大。
- 它使代码更加容易被理解。
- 复杂的数据流，变得容易管理和描述。  

<img src="https://koenig-media.raywenderlich.com/uploads/2016/02/Screen-Shot-2016-02-15-at-22.03.12.png" width=80%/>

## <p id="RACvsRxS"> ReactiveCocoa 和 RxSwift 简述 </p>

现在你已经对什么是 FRP 有了一个比较好的认识，并且知道了它如何帮助你简单的管理复杂的异步数据流。接下来让我们来看看这两个流行的 FRP 框架：ReactiveCocoa 和 RxSwift，然后你可以挑一个适合你的来使用。  
详细分析之前，我们先简单了解一下这两个框架的发展史。

### ReactiveCocoa

[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) 是在 Github 上发布的。当时开发者们在 Github Mac 客户端上工作，他们发现很难管理他们应用的数据流。后来他们从微软的 [ReactiveExtensions](https://msdn.microsoft.com/en-gb/data/gg577609.aspx?f=255&MSPPError=-2147217396)（一个 C# 的 FRP 实现）那里得到灵感，创建了他们的 Objective-C 实现。

当他们正准备发布 Objective-C 实现的 3.0 版本的时候，Swift 发布了。他们意识到 Swift 的函数式风格更适合 ReactiveCocoa，所以[他们马上着手于 Swift 的实现](https://github.com/ReactiveCocoa/ReactiveCocoa/pull/1382)，成为了 [3.0 版本](https://github.com/ReactiveCocoa/ReactiveCocoa/tree/v3.0.0)。3.0 版本更具函数式风格，利用到了柯里化（currying）和 pipe-forward 运算符技术。  

Swift 2.0 引入了[面向协议编程（protocol-oriented programming](http://www.raywenderlich.com/109156/introducing-protocol-oriented-programming-in-swift-2) 概念，这也导致了 ReactiveCocoa API 另一个具有重大意义的变化，4.0 版本减少使用 pipe-forward 运算符，而开始运用协议拓展。  

写这篇文章时，ReactiveCocoa 已经是在 Github 上已经收获了超过 13000 颗星的非常流行的库了。

### RxSwift

微软的 ReactiveExtensions 启发许多其他的框架，将 FRP 的概念融入了 JavaScript，Java，Scala，还有许多其他的编程语言。最终形成了 [ReactiveX](http://reactivex.io/)，一个为 FRP 实现方式创建通用 API 的组织；这允许许多框架作者可以协同工作。也正因为这样，一个熟悉 RxScal（Scala 的实现）的开发者会发现把它转换成 Java 类似的实现（RxJava）将会很容易。  

[RxSwift](https://github.com/ReactiveX/RxSwift) 是相对较新加入 ReactiveX 的，而且不如 ReactiveCocoa 更加流行（写这段话时，它在 Github 上大概有 4000 颗星）。不过事实上，RxSwift 作为 ReactiveX 的一部分，毫无疑问将会很流行并长久发展下去。  

值得一提的是，RxSwift 与 ReactiveCocoa 都有一个共同的原型：ReactiveExtensions。

## RxSwift 与 ReactiveCocoa 对比

是时候该了解一下细节了。RxSwift 与 ReactiveCocoa 对于 FRP 的支持体现了许多不同的方面，让我们来看看他们其中重要的几部分。

### 热信号、冷信号

想象一下，你需要发起一个网络请求，并且解析它的响应数据，然后展示给用户：

```
let requestFlow = networkRequest.flatMap(parseResponse)
 
requestFlow.startWithNext {[weak self] result in
  self?.showResult(result)
}
```

只有当你订阅一个 `signal`（也就是进行 `startWithNext` 操作）的时候，网络请求才会被创建并发起。这种 `signal` 被称作是**冷（cold）**的，因为正如你所猜到的，直到你订阅它们之前，它是处于“冻结”状态的。  

另一种则称为**热（hot）** 信号，当订阅一个热信号时，它就已经开始了，所以你可以观察到第三或者第四次事件，或者更多。比较典型的例子就是敲击键盘产生的事件流。对于“开始”敲击键盘，并没有什么意义，就像创建一个服务器请求。

总结一下：

* 一个**冷信号**是指，当你想要订阅他的时候，需要执行开始任务，每个新的订阅者都需要执行开始任务，订阅 **`requestFlow`** 三次也就意味着相对应的要创建三个网路请求。  
* 一个**热信号**创建时就已经可以发送事件了，订阅者不需要去开启它。通常 UI 交互是属于热信号。

ReactiveCocoa 针对热、冷信号分别提供了这两种类型：`Signal<T,E>` 与 `SignalProducer<T,E>`。而 RxSwift 提供了一种同时支持冷、热信号的类型：`Observable<T>`。

区分热信号、冷信号两种不同的类型真的有必要么？  

我个人认为，知道一个信号的含义是很重要的，因为它更好的描述了如何在一个特定的上下文中运用它。当处理一个复杂的系统时，这些将会有很大不同。  

且不说有没有这两种类型的支持，仅仅了解热信号、冷信号这些概念就非常重要。  
正如 [André Staltz](https://twitter.com/andrestaltz) 所说的：  

> 如果你忽略了它，那么它一定会回来给你狠狠的一击。别说我没告诉过你。

假设你正在处理一个**热信号**，然后由于某种原因它变成了冷信号，这个时候你将会对每个订阅者进行副作用编程。这将会给你的应用带来很大影响。举个通用的例子，假设在你的应用中，有三个或四个实体需要监听同一种网络请求，而对于每个新的订阅，都会发起一个新的网络请求。

ReactiveCocoa 加一分！

<img src="https://koenig-media.raywenderlich.com/uploads/2016/02/Screen-Shot-2016-02-15-at-23.12.28.png" width=80%/>

Test
=======

### 错误处理

讨论错误处理之前，我们先概括一下 RxSwift 和 ReactiveCocoa 分发的事件的一些性质。这两个框架中，主要有三种事件类型：

1. `Next<T>`：每当一个新的值（`T` 类型）被传到事件流中时，这种事件就会被触发。在上面跟踪定位的那个例子中，`T` 指的就是 `CLLocation`。
2. `Compleled`：表示事件流的终止。收到这个事件之后，将不会在发送 `Next<T>` 和 `Error<E>`。
3. `Error<E>`：表示一个错误。在服务器请求的例子中，当你收到一个服务器错误时，这个事件将会被发送。`E` 是遵循了 `ErrorType` 协议的**错误类型**。收到这个事件之后，将不会在发送 `Next<T>` 和 `Compleled`。

你应该已经注意到上一章节提到 ReactiveCocoa 的 `Singal<T, E>` 和 `SignalProducer<T, E>` 有两个参数类型，而 RxSwift 的 `Observable<T>` 只有一个。前者的第二个类型（`E`）是遵循了 `ErrorType` 协议的子类型。在 RxSwift 中这个类型被删除了，取而代之的是一个需要内部处理的 `ErrorType` 协议类型。

所以这些是什么意思呢？

实际上，这意味着在 RxSwift 中，错误可以从许多不同的地方抛出来。

```
create { observer in
  observer.onError(NSError.init(domain: "NetworkServer", code: 1, userInfo: nil))
}
```

以上创建了一个信号（或者，RxSwift 中称之为**观察序列（observable sequence）**），然后立刻抛出了一个错误。

这是另一个：

```
create { observer in
  observer.onError(MyDomainSpecificError.NetworkServer)
}
```

一个 `Observable` 强制**错误**必须只能是遵从 `ErrorType` 类型的，你可以发送任何你需要的错误类型。但是可能有些不方便，例如下面的例子：

```
enum MyDomanSpecificError: ErrorType {
  case NetworkServer
  case Parser
  case Persistence
}
 
func handleError(error: MyDomanSpecificError) {
  // Show alert with the error
}
 
observable.subscribeError {[weak self] error in
  self?.handleError(error)
 }
```

这段代码无效，因为函数 `handleError` 希望得到的是 `MyDomainSpecificError` 类型，而不是 `ErrorType` 类型。所以你必须做两件事：

1. 试着将 `error` 转换成 `MyDomainSpecificError` 类型。
2. 处理当 `error` 不能转成 `MyDomainSpecificError` 的情况。

第一点很容易利用 `as?` 来解决，但是第二点不太容易确定如何处理。一个可能的解决方案就是提供一个 `Unknown` 类型：

```
enum MyDomanSpecificError: ErrorType {
  case NetworkServer
  case Parser
  case Persistence
  case Unknown
}
 
observable.subscribeError {[weak self] error in
  self?.handleError(error as? MyDomanSpecificError ?? .Unknown)
}
```

在 ReactiveCocoa 中，当你创建一个 `Signal<T,E>` 或者 `SignalProducer<T,E>` 时，就相当于解决了上述的第一点，如果你想要传递其他类型，编译器会给警告。最后：ReactiveCocoa 中，编译器不允许你传一个不同于你之前指定好的错误类型。

ReactiveCocoa 再加一分！

### UI 绑定

在 iOS 的标准 API 中，比如 `UIKit`，并不是使用的 FRP 的语言语法。所以为了使用 RxSwift 和 ReactiveCocoa，你必须桥接这些 API，例如将点击事件（利用 `target-action` 编码形式）转换成 `signal` 或 `observable`。

正如你所想象的，这需要许多工作，所以这两个库都额外提供了许多桥和绑定。

ReactiveCocoa 带来了许多从 Objective-C 时期完成的工作。你会发现[有一部分已经完成了](https://github.com/ReactiveCocoa/ReactiveCocoa/tree/master/ReactiveCocoa/Objective-C)，并且已经桥接到了 Swift 上。这些只包括 UI 绑定，其他操作还没有没翻译成 Swift。所以，显得有些奇怪。你正在使用一个不是 Swift API 中的类型（比如 `RACSignal`），然后又强制用户将 Objective-C 类型转为 Swift（例如使用 `toSignalProducer()` 方法）。

不止这些，我觉得我看源码的时间比看文档的时间都长，这显然跟不上时代。注意这点是很重要的，虽然从解释[理论和思路](https://github.com/ReactiveCocoa/ReactiveCocoa/tree/master/Documentation)上来讲，文档写的很好，但从使用角度上来说还远远不够。

正是由于这点，你会发现有许多的 ReactiveCocoa 教程。

与此不同的是，RxSwift 提供的绑定很容易使用！你不仅能看到[巨长的目录](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/API.md)，还能找到巨多的[例子](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Examples.md)，还有一份[更加完善的文档](https://github.com/ReactiveX/RxSwift/tree/master/Documentation)。对于一些人来说，这已经足够让你去选择 RxSwift，而不选 ReactiveCocoa 了。

RxSwift 加一分！

<img src="https://koenig-media.raywenderlich.com/uploads/2016/02/Happy-Crying-Face-Meme-11.png" width=60%/>

### 社团

ReactiveCocoa 比起 RxSwift 很久之前就已经出现了。有许多人还在继续维护，在网上也有很多教程，而且通过在 [StackOverflow 的 ReactiveCocoa 标签](https://stackoverflow.com/questions/tagged/reactive-cocoa)可以找到很好的资源。

ReactiveCocoa 有一个 Slack 的群，不过很小只有 209 个人，所以有许多人提的问题（包括我提的和其他人提的）都没有回答。由于时间紧急，我被迫私信了 ReactiveCocoa 的核心成员，所以我猜测其他人也可能和我的情况的一样。不过，你很有可能在网上找一个教程来解释你的问题。

RxSwift 是新人，不过现在却很有[一枝独秀](https://github.com/ReactiveX/RxSwift/graphs/contributors)的感觉。他也有一个 Slack 的群，而且很大已经有 961 个成员了，群内讨论热烈。你也可以比较容易在这里找到回答你问题的人。

总之吧，现在两个框架的社团支持以各自不同的强大方式支持着，所以这个小节，打成平手。

### 你该如何选择？

正如 Ash Furrow 在 “ReactiveCocoa vs RxSwift” 中所说的：

>“听我的，如果你是一个新手，选哪个真的没有关系。是的，虽然他们有些技术上的区别，不过对于新手没太大意义。试着用其中一个，然后再试试另一个。看看哪个更适合你，然后在想想为什么你会选择它。”

我的意见也大概如此。只有当你有足够的体验之后，才能领会他们之中细微的差别。

不过，如果你现在正处于一个需要选择其中一个，但是又没时间去使用它们的情况，可以看看我的意见：

#### 选择 ReactiveCocoa，如果：

* 你想要更好的描述你的系统。用不同的类型来区分热信号和冷信号，同时通过类型化参数处理错误，对于你的系统会有很好的效果。
* 想要一个大规模的测试框架，被许多人使用在不同的项目中。

#### 选择 RxSwift，如果：

* UI 绑定对于你的系统很重要
* 你是一个 FRP 新手，希望得到一些手把手的教学。
* 你已经了解了 RxJS 或者 RxJava。因为它们和 RxSwift 都是属于 ReactiveX 组织的，如果你了解其中之一，其他的也只是语法上的不同而已。

## 何去何从?

无论你选择了 ReactiveCocoa 或者选择 RxSwift，你都不会后悔的。它们都是很强大的框架，并且可以帮助你很好的描述你的系统。

需要注意的很重要的一点是，一旦你选择 RxSwift 或 ReactiveCocoa 其中之一，想要切换到另一方也只是几个小时的问题。就我以锻炼的目的，从 ReactiveCocoa 转到 RxSwift 的经验来说，大部分麻烦的问题也就是错误处理了。总结来说，最大的思想转变就是完全使用 FRP，而不是仅仅实现其中一个部分。

以下的链接可以帮助你融入函数式响应性编程、RxSwift 和 ReactiveCocoa 中：

* Conal Elliott 的[博客](http://conal.net/blog/)。
* Conal Elliott 在 Stackoverflow 对于 [“What is (functional) reactive programming?”](https://stackoverflow.com/questions/1028250/what-is-functional-reactive-programming) 强大的回答。
* [André Staltz](https://twitter.com/andrestaltz)，必须读的文章 [“Why I cannot say FRP but I just did”](https://medium.com/@andrestaltz/why-i-cannot-say-frp-but-i-just-did-d5ffaa23973b#.62dnhk32p)。
* [RxSwift 的 Github 地址](https://github.com/ReactiveX/RxSwift)。
* [ReactiveCocoa 的 Github 地址](https://github.com/ReactiveCocoa/ReactiveCocoa)。
* [Rex 的 Github 地址](https://github.com/neilpa/Rex)。
* iOS 开发者最终的 [FRP 宝库](https://gist.github.com/JaviLorbada/4a7bd6129275ebefd5a6)。这里你能够找到包括 RxSwift 和 ReactiveCocoa 两者的资源。
* 我们 [Marin Todorov](https://twitter.com/icanzilb) 的 [RxSwift 探索](http://rx-marin.com/)。

