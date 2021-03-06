## 为使用不同类型的值而设计的 trait 对象

> [ch17-02-trait-objects.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch17-02-trait-objects.md)
> <br>
> commit 67876e3ef5323ce9d394f3ea6b08cb3d173d9ba9

 在第八章中，我们谈到了 vector 只能存储同种类型元素的局限。在列表 8-1 中有一个例子，其中定义了一个拥有分别存放整型、浮点型和文本型成员的枚举类型 `SpreadsheetCell`，使用这个枚举的 vector 可以在每一个单元格（cell）中储存不同类型的数据，并使得 vector 整体仍然代表一行（row）单元格。这当编译代码时就知道希望可以交替使用的类型为固定集合的情况下是可行的。

<!-- The code example I want to reference did not have a listing number; it's
the one with SpreadsheetCell. I will go back and add Listing 8-1 next time I
get Chapter 8 for editing. /Carol -->

有时，我们希望使用的类型的集合对于使用库的程序员来说是可扩展的。例如，很多图形用户接口（GUI）工具有一个项目列表的概念，它通过遍历列表并调用每一个项目的 `draw` 方法来将其绘制到屏幕上。我们将要创建一个叫做 `rust_gui` 的库 crate，它含一个 GUI 库的结构。这个 GUI 库包含一些可供开发者使用的类型，比如 `Button` 或 `TextField`。使用 `rust_gui` 的程序员会想要创建更多可以绘制在屏幕上的类型：其中一些可能会增加一个 `Image`，而另一些可能会增加一个 `SelectBox`。本章节并不准备实现一个功能完善的 GUI 库，不过会展示其中各个部分是如何结合在一起的。

编写 `rust_gui` 库时，我们并不知道其他程序员想要创建的全部类型，所以无法定义一个 `enum` 来包含所有这些类型。我们所要做的是使 `rust_gui` 能够记录一系列不同类型的值，并能够对其中每一个值调用 `draw` 方法。 GUI 库不需要知道当调用 `draw` 方法时具体会发生什么，只需提供这些值可供调用的方法即可。

在拥有继承的语言中，我们可能定义一个名为 `Component` 的类，该类上有一个 `draw` 方法。其他的类比如 `Button`、`Image` 和 `SelectBox` 会从 `Component` 派生并因此继承 `draw` 方法。它们各自都可以覆盖 `draw` 方法来定义自己的行为，但是框架会把所有这些类型当作是 `Component` 的实例，并在其上调用 `draw`。

### 定义通用行为的 trait

不过，在 Rust 中，我们可以定义一个 `Draw` trait，包含名为 `draw` 的方法。接着可以定义一个存放**trait 对象**（*trait
object*）的 vector，trait 对象是一个位于某些指针，比如 `&` 引用或 `Box<T>` 智能指针，之后的 trait。第十九章会讲到为何 trait 对象必须位于指针之后的原因。

之前提到过，我们并不将结构体与枚举称之为“对象”，以便与其他语言中的对象相区别。结构体与枚举和 `impl` 块中的行为是分开的，不同于其他语言中将数据和行为组合进一个称为对象的概念中。trait 对象将由指向具体对象的指针构成的数据和定义于 trait 中方法的行为结合在一起，从这种意义上说它**则**更类似其他语言中的对象。不过 trait 对象与其他语言中的对象是不同的，因为不能向 trait 对象增加数据。trait 对象并不像其他语言中的对象那么通用：他们（trait 对象）的作用是允许对通用行为的抽象。

trait 对象定义了在给定情况下所需的行为。接着就可以在要使用具体类型或泛型的地方使用 trait 来作为 trait 对象。Rust 的类型系统会确保任何我们替换为 trait 对象的值都会实现了 trait 的方法。这样就无需在编译时就知道所有可能的类型，就能够用同样的方法处理所有的实例。列表 17-3 展示了如何定义一个带有 `draw` 方法的 trait `Draw`：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub trait Draw {
    fn draw(&self);
}
```

<span class="caption">列表 17-3:`Draw` trait 的定义</span>

<!-- NEXT PARAGRAPH WRAPPED WEIRD INTENTIONALLY SEE #199 -->



因为第十章已经讨论过如何定义 trait，这看起来应该比较眼熟。接下来就是新内容了：列表 17-4 有一个名为 `Screen` 的结构体定义，它存放了一个叫做 `components` 的 `Box<Draw>` 类型的 vector 。`Box<Draw>` 是一个 trait 对象：它是 `Box` 中任何实现了 `Draw` trait 的类型的替身。

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub trait Draw {
#     fn draw(&self);
# }
#
pub struct Screen {
    pub components: Vec<Box<Draw>>,
}
```

<span class="caption">列表 17-4: 一个 `Screen` 结构体的定义，它带有一个字段`components`，其包含实现了 `Draw` trait 的 trait 对象的 vector</span>

在 `Screen` 结构体上，我们将定义一个 `run` 方法，该方法会对其 `components` 上的每一个元素调用 `draw` 方法，如列表 17-5 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub trait Draw {
#     fn draw(&self);
# }
#
# pub struct Screen {
#     pub components: Vec<Box<Draw>>,
# }
#
impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

<span class="caption">列表 17-5:在 `Screen` 上实现一个 `run` 方法，该方法在每个 component 上调用 `draw` 方法</span>

这与定义使用了带有 trait bound 的泛型类型参数的结构体不同。泛型类型参数一次只能替代一个具体的类型，而 trait 对象则允许在运行时替代多种具体类型。例如，可以像列表 17-6 那样定义使用泛型和 trait bound 的结构体 `Screen`：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub trait Draw {
#     fn draw(&self);
# }
#
pub struct Screen<T: Draw> {
    pub components: Vec<T>,
}

impl<T> Screen<T>
    where T: Draw {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

<span class="caption">列表 17-6: 一种 `Screen` 结构体的替代实现，它的 `run` 方法使用泛型和 trait bound</span>

这只允许我们拥有一个包含全是 `Button` 类型或者全是 `TextField` 类型的 component 列表的 `Screen` 实例。如果只拥有相同类型的集合，那么使用泛型和 trait bound 是更好的，因为在编译时使用具体类型其定义是单态（monomorphized）的。

相反对于存放了 `Vec<Box<Draw>>` trait 对象的 component 列表的 `Screen` 定义，一个 `Screen` 实例可以存放一个既可以包含 `Box<Button>`，也可以包含 `Box<TextField>` 的 `Vec`。让我们看看它是如何工作的，接着会讲到其运行时性能影响。

### 来自我们或者库使用者的 trait 实现

现在来增加一些实现了 `Draw` trait 的类型。我们将提供 `Button` 类型，再一次重申，真正实现 GUI 库超出了本书的范畴，所以 `draw` 方法体中不会有任何有意义的实现。为了想象一下这个实现看起来像什么，一个 `Button` 结构体可能会拥有 `width`、`height`和`label`字段，如列表 17-7 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub trait Draw {
#     fn draw(&self);
# }
#
pub struct Button {
    pub width: u32,
    pub height: u32,
    pub label: String,
}

impl Draw for Button {
    fn draw(&self) {
        // Code to actually draw a button
    }
}
```

<span class="caption">列表 17-7: 一个实现了`Draw` trait 的 `Button` 结构体</span>

在 `Button` 上的 `width`、`height` 和 `label` 字段会和其他组件不同，比如 `TextField` 可能有 `width`、`height`、`label` 以及 `placeholder` 字段。每一个我们希望能在屏幕上绘制的类型都会使用不同的代码来实现 `Draw` trait 的 `draw` 方法，来定义如何绘制像这里的 `Button` 类型（并不包含任何实际的 GUI 代码，这超出了本章的范畴）。除了实现 `Draw` trait 之外，`Button` 还可能有另一个包含按钮点击如何响应的方法的 `impl` 块。这类方法并不适用于像 `TextField` 这样的类型。

一些库的使用者决定实现一个包含 `width`、`height`和`options` 字段的结构体 `SelectBox`。并也为其实现了 `Draw` trait，如列表 17-8 所示：

<span class="filename">文件名: src/main.rs</span>

```rust
extern crate rust_gui;
use rust_gui::Draw;

struct SelectBox {
    width: u32,
    height: u32,
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        // Code to actually draw a select box
    }
}
```

<span class="caption">列表 17-8: 在另一个使用 `rust_gui` 的 crate 中，在 `SelectBox` 结构体上实现 `Draw` trait</span>

库使用者现在可以在他们的 `main` 函数中创建一个 `Screen` 实例，并通过将 `SelectBox` 和 `Button` 放入 `Box<T>` 转变为 trait 对象来将它们放入屏幕实例。接着可以调用 `Screen` 的 `run` 方法，它会调用每个组件的 `draw` 方法。列表 17-9 展示了这个实现：

<span class="filename">文件名: src/main.rs</span>

```rust
use rust_gui::{Screen, Button};

fn main() {
    let screen = Screen {
        components: vec![
            Box::new(SelectBox {
                width: 75,
                height: 10,
                options: vec![
                    String::from("Yes"),
                    String::from("Maybe"),
                    String::from("No")
                ],
            }),
            Box::new(Button {
                width: 50,
                height: 10,
                label: String::from("OK"),
            }),
        ],
    };

    screen.run();
}
```

<span class="caption">列表 17-9: 使用 trait 对象来存储实现了相同 trait 的不同类型的值</span>

即使我们不知道何时何人会增加 `SelectBox` 类型，`Screen` 的实现能够操作`SelectBox` 并绘制它，因为 `SelectBox` 实现了 `Draw` trait，这意味着它实现了 `draw` 方法。

只关心值所反映的信息而不是值的具体类型，这类似于动态类型语言中称为**鸭子类型**（*duck typing*）的概念：如果它走起来像一只鸭子，叫起来像一只鸭子，那么它就是一只鸭子！在列表 17-5 中 `Screen` 上的 `run` 实现中，`run` 并不需要知道各个组件的具体类型是什么。它并不检查组件实例是 `Button` 或者是`SelectBox`，它只是调用组件上的 `draw` 方法。通过指定 `Box<Draw>` 作为 `components` vector 中值的类型，我们就定义了 `Screen` 需要可以在其上调用 `draw` 方法的值。

使用 trait 对象和 Rust 类型系统来使用鸭子类型的优势是无需在运行时检查一个值是否实现了特定方法或者担心在调用时因为值没有实现方法而产生错误。如果值没有实现 trait 对象所需的 trait 则 Rust 不会编译这些代码。

例如，列表 17-10 展示了当创建一个使用 `String` 做为其组件的 `Screen` 时发生的情况：

<span class="filename">文件名: src/main.rs</span>

```rust
extern crate rust_gui;
use rust_gui::Draw;

fn main() {
    let screen = Screen {
        components: vec![
            Box::new(String::from("Hi")),
        ],
    };

    screen.run();
}
```

<span class="caption">列表 17-10: 尝试使用一种没有实现 trait 对象的 trait 的类型</span>

我们会遇到这个错误，因为 `String` 没有实现 `Draw` trait：

```
error[E0277]: the trait bound `std::string::String: Draw` is not satisfied
  -->
   |
 4 |             Box::new(String::from("Hi")),
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `Draw` is not
   implemented for `std::string::String`
   |
   = note: required for the cast to the object type `Draw`
```

这告诉了我们，要么是我们传递了并不希望传递给 `Screen` 的类型并应该提供其他类型，要么应该在 `String` 上实现 `Draw` 以便 `Screen` 可以调用其上的 `draw`。

### trait 对象执行动态分发

回忆一下第十章讨论过的，当对泛型使用 trait bound 时编译器所进行单态化处理：编译器为每一个被泛型类型参数代替的具体类型生成了非泛型的函数和方法实现。单态化所产生的代码进行**静态分发**（*static dispatch*）：当方法被调用时，伴随方法调用的代码在编译时就被确定了，同时寻找这些代码是非常快速的。

当使用 trait 对象时，编译器并不进行单态化，因为并不知道所有可能会使用这些代码的类型。相反，Rust 记录当方法被调用时可能会用到的代码，并在运行时计算出特定方法调用时所需的代码。这被称为**动态分发**（*dynamic dispatch*），进行这种代码搜寻是有运行时开销的。动态分发也阻止编译有选择的内联方法的代码，这会禁用一些优化。尽管在编写和支持代码的过程中确实获得了额外的灵活性，但仍然需要权衡取舍。

### Trait 对象要求对象安全

<!-- Liz: we're conflicted on including this section. Not being able to use a
trait as a trait object because of object safety is something that
beginner/intermediate Rust developers run into sometimes, but explaining it
fully is long and complicated. Should we just cut this whole section? Leave it
(and finish the explanation of how to fix the error at the end)? Shorten it to
a quick caveat, that just says something like "Some traits can't be trait
objects. Clone is an example of one. You'll get errors that will let you know
if a trait can't be a trait object, look up object safety if you're interested
in the details"? Thanks! /Carol -->

不是所有的 trait 都可以被放进 trait 对象中；只有**对象安全**（*object safe*）的 trait 才可以。 一个 trait 只有同时满足如下两点时才被认为是对象安全的:

* trait 不要求 `Self` 是 `Sized` 的
* 所有的 trait 方法都是对象安全的

`Self` 关键字是我们要实现 trait 或方法的类型的别名。`Sized` 是一个类似第十六章中介绍的 `Send` 和 `Sync` 那样的标记 trait。`Sized` 会自动为在编译时有已知大小的类型实现，比如 `i32` 和引用。包括 slice （`[T]`）和 trait 对象这样的没有已知大小的类型则没有。

`Sized` 是一个所有泛型参数类型默认的隐含 trait bound。Rust 中大部分实用的操作都要求类型是 `Sized` 的，所以将 `Sized` 作为默认 trait bound 要求，就可以不必在每一次使用泛型时编写 `T: Sized` 了。然而，如果想要使用在 slice 上使用 trait，则需要去掉 `Sized` trait bound，可以通过指定 `T: ?Sized` 作为 trait bound 来做到这一点。

trait 有一个默认的 bound `Self: ?Sized`，这意味着他们可以在是或者不是 `Sized` 的类型上实现。如果创建了一个去掉了 `Self: ?Sized` bound 的 trait `Foo`，它可能看起来像这样：

```rust
trait Foo: Sized {
    fn some_method(&self);
}
```

trait `Sized` 现在就是 trait `Foo` 的**父 trait**（*supertrait*）了，也就意味着 trait `Foo` 要求实现 `Foo` 的类型（也就是 `Self`）是 `Sized` 的。我们将在第十九章中更详细的介绍父 trait。

像 `Foo` 这样要求 `Self` 是 `Sized` 的 trait 不被允许成为 trait 对象的原因是，不可能为 trait 对象实现 `Foo` trait：trait 对象不是 `Sized` 的，但是 `Foo` 又要求 `Self` 是 `Sized` 的。一个类型不可能同时既是有确定大小的又是无确定大小的。

关于第二条对象安全要求说到 trait 的所有方法都必须是对象安全的，一个对象安全的方法满足下列条件之一：

* 要求 `Self` 是 `Sized` 的，或者
* 满足如下三点：
    * 必须不包含任何泛型类型参数
    * 其第一个参数必须是 `Self` 类型或者能解引用为 `Self` 的类型（也就是说它必须是一个方法而非关联函数，并且以 `self`、`&self` 或 `&mut self` 作为第一个参数）
    * 必须不能在方法签名中除第一个参数之外的地方使用 `Self`

虽然这些规则有一点形式化, 但是换个角度想一下：如果方法在它的签名的其他什么地方要求使用具体的 `Self` 类型，而一个对象又忘记了它具体的类型，这时方法就无法使用它遗忘的原始的具体类型了。当使用 trait 的泛型类型参数被放入具体类型参数时也是如此：这个具体的类型就成了实现该 trait 的类型的一部分。一旦这个类型因使用 trait 对象而被擦除掉了之后，就无法知道放入泛型类型参数的类型是什么了。

一个 trait 的方法不是对象安全的例子是标准库中的 `Clone` trait。`Clone` trait 的 `clone` 方法的参数签名看起来像这样：

```rust
pub trait Clone {
    fn clone(&self) -> Self;
}
```

`String` 实现了 `Clone` trait，当在 `String` 实例上调用 `clone` 方法时会得到一个 `String` 实例。类似的，当调用 `Vec` 实例的 `clone` 方法会得到一个 `Vec` 实例。`clone` 的签名需要知道什么类型会代替 `Self`，因为这是它的返回值。

如果尝试在像列表 17-3 中 `Draw` 那样的 trait 上实现 `Clone`，就无法知道 `Self` 将会是 `Button`、`SelectBox` 亦或是将来会实现 `Draw` trait 的其他什么类型。

如果尝试做一些违反有关 trait 对象但违反对象安全规则的事情，编译器会提示你。例如，如果尝试实现列表 17-4 中的 `Screen` 结构体来存放实现了 `Clone` trait 而不是 `Draw` trait 的类型，像这样：

```rust,ignore
pub struct Screen {
    pub components: Vec<Box<Clone>>,
}
```

将会得到如下错误：

```
error[E0038]: the trait `std::clone::Clone` cannot be made into an object
 -->
  |
2 |     pub components: Vec<Box<Clone>>,
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `std::clone::Clone` cannot be
  made into an object
  |
  = note: the trait cannot require that `Self : Sized`
```

<!-- If we are including this section, we would explain how to fix this
problem. It involves adding another trait and implementing Clone manually for
that trait. Because this section is getting long, I stopped because it feels
like we're off in the weeds with an esoteric detail that not everyone will need
to know about. /Carol -->
