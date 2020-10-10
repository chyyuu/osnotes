## 2020-09-24

Rust 当中只关心位置和值。除非与 trait object 相关，否则不会用“对象”这个术语。

Rust 的很多内建类型均标记为 `Copy` trait。标记为 `Copy` 的类型在按值传参、赋值的时候会取复制语义，可以理解成将一块内存原样复制到另一个地方，并通常绑定到不同的名字下面；否则，通常为了保证值不能出现多次或是有一些限制，在按值传参、赋值的时候会取**移动**语义。它大致可以分成两个阶段：前一阶段和 `Copy` 一致，后一阶段则是要在编译器层面 invalidate 掉被移动的变量，禁止它再被访问。那么被移动的变量对应的位置又该何去何从呢？这个问题暂时难以得出结论。

另一个问题是，对于自己创建的类型，`Copy` 和 `Drop` 不能并存。这可能是因为，标记为 `Copy` trait 的类型，和一段固定大小的空间没有什么区别。而未被标记为 `Copy` trait 的类型，由于它被限制为只能取移动语义，往往在释放的时候都不是直接扔掉那一块空间了事即可，而是要有一些相应的处理。

一个强大的[链接](https://zhuanlan.zhihu.com/p/189694498)

## 2020-09-25

上面的经典文章，提到了位置和值的概念，按照我的理解，位置就是指一块内存（无论是全局数据、堆、还是栈），而值当然指的是这块内存中保存的内容。

**位置**

一个位置可能有以下的五种状态：

1. 位置存在，但里面还没有值；
2. 位置存在，且里面也有值；
3. 位置存在，里面曾经有值，但现在被移走了；
4. 位置存在且有值，正在被共享借用（多个不可变借用）；
5. 位置存在且有值，正在被独占借用（单个可变借用）。

我们分别重新叫做未初始化、有值、deleted、&、&mut 状态吧。

状态只是位置的第一重属性，他还具有其他属性。

比如位置的类型属性，包括类型大小、内存对齐以及析构方法，还有一些标记。当一个类型被标记为 `Copy` trait 的时候，与该类型相关的位置就不存在 deleted 状态。事实上，将自定义的类型标记为 `Copy` 之后，无法再为其实现 `Drop` trait，其中的原因我昨天尝试分析了一下。

还有位置的可变性属性（w: 大概可变性放在位置这个层面而不是变量这个层面会更好一些）。当没有 mut 的时候，一个位置就不能位于状态 &mut，且离开了未初始化状态之后，里面的值不能发生变化。

位置的来源：通过定义变量、函数参数、返回值均可生成带名字的位置，变量可以视为**名字与位置的绑定**。此外，还有一些隐式的临时变量也会有对应的位置，只是我们无法显式访问罢了。

**值**

类型决定了值的内存布局。

Python 和 Java 这类语言提供值的运行时管理器，而 Rust 一切都在编译器中完成。

**观测交互**

打印一个位置的地址会引发编译器关于内存分配策略的变化。

**赋值(assignment)**

Rust 的赋值机制相比其他语言有很多不同。

假设 a,b 两个位置有着相同的类型，我们想要实现 a = b(即 a 为目标，b 为源)

在编译的时候我们需要依次检查：

1. 若 a,b 处于 & 或 &mut 状态，它们将在这一瞬间回到有值状态。这也就意味着所有关于它们的引用不能在这之后继续存在，编译期的借用检查器会帮我们检查是否产生生命周期冲突。
2. 若位置 a 处于有值或 deleted 状态，且位置 a 并不是 mut 的，报错；
3. 若位置 b 处于未初始化状态或 deleted 状态，报错。

这里我写了一段代码来验证第一条：

```rust
fn main() {
    let mut p1 = Person { v: 10, };
    let rp = &p1;
    let p2 = Person { v: 11, };
    p1 = p2;
    println!("{:?}", *rp);
}
```

会报出编译错误：

```rust
error[E0506]: cannot assign to `p1` because it is borrowed
  --> src\main.rs:15:5
   |
13 |     let rp = &p1;
   |              --- borrow of `p1` occurs here
14 |     let p2 = Person { v: 11, };
15 |     p1 = p2;
   |     ^^ assignment to borrowed `p1` occurs here
16 |     println!("after world!");
17 |     println!("{:?}", *rp);
   |                      --- borrow later used here
```

我敢说如果是 C 语言这样肯定没有问题。

这告诉我们，借用是对于**值**的借用而非对于**位置**的借用。注意下面的运行时步骤，若没有被标记为 Copy，实际上 p1 里面的值在运行时第一步已经被销毁，在值被销毁之后，之前的对于它的借用 rp 也就失效了。而我们又尝试去访问它，即在编译期发现了借用错误。

注意，即使将 `Person` 标记为 `Copy` 该问题也同样没有得到解决，不能通过编译。

但是 p2 的情况则稍有不同。若 p2 被标记为 Copy 的话，p2 位置里面的值其实不会受到影响，因此跨越它的借用不会出现问题；但反之，p2 位置里面的值需要被 invalidate，由于借用是对值进行借用，就不行了。

所以在我看来，原作的说明不太完整。已经提出评论了，期待和 dalao 交流一下。

运行时步骤：

1. 如果 a 处于有值状态，且类型不是 `Copy` 的，那么**对于 a 里面的值进行析构**，从而进入 deleted 状态；
2. 将位置 b 里面的值按位复制到位置 a 中，位置 a 进入有值状态；
3. 如果类型不是 `Copy` 的，位置 b 进入 deleted 状态。

(w: 这样看来的话，一个位置处于未初始化和 deleted 的状态虽然都不能访问，但是报出的错误却应该是不同的。)

**所有权到底是什么**

感觉一下子懂了好多东西，明天再继续来看，[here](https://zhuanlan.zhihu.com/p/201220495)。

## 2020-09-28

原作者对文章进行了修改，在编译器在赋值时候对源位置的借用检查，确实需要考虑到类型是否为 Copy。

此外，原作者否定了我说“借用是对值的借用而非对位置的借用”的说法。实际上，借用是对“**有值的位置**”的借用。值不能脱离位置而存在。最典型未实现 `Copy` trait 将源位置的值 move 出来的情况，实际上是源位置进入了 deleted 状态，所以它不再是一个"有值的位置"，故此引用失效。目标位置在被赋值的时候也会短暂进入 deleted 状态，从而将先前对它的借用无效化。

## 2020-10-05

抽一点时间继续来看经典文章。

Rust 中的所有权模型实际上是没有我们之前构想的位置-值模型精确的。若从位置-值模型来理解所有权的话，大概也就是需要做到 `move` 语义的正确实现，以及在适当的时间点销毁位置和里面可能的值。如果是从源码级的角度来理解，一个变量通常绑定到一个位置上面并拥有其所有权，在变量的生命周期结束后负责销毁对应的位置。严格来说每个位置都应该被一个变量所有。但是变量里面包括编译器为我们生成的一些临时的匿名变量，我们是没法通过它们直接来访问对应的内存的，而只能通过间接的方式。比如我们不会为链表上每个节点都开一个变量，而是只保存它的表头。这样的话，链表的一切操作都只能从表头发起，我们没办法单独回收一个中间节点。

回头去看了一下生命周期参数。其主要的目的是符合借用规则，即输出引用不能 outlive 输入引用。如果一个函数没有输出引用的话，其实输入引用没必要加入任何生命周期泛型。生命周期泛型是一个泛型，会在编译期调用函数的时候将泛型单例化为具体的生命周期。最关键的检查就是与输出引用有关。生命周期泛型限制，如 `'b: 'a` 在编程之道书中提到的 `longest(a: &'a str, b: &'b str) -> &'a str`，这里的限制是指输入的 `'b` 包含 `'a`，或 `'a` 是 `'b` 的一个子集。这样的话才能不违背借用规则。如果没有输出引用，各个输入引用可以随意命名为相同或不同生命周期参数，互相不影响。 

重新从 `std::thread` 的角度来考虑 `Send` 和 `Sync` 的问题。为何 `Rc` 未实现 `Send`？`Send` 意味着可以安全的转移所有权，从 `move` 语义来讲就是说值从一个栈按位复制到另一个栈，并 invalidate 原来栈上的那个变量。但是按位复制该变量的时候也需要经历多条指令。如果在 `move` 到另一个栈的过程中，另一个核访问了另一个 `Rc`，就会产生问题。而 `Arc` 可能是利用原子指令来实现类似于无锁数据结构，这种并发访问就不会出问题。

## 2020-10-10

之前提到闭包的时候想到了，现在记录下来。其实闭包和 Future 在某种程度上比较相像，都是编译器自动为我们生成一个结构体并实现 `Fn, FnMut, FnOnce` 中某个 trait，它关键的变量捕获功能其实就是把当前作用域待捕获的变量以引用或者转移所有权（类比直接传参，取决于类型是否实现 `Copy` trait）的形式保存在结构体的 fields 中。这样听起来的话，的确可以称为一个闭包（Closure）了。可以发现调用它的时候无需传参，且无论在任何地方都可以调用。

现在在回顾 zCore 里面的异步 OS 设计模式，里面用到了大量的 Rust 函数式编程，所以去回顾一下闭包和迭代器。

闭包可以从当前上下文中捕获变量并储存在编译器自动生成的结构体中。一共有三种不同的捕获方式：

`FnOnce` 消耗闭包里面用到的变量，并将其所有权转移到闭包环境之内。因此它只能被调用一次，不然会出现 moved value 错误。 

`FnMut` 以可变引用的形式捕获闭包里面用到的变量。

`Fn` 则是以不可变引用的形式捕获闭包里面用到的变量。

捕获的时机是在闭包被声明的时候，也就是匿名结构体被构造出来的时候。

当然，除了捕获的变量之外，我们还可能传一些参数给闭包，这些内容共同构成了闭包的执行环境。闭包可以捕获变量，可以在另一个环境下调用，更加灵活，但是相比函数调用有一些内存和所有权转移开销。

`FnOnce, FnMut, Fn` 更像是一种标记，可以在闭包作为参数的时候对闭包的捕获方式进行一些限制。比如如果类型为 `FnOnce`，那么捕获的变量可以以转移所有权、可变/不可变引用的方式捕获；如果类型为 `FnMut`，那捕获的变量就只能以可变/不可变引用的方式捕获；如果类型为 `Fn`，那就只能以不可变引用的方式捕获。

编译器会分析我们对于每个环境变量的捕获方式，所有权转移/可变引用/共享引用都有可能。通过 `move |...|` 可以强行将闭包里面用到的所有变量都进行所有权转移。但是这取决于变量的类型，如果被 `Copy` 标记的话并不会进行所有权转移，而只是会在闭包结构体里面复制一份。

比如下面的代码就不会有问题：

```c
fn main() {
    let x = 3;
    let p = move || { println!("{}", x) };
    println!("x = {}", x);
    p();
}
```

目前看来，如果将一个闭包作为参数，很有可能是传进去的那个环境有闭包所需的参数，然后就直接在那个环境跑闭包了，或者是将闭包进一步包装留待以后使用。

一句题外话是，如果想给一个结构体 derive `Copy` 需要先 derive `Clone`。一般情况下来讲 `Clone` 应该是深复制，大概率导致其拷贝从二进制的角度来说是不同的；而 `Copy` 的行为则是在赋值/传参/捕获过程中发生的完全的按位复制。这样，这种“连 `Clone` 都不支持更别提 `Copy` 了”的机制倒是也有一定的道理。