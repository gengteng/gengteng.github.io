---
title: 零成本异步IO
date: 2021-01-30 09:13:54
tags: [Rust]
---
> 这是 Withoutboats 在 2019 年 3 月的 Rust Latam 上所做报告的一个整理。这个报告主要介绍他参与开发了一年半的语言特性，包括 Rust 异步 I/O 的发展历程，以及目前已经稳定的零成本抽象的`async`/`await` 语法的关键实现原理。

> Withoutboats 是就职于 Mozilla 的一名研究员，主要从事 Rust 语言开发。他开发的这个语言特性叫做 `async`/`await`，这可能是本年度我们在 Rust 语言上做的最重要的事。这解决了困扰我们很久的问题，即我们如何能在 Rust 中拥有零成本抽象的异步IO。

> 注：因个人水平有限，翻译和整理难免有错误或疏漏之处，欢迎读者批评指正。

## `async`/`await`

首先，介绍一下 `async`/`await`。

`async` 是一个修饰符，它可以应用在函数上，这种函数不会在调用时一句句运行完成，而是立即返回一个 `Future` 对象，这个 `Future` 对象最终将给出这个函数的实际返回结果。而在一个这样的 `async` 函数中，我们可以使用await运算符，将它用在其它会返回 `Future` 的函数上，直到那些 `Future` 返回实际结果。通过这种方法，异步并发开发更加方便了。

``` rust
let user = await db.get_user("withoutboats");

impl Database {
    async fn get_user(&mut self, user: &str) -> User {
        let sql = format!("select FROM users WHERE username = {}", user);
        let db_response = await self.query(&sql);
        User::from(db_response)
    }
}
```

这是一段简短的代码样例，我们具体解释一下 `Future` 。这段代码基本上做的就是一种类似于 ORM 框架所作的事。你有一个叫 `get_user` 的函数，它接受一个字符串类型的用户名参数，并通过在数据库中查找对应用户的记录来返回一个User对象。它使用的是异步 I/O ，这意味着它得是一个异步函数，而不是普通函数，因此当你调用它时，你可以异步等待（`await`）它；然后我们看一下函数的实现，首先是用用户名参数拼接出要执行的 SQL 语句，然后是查询数据库，这就是我们实际执行 I/O 的地方，所以这个查询（query）返回的是 `Future` ，因为它使用的是异步 I/O 。所以在查询数据库时，你只需要使用异步等待（`await`）来等待响应，在获得响应后就可以从中解析出用户。这个函数看起来像个玩具，但我想强调的是，它与使用阻塞式 I/O 的唯一区别就是这些注解（指`async`/`await`）了，你只需将函数标记为异步（`async`），并在调用它们时加上 `await` 就行了，开发的心智负担很小，以至于你会忘了自己是在写异步 I/O 而不是阻塞 I/O 。而 Rust 的这种实现让我尤其感到兴奋的是，它的 `async`/`await` 和 `Future` 都是零成本抽象的。

## 零成本抽象

零成本抽象是 Rust 比较独特的一项准则，这是使 Rust 与其他许多语言相区别的原因之一。在添加新功能时，我们非常关心这些新功能是不是零成本的。不过这并不是我们想出来的点子，它在 C++ 中也很重要，所以我认为最好的解释是 Bjarne Stroustrup 的这句话：

> 零成本抽象意味着你不使用的东西，你不用为它付出任何代价，进一步讲，你使用的东西，你无法写出比这更好的代码。

> Zero Cost Abstractions: What you don't use, you don't pay for. And further: What you do use, you couldn't hand code any better.

也就是说零成本抽象有两个方面：

1. 该功能不会给不使用该功能的用户增加成本，因此我们不能为了增加新的特性而增加那些会减慢所有程序运行的全局性开销。
2. 当你确实要使用该功能时，它的速度不会比不使用它的速度慢。如果你觉得，我想使用这个非常好用的功能把开发工作变得轻松，但是它会使我的程序变慢，所以我打算自己造一个，那么这实际上是带来了更大的痛苦。

所以，我将回顾一下我们如何尝试解决异步 I/O 和 Rust 的问题，以及在我们实现这一目标的过程中，某些未能通过这两项零成本测试的特性。

## 绿色线程的尝试

我们要解决的问题是 **异步 I/O** 。通常 I/O 处于阻塞状态，因此当你使用 I/O 时，它会阻塞线程，中止你的程序，然后必须通过操作系统重新调度。阻塞式 I/O 的问题是当你尝试通过同一程序提供大量连接时，它无法真正实现扩展。因此对于真正的大规模网络服务，你需要某种形式的非阻塞的或者说异步的 I/O 。尤其是 Rust 是针对具有真正的高性能要求而设计的语言，它是一种系统编程语言，面向那些真正在乎计算资源的人。要在网络的世界中真正取得成功，我们就需要某种解决方案来解决这个异步 I/O 问题。

但是 **异步 I/O 的最大问题是它的工作方式** ：在你调用 I/O 时，系统调用会立即返回，然后你可以继续进行其他工作，但你的程序需要决定如何回到调用该异步 I/O 暂停的那个任务线上，这就使得在编码上，异步 I/O 的代码要比阻塞 I/O 的代码复杂得多。所以，很多，尤其是以可扩展的网络服务这类特性为目标的语言，一直在试图解决这个问题。比如，让它不再是最终用户需要解决的问题，而是编程语言的一部分或者某个库的一部分等等。

Rust 最初使用的第一个解决方案是 **绿色线程**，它已经在许多语言中获得成功。绿色线程基本上就像阻塞式 I/O 一样，使用的时候就像是普通的线程，它们会在执行 I/O 时阻塞，一切看起来就跟你在使用操作系统的原生方式一样。但是，它们被设计为语言运行时的一部分，来对那些需要同时运行成千上万甚至数百万个绿色线程的网络服务用例进行优化。一个使用该模型的典型的成功案例就是 Go 语言，它的绿色线程被称为 goroutine。对于 Go 程序来说，同时运行成千上万个 goroutine 是很正常的，因为与操作系统线程不同，创建它们的成本很低。

|   | 操作系统线程 | 绿色线程 |
| ------------ | ------------ | ------------ |
|内存开销|较大的堆栈，增加大量内存占用|初始堆栈非常小|
|CPU开销|上下文切换至操作系统的调度器，成本很高|由程序本身的运行时调度|


即 **绿色线程的优点** 在于，产生操作系统线程时的内存开销要高得多，因为每个操作系统线程会创建一个很大的堆栈，而绿色线程通常的工作方式是，你将产生一个以很小的堆栈，它只会随着时间的推移而增长，而产生一堆不使用大量内存的新线程并不便宜；并且使用类似操作系统原语的问题还在于你依赖于操作系统调度，这意味着你必须从程序的内存空间切换到内核空间，如果成千上万的线程都在快速切换，上下文切换就会增加很多开销。而将调度保持在同一程序中，你将避免使用这些上下文，进而减少开销。所以我相信绿色线程是一个非常好的模型，适用于许多语言，包括 Go 和 Java。

在很长一段时间内， Rust 都有绿色线程，但是在 1.0 版本之前删掉了。我们删掉它是因为它不是零成本抽象的，准确的说就是我在第一个问题中谈到的，它给那些不需要它的人增加了成本。比如你只想编写一个不是网络服务的屏幕打印的 Rust 程序，你必须引入负责调度所有绿色线程的语言运行时。这种方法，尤其是对于试图把 Rust 集成到一个大的 C 应用程序中的人来说，就成为一个问题。很多 Rust 的采用者拥有一些大型C程序，他们想开始使用 Rust 并将 Rust 集成到他们的程序中，只是一小段 Rust 代码。问题是，如果你必须设置运行时才能调用 Rust ，那么这一小部分的 Rust 程序的成本就太高了。因此从 1.0 开始，我们就从语言中删除了绿色线程，并删除了语言的运行时。现在我们都知道它的运行时与 C 基本上相同，这就使得在 Rust 和 C 之间调用非常容易，而且成本很低，这是使 Rust 真正成功的关键因素之一。删除了绿色线程，我们还是需要某种异步 I/O 解决方案；但是我们意识到 **这应该是一个基于库的解决方案，我们需要为异步 I/O 提供良好的抽象，它不是语言的一部分，也不是每个程序附带的运行时的一部分，只是可选的并按需使用的库。**


## Future 的解决方案

最成功的库解决方案是一个叫做 `Future` 的概念，在 JavaScript 中也叫做 `Promise`。`Future` 表示一个尚未得出的值，你可以在它被解决（resolved）以得出那个值之前对它进行各种操作。在许多语言中，对 `Future` 所做的工作并不多，这种实现支持很多特性比如组合器（Combinator），尤其是能让我们在此基础上实现更符合人体工程学的 `async`/`await` 语法。

`Future` 可以表示各种各样的东西，尤其适用于表示异步 I/O ：当你发起一次网络请求时，你将立即获得一个 `Future` 对象，而一旦网络请求完成，它将返回任何响应可能包含的值；你也可以表示诸如“超时”之类的东西，“超时”其实就是一个在过了特定时间后被解决的 `Future` ；甚至不属于 I/O 的工作或者需要放到某个线程池中运行的CPU密集型的工作，也可以通过一个 `Future` 来表示，这个 `Future` 将会在线程池完成工作后被解决。

```rust
trait Future {
	type Output;
	fn schedule<F>(self, callback: F)
		where F: FnOnce(Self::Output);
}
```

**`Future` 存在的问题** 是它在大多数语言中的表示方式是这种基于回调的方法，使用这种方式时，你可以指定在 `Future` 被解决之后运行什么回调函数。也就是说， `Future` 负责弄清楚什么时候被解决，无论你的回调是什么，它都会运行；而所有的不便也都建立在此模型上，它非常难用，因为已经有很多开发者进行了大量的尝试，发现他们不得不写很多分配性的代码以及使用动态派发；实际上，你尝试调度的每个回调都必须获得自己独立的存储空间，例如 crate 对象、堆内存分配，这些分配以及动态派发无处不在。这种方法没有满足零成本抽象的第二个原则，如果你要使用它，它将比你自己写要慢很多，那你为什么还要用它。

## 基于轮询的解决方案

```rust
// 基于轮询的 Future

trait Future {
	type Output;
	fn poll(&mut self, waker: &Waker)
		-> Poll<Self::Output>;
}

enum Poll<T> {
	Ready(T),
	Pending,
}
```

这个非常出色的基于轮询的新方案——我们编写了这个模型，我归功于 Alex 和 Aaron Turon，是他们提出了这个想法——不是由 `Future` 来调度回调函数，而是由我们去轮询 `Future`，所以还有另一个被称为执行器（executor）的组件，它负责实际运行 `Future` ；执行器的工作就是轮询 `Future` ，而 `Future` 可能返回“尚未准备就绪（Pending）”，也可能被解决就返回“已就绪（Ready）”。

该模型有很多优点。其中一个优点是，你可以非常容易地取消 `Future` ，因为取消 `Future` 只需要停止持有 `Future`。而如果采用基于回调的方法，要通过调度来取消并使其停止就没这么容易了。

同时它还能够使我们在程序的不同部分之间建立真正清晰的抽象边界，大多数 `Future` 库都带有事件循环（event loop），这也是调度你的 `Future` 执行 I/O 的方法，但你实际上对此没有任何控制权。而在 Rust 中，各组件之间的边界非常整洁，执行器（executor）负责调度你的 `Future` ，反应器（reactor）处理所有的 I/O ，然后是你的实际代码。因此最终用户可以自行决定使用什么执行器，使用他们想使用的反应器，从而获得更强的控制力，这在系统编程语言中真的很重要。而此模型最重要的真正优势在于，它使我们能够以一种真正零成本的完美方式实现这种状态机式的 `Future` 。也就是当你编写的 `Future` 代码被编译成实际的本地（native）代码时，它就像一个状态机；在该状态机中，每次 I/O 的暂停点都有一个变体（variant），而每个变体都保存了恢复执行所需的状态。这表示为一个枚举（enum）结构，即一个包含变体判别式及所有可能状态的联合体（union）。

![StateMachines](StateMachines.jpg)

> 译者注：报告视频中的幻灯片比较模糊，我对其进行了重绘与翻译，下同。

上面的幻灯片尽可能直观地表示了这个状态机模型。可以看到，你执行了两个 I/O 事件，所以它有这几个状态。对于每个状态它都提供了所需的内存空间，足够你在 I/O 事件后恢复执行。

整个 `Future` 只需要一次堆内存分配，其大小就是你将这个状态机分配到堆中的大小，并且没有额外的开销。你不需要装箱、回调之类的东西，只有真正零成本的完美模型。这些概念对于很多人来说比较难于理解，所以这是我力求做到最好的幻灯片，直观地呈现这个过程中发生了什么：你创建一个 `Future`，它被分配到某个内存中特定的位置，然后你可以在执行器（executor）中启动它。

![PollWakeCycle1](PollWakeCycle1.jpg)

执行器会轮询 `Future`，直到最终 `Future` 需要执行某种 I/O 。

![PollWakeCycle2](PollWakeCycle2.jpg)

在这种情况下，该 `Future` 将被移交给处理 I/O 的反应器，即 `Future` 会等待该特定 I/O 。最终，在该 I/O 事件发生时，反应器将使用你在轮询它时传递的Waker 参数唤醒 `Future` ，将其传回执行器；

![PollWakeCycle3](PollWakeCycle3.jpg)

然后当需要再次执行I/O时，执行器再将其放回反应器；它将像这样来回穿梭，直到最终被解决（resolved）。在被解决并得出最终结果时，执行器知道它已经完成，就会释放句柄和整个`Future`，整个调用过程就完成了。

![PollWakeCycle4](PollWakeCycle4.jpg)

总结一下：这种模型形成了一种循环，我们轮询 `Future` ，然后等待 I/O 将其唤醒，然后一次又一次地轮询和唤醒，直到最终整个过程完成为止。

![PollWakeCycle](PollWakeCycle.jpg)

并且这种模型相当高效。

![QuiteFast](QuiteFast.jpg)

这是在有关 `Future` 的第一篇文章中发布的基准测试，与其他语言的许多不同实现进行了对比。柱形越高表示性能越好，我们的 `Future` 模型在最左侧，所以说我们有了非常出色的零成本抽象，即使是与许多其他语言中最快的异步 I/O 实现相比也是相当有竞争力的。

但是，问题在于，你并不希望手动编写这些状态机，把整个应用程序所有状态写成一个状态机并不是件轻松愉快的事。而这种 `Future` 抽象的真正有用之处在于，我们可以在其之上构建其他 API 。

## 其它API

### 组合器（Combinator）

我们能使用的第一个解决方案是 `Future` 组合器（Combinator）。你可以通过将这些组合器方法应用于 `Future` 来构建状态机，它们的工作方式类似于迭代器（`Iterator`）的适配器（如 `filter`、`map`）。

```rust
fn fetch_rust_lang(client: Client)
	-> impl Future<Output = String> {
	client.get("rust-lang.org").and_then(|response| {
		response.concat_body().map(|body| {
			String::from_utf8_lossy(body)
		})
	})
}
```

这个函数的作用是请求 “rust-lang.org”，然后将响应转换为字符串。它并不返回一个字符串，而是返回一个字符串的 `Future` ，因为它是一个异步函数。函数体中有这样一个 `Future`，它包含一些会被调用的 I/O 操作，用 `and_then` 和 `map` 之类的组合器将这些操作全部组合在一起。我们构建了所有这些用处各异的组合器，例如，`and_then`，`map`，`filter`，`map_error`等等。我们已经知道这种方式是有一些缺点的，尤其是诸如嵌套回调之类，可读性非常差。

### `async` / `await` 的实现

因为组合器有这样的缺点，所以我们开始尝试实现 `async` / `await`。`async` / `await` 的第一个版本并不是 Rust 语言的一部分，而是由该库像语法插件一样提供的。

```rust
#[async]
fn fetch_rust_lang(client: Client) -> String {
	let response = await!(client.get("rust-lang.org"));
	let body = await!(response.concat_body());
	String::from_utf9_lossy(body)
}
```

这个函数与之前那个版本的一样，它只是获取 Rust 官网并将其转换为字符串；但是它使用 `async` / `await` 来实现，所以更像是顺序执行，看起来更像普通的阻塞 I/O 的工作方式；就像开头那个实例中呈现的一样，它们唯一区别是注解（指 `async` / `await`）。我们已经知道，`async` 注解会将此函数转换为一个返回 `Future` 的函数，而不是立即返回结果，并且我们需要异步等待（`await`）这些在函数内部构造的 `Future`。

```rust
await!($future) => {
	loop {
		match $future.poll() {
			Poll::Pending => yield Poll::Pending,
			Poll::Ready(value) => break value,
		}
	}
}
```

在我们的轮询模型中，`await` 是一种语法糖；它会进入上面这种循环，你要做的就是在循环中轮询，在一段时间内你将一直得到“尚未准备就绪（Pending）”，然后一直等到它再次被唤醒，终于你等待的 `Future` 完成了，然后你使用该值跳出了循环，这就是这些 `await` 表达式的计算结果。

这似乎是一个非常不错的解决方案，`async` / `await` 的写法会被编译成我们超棒的零成本的 `Future`。不过从已发布的 `Future` 的使用者的反馈看，我们还是发现了一些问题。

### 新的问题

基本上所有试图使用 `Future` 的人都会遇到非常令人困惑的错误消息。在这些消息中，编译器会提示你的`Future`的生命周期不是静态的（`'static`）或没有实现某个 `trait` 等等；这些提示你并不真正理解，但编译器想提出有用的建议，你也就跟着这个建议去做，直到编译成功；你可能会给闭包加上 `move` 关键字，或者把某些值放到引用计数的指针（`Rc`）中，然后将复制（`clone`）它；你将所有这些开销添加到了似乎并不必要的事情上，却不明白为什么要这样做，而当你已经疲于处理这些时，代码已经乱成一锅粥了，所以很多人都被 `Future` 的问题卡住了。

![CombinatorChain](CombinatorChain.jpg)

而且它对于组合器产生这种非常大的类型也没什么办法，你的整个终端窗口（terminal）将被其中一个组合器链的类型填满。你用了`and_then`，以及又一个 `and_then`，然后是 `map_err` 紧跟一个 TCP 流等等等等，你必须仔细研究一下，才能弄清楚所遇到的实际错误是什么。

> “When using Futures, error messages are inscrutable.”
> “当使用 Future 时，错误信息难以理解。”
> “Having to use RefCell or clone everything for each future leads to overcomplicated code that makes me wish Rust had garbage collection.”
> “不得不使用 RefCell 以及为每个 future 克隆所有它需要的值产生了过于复杂的代码，这让我开始期待 Rust 能具备垃圾回收功能了。”

我在 reddit 上发现了这条消息，我认为它确实很好地总结了所有有关 `Future` 的抱怨。使用 `Future` 时，错误消息难以理解；不得不使用 RefCell 以及为每个 future 克隆所有它需要的值产生了过于复杂的代码，这让我开始期待 Rust 能具备垃圾回收功能了（观众笑）。是的，这不是什么好反馈。

因此，从大约一年半前的情况来看，很明显，有两个问题需要解决，才能让大家更容易使用。

**首先**，我们需要更好的错误消息，最简单的方法就是将语法构建到语言中，然后它们就可以在你所有的诊断和错误处理代码中加入钩子，从而使你能够真正拥有良好的 `async` / `await` 的错误消息。**其次**，人们遇到的大多数错误实际上是因为他们被一个晦涩难解的问题卡住了——借用问题。正是因为 `Future` 的设计方式存在着这种根本的局限性，导致一些很普通的编程范式都无法表达。所谓借用问题，就是在最初的 `Future` 的设计中你不能跨过**异步等待点**（await point）进行借用，也就是说，如果你要异步等待（await）某件事，你就不能在那个时候持有任何存活的引用。当人们遇到了这种问题，他们通常会被卡住，最终他们会尝试在 `await` 的时候进行借用然后发现实际上做不到。所以如果我们能够使这种借用被允许，那么大多数这些错误将消失，一切都将变得更易于使用，你可以使用 `async` 和 `await` 编写普通的 Rust 代码，并且一切都会正常进行。

这种跨越 `await` 的借用是非常普遍的，因为 Rust 的 API 天然就具有引用。当你实际编译 `Future` 时，它必须能够恢复所有状态，而当你拥有某些其它东西的引用时，它们位于同一栈帧中，最终你会得到一个**自引用**的 `Future` 结构（Self-Referential Future）。这是开头的那个 `get_user` 方法，我们有这样一个 SQL 字符串，而在使用 SQL 字符串调用 `query` 方法时，我们传递的是 SQL 字符串的引用。

```rust
let sql: String = format!("SELECT FROM users WHERE username = {}", user);
let db_response = await self.query(&sql);
```

这里存在的问题是，对 SQL 字符串的引用是对存储在相同 `Future` 状态中的其他内容的引用，因此它成为一种自引用结构。

![SelfReferentialFuture](SelfReferentialFuture.jpg)

如果把这个 `Future` 视作一个真的结构体的话，这就是理论上它所拥有的字段。除了代表数据库句柄的 `self` 之外，还有 SQL 字符串以及对这个 SQL 字符串的引用，即一个最终指回同一结构体中某个字段的引用。

一些新的潜在结构会成为非常棘手的问题，因为我们没有通用的解决方案。当你移动该结构时，我们不允许你使用自引用，是因为我们会在新位置创建一个原有结构的新副本，旧副本会变为无效，但是在复制时，新副本的引用仍指向旧副本的字段，该指针就变成了悬空指针（dangling pointer），而这正是 Rust 必须避免的内存问题。

![DanglingPointer](DanglingPointer.jpg)

所以我们不能使用自引用结构，因为如果你移动它们，那么它们将失效。然而，我们实际上并不需要真正移动这些 `Future` 。如果你还记得在堆中通过句柄使用 `Future` 的模型，它在反应器和执行器之间来回传递，所以 `Future` 本身永远不会真正移动；而只要你保证不移动，`Future` 包含自引用就完全没问题。

所以我们需要通过采用某种方式在 `Future` 的 API 中表达 “在你轮询时，你不允许随意移动它” 来解决这个问题。如果我们能够表达这一点，我们就可以允许 `Future` 中出现自引用，进而就可以在异步函数中真正使用这些引用，并且一切都会正常工作。因此我们研究了这个问题，最终开发出了被称为 `Pin` 的新 API 。

### `Pin`

`Pin` 是一种围绕其他指针类型的适配器，可以将其它指针变为 **固定引用**（pinned reference）。除了原有指针的保证外，它还保证这个引用再也不会被移动，所以我们可以确保它将一直处在同一内存位置，直到最终被释放。如果你的 API 中有一些内容已表明必须使用 `Pin`，那么你就知道了它再也不会被移动，这样就你可以使用那种自引用的结构体了。因此，我们修改了 `Future` 的工作方式，现在它变成了一个经过装箱（boxed）的 `Future` ，实际上是一个由 `Pin` 封装的 `Box<Future>` ；这样无论在哪里装箱，我们把它放在堆内存中，它可以保证永远不会再移动；而当你轮询 `Future` 时，不再使用一个普通引用，而是使用一个固定引用，这样 `Future` 就知道它不会被移动。而使这一切起作用的诀窍是，要从固定引用中获取非固定引用，你只能使用不安全的代码。

```rust
struct Pin<P>(P);

impl<T> Pin<Box<T>> {
	fn new(boxed: Box<T>) -> Pin<Box<T>> { ... }
	fn as_mut(self) -> Pin<&mut T> { ... }
}

impl<T> Pin<&mut T> {
	unsafe fn get_unchecked_mut(self) -> &mut T { ... }
}
```

所以 Pin API 大致就像这样。在这里你有一个 `Pin` 结构，它只是另一个指针类型的封装，它没有任何运行时开销或者其它东西，仅仅是将其划分为一个固定的（pinned）对象，然后一个固定的 `Box` 指针可以转换为一个固定引用，但是将一个固定引用转换为一个非固定引用的唯一方法是使用一个不安全的（unsafe）函数。

```rust
trait Future {
	type Output;
	
	fn poll(self: Pin<&mut self>, waker: &Waker)
		-> Poll<Self::Output>;
}
```

这样一来，我们要做的就是修改 `Future` 的 API ，其轮询方法将不再接受一个普通引用，而是接受一个固定引用，而这其实就是我们将要稳定发布的 `Future` API。而做了这个修改之后，开头示例的写法就能正常工作了。这样你就可以像写阻塞 I/O 的代码那样编写异步 I/O 的代码了，只需要加上 `async` 和 `await` 注解，你就能得到这个出色的零成本抽象的异步实现，而即便你自己手写，这基本上也是你能写出的开销最低的实现了。

## `async` / `await` 的现状及未来

目前的情况是，Pinning 大约在一个月前的最新版本中稳定了，我们正在稳定 `Future` 的 API ，因此大概会在 1.35，也可能会推到 1.36 稳定，基本上在两三个月内就会知道了。并且我们希望今年的某个时候我们能够拥有 `async` / `await`，希望今年夏末能将其稳定下来，这样人们就可以使用这种类似阻塞 I/O 的语法编写无阻塞的 I/O 网络服务了。除了这些稳定化工作，我们也已经开始研究某些更长期的功能，比如流（Stream），我认为它可能是异步的下一个大功能。我们知道一个 `Future` 只产生一个值，而一个流可以异步地产生很多值；异步地产生值本质上就像是一个异步迭代器，你能够在一个流上进行异步的循环；这个功能对于许多用例来说非常重要，比如流式传输HTTP、WebSocket 推送请求之类的东西，不用像我们的 RPC 模型那样发出网络请求然后获得单个响应，而是能够使用请求流和响应流，在两者之间来回调用。

目前使用异步仍然存在一个限制，即不能在 `trait` 中使用 `async`。有许多编译器开发工作正在进行，使其能够支持这个特性。除了这些功能，有时候我们也希望能拥有生成器（Generator），类似于 Python 或 JavaScript 的生成器，除了拥有可以 `return` 的函数，还能使用可以 `yield` 的函数，这样你就可以在 `yield` 之后再次恢复执行；并且你可以将这些函数作为编写迭代器和流的方式，就像异步函数能够让你像编写普通函数那样编写 `Future` 一样。

最后，我想回顾一下成就这种零成本异步 I/O 的关键点：首先就是这种基于轮询的 `Future`，它将这些 `Future` 编译到这种相当紧凑的状态机中；其次是这种实现 `async` / `await` 语法的方式，即因为有了 `Pin`，我们能够跨越异步等待点使用引用。

谢谢大家。

> 译者注：报告中计划稳定的特性均已稳定发布，参见 [areweasyncyet.rs](https://areweasyncyet.rs/)。
> 按照目前稳定的版本，`await` 已改为后置运算符 `.await`，所以本文开头的 `get_user` 方法应当修改为：
```rust
let user = db.get_user("withoutboats").await;

impl Database {
    async fn get_user(&mut self, user: &str) -> User {
        let sql = format!("select FROM users WHERE username = {}", user);
        let db_response = self.query(&sql).await;
        User::from(db_response)
    }
}
```
