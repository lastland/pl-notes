---
layout: post
title: "Lightweight Modular Staging"
description: ""
category:
tags: [Scala, Multi-stage Programming]
---
{% include JB/setup %}

* toc
{:toc}

本文是 [Lightweight Modular Staging: A Pragmatic Approach to Runtime Code Generation and Compiled DSLs](http://infoscience.epfl.ch/record/150347/files/gpce63-rompf.pdf) (2010) 一文的阅读笔记。

# 导言

之前我们已经介绍过 [Multi-stage Programming](../../../08/05/1)，这一概念可以让我们以 general 的形式编写出可以在运行时被特化的代码。最经典的例子就是 `power` 函数，我们希望以递归的形式编写代码，但希望最后得到的代码是像 `x * x * x` 这样的而非一层又一层递归或者函数调用。

在 MetaOCaml 或者其他的支持 multi-stage programming 的语言当中，我们可以通过 *brackets* 和 *escape* 两种操作来编写一个可以生成特化函数的函数。譬如在 OCaml 中，一个 `power` 函数可以这样实现：

{% highlight ocaml %}
    let rec power (n, x) =
        match n with
         0 -> 1 | n -> x * (power (n-1, x));;
{% endhighlight %}

使用 MetaOCaml 可以这样编写一个形式上类似的函数：

{% highlight ocaml %}
    let rec power (n, x) =
       match n with
         0 -> .<1>. | n -> .<.~x * .~(power (n-1, x))>.;;
{% endhighlight %}

对这两个函数的详细解释可以参看[我之前的笔记](../../../08/05/1)。

尽管 MetaOCaml 这种实现看上去已经非常简洁了，但是 EPFL 的 Programming Methods Group [^1] 认为在 Scala 中这一任务可以变得更加简单，且更加 modular，所以他们提出了这个 Lightweight Modular Staging （为了简便，下文中将简称 LMS）。

还是以 `power` 函数为例，在 Scala 中我们可以这样实现：

{% highlight scala %}
    def power(b: Double, x: Int): Double =
        if (x == 0) 1.0 else b * power(b, x - 1)
{% endhighlight %}

而当想要编写 multi-stage 版本时，利用 LMS，我们只需要把上述代码改成下面这幅样子：

{% highlight scala %}
    trait PowerA { this: Arith =>
        def power(b: Rep[Double], x: Int): Rep[Double] =
            if (x == 0) 1.0 else b * power(b, x - 1) }
{% endhighlight %}

请注意新版本的 `power` 函数的函数体内部的变化是……没有任何变化！LMS 版本涉及到的改变总共只有两个：一，函数的参数 `b` 和返回值的类型从 `Double` 变成了 `Rep[Double]`；二，函数外面被包了一个 trait，且这个 trait 要求所有使用该 trait 的类都必须同时使用 `Arith` 这个 trait 或者它的 `subtrait`。

这两种变化的第一点所蕴含的思想便是用类型而非其他特殊操作符来区分不同的 stage，因此 LMS 可以作为一门表达能力足够强的编程语言的的 library 来实现，不需要涉及编译器的改动 [^2]，因此称得上是 **lightweight**；第二点则使得使用 `PowerA` 这一 trait 的用户不一定非要使用 `Arith`，他可以使用 `Arith` 的任何子类，而这些子类的实现可能各不一样，或各有新的优化，用户可以自己选择、组合想要使用的实现或优化，甚至自己动手编写新的，因此称得上是 **modular**。

利用这一方法实现的 multi-stage 代码还可以被保证也是类型安全的——如果存在类型错误，会在编译时就报错，不需要覆盖各个分支的运行测试。

除此之外，LMS 本身还包含了一些 multi-stage 中常用的优化，譬如公共子表达式的消除，以方便这类代码的编写。

[^1]: 就是 Scala 的原作者 Martin Odersky 的小组。这篇论文的两个作者（二作就是 Martin 本尊）都来自这一小组，且参与了 Scala 的开发。第一作者还实现了一个叫做 Scala-Virtualized 的项目，我们后文中也会提到。

[^2]: 但对 Scala 来说，这还不是真的。因为尽管 Scala 表达力已经很强，但它不支持对一些特殊操作符譬如 `if`, `val` 的重载，而且不支持我们将在后文中看到的那种方便的定义中缀操作符的方法，所以实际上 LMS 是在另一个版本的 Scala 编译器，即 Scala-Virtualized 当中实现的。

# Lightweight Modualr Staging

前面我们已经看到了用 LMS 实现的 multi-stage 版本的 `power` 函数，为了方便查看，这里我们再把它列在这里：

{% highlight scala %}
    trait PowerA { this: Arith =>
        def power(b: Rep[Double], x: Int): Rep[Double] =
            if (x == 0) 1.0 else b * power(b, x - 1) }
{% endhighlight %}

## 坏的实现：字符串

形式已经有了，那么接下来怎么实现呢？一种最直观也是最 naive 的实现如下：

{% highlight scala %}
    trait BaseStr extends Base {
        type Rep[+T] = String
    }

    trait ArithStr extends Arith with BaseStr {
        implicit def unit(x: Double) = x.toString
        def infix_+(x: String, y: String) = "(%s+%s)".format(x,y)
        def infix_*(x: String, y: String) = "(%s*%s)".format(x,y)
        ...
    }
{% endhighlight %}

我们先无视 `Base` 这个东西。在 `BaseStr` 中我们设定了一个 type alias，也就是所有的 `Rep[T]` 其实都是 `String` [^3]。

在 `ArithStr` 中我们首先定义了一个隐式转换。这是因为我们希望直接像 `power(2.0, 3)` 这样调用函数，但 `power` 函数要求第一个参数的类型必须是 `Rep[Double]` 而非 `Double`。在 Scala 中，我们可以定义这样一个隐式转换，而后编译器会自动在碰到需要 `Rep[Double]` 而得到的却是 `Double` 时把这个函数调用放进去——这一过程也是在编译时进行的。因为这里 `Rep[Dounle]` 其实就是 `String`，所以只要调用 `toString` 方法就行了。

然后在 `ArithStr` 中我们又定义了两个中缀操作符，编译器在碰到两个 `String` 的加法时会自动调用 `infix_+` 这一方法，并得到另一个 `String` 的结果。乘法同理。

好了，这样寥寥几行代码我们就有了一段可以生成代码的代码。尽管我们还没有实现编译这块的工作，但不妨先看看这样的实现生成的代码质量如何。我们通过下面的方式调用我们的 `PowerA`：

{% highlight scala %}
    new PowerA with ArithStr {
        println {
            power("(x0+x1)",4)
        }
    }
{% endhighlight %}

运行的结果如下：

{% highlight scala %}
    ((x0+x1)*((x0+x1)*((x0+x1)*((x0+x1)*1.0))))
{% endhighlight %}

很遗憾，这是一个看上去很蠢的结果，即使不论最后多余的 `*1.0`，大量重复的 `x0+x1` 实在难以称得上好的代码。这是因为我们选择了使用字符串来实现这些东西，简单粗暴的同时我们失去了对代码进行优化的可能。

[^3]: 这里的 `+` 表示 covariant type，这里我们不多做解释。我可能会在将来更新一篇关于这一概念的笔记。

## 好的实现：图

由于上述原因，实际的 LMS 在实现时采用了图作为代码的中间表示手段。下面是用图来实现这一方法的代码示例：

{% highlight scala %}
    trait Expressions {
        // expressions (atomic)
        abstract class Exp[+T]
        case class Const[T](x: T) extends Exp[T]
        case class Sym[T](n: Int) extends Exp[T]

        def fresh[T]: Sym[T]

        // definitions (composite, subclasses provided
        // by other traits)
        abstract class Def[T]

        def findDefinition[T](s: Sym[T]): Option[Def[T]]
        def findDefinition[T](d: Def[T]): Option[Sym[T]]
        def findOrCreateDefinition[T](d: Def[T]): Sym[T]

        // bind definitions to symbols automatically
        implicit def toAtom[T](d: Def[T]): Exp[T] =
            findOrCreateDefinition(d)

        // pattern match on definition of a given symbol
        object Def {
            def unapply[T](s: Sym[T]): Option[Def[T]] =
                findDefinition(s)
        }
    }
{% endhighlight %}

这里我们定义了一些基本的 Node 和一些基本操作。主义的是这里的一个叫做 `Def[T]` 的类，每一个该类型都对应了一个 `Sym[T]` 的类型，而后者是 `Exp[T]`，后面我们也将看到它也就是 `Rep[T]` 的真身。 `Sym[T]` 表示的是一个类型为 `T` 的符号；`toAtom` 则定义了一个 `Def[T]` 到符号的隐式转换，转换的规则是寻找当前是否存在和该 `Def[T]` 相关的符号，如果不存在，那么为它创建一个。这些和公共子表达式的消除这一功能密切相关。
下面我们继续看 `Arith` 的相关实现：

{% highlight scala %}
    trait BaseExp extends Base with Expressions {
        type Rep[+T] = Exp[T]
    }

    trait ArithExp extends Arith with BaseExp {
        implicit def unit(x: Double) = Const(x)
        case class Plus(x: Exp[Double], y: Exp[Double])
            extends Def[Double]
        case class Times(x: Exp[Double], y: Exp[Double])
            extends Def[Double]
        def infix_+(x: Exp[Double], y: Exp[Double]) = Plus(x, y)
        def infix_*(x: Exp[Double], y: Exp[Double]) = Times(x, y)
    }
{% endhighlight %}

`BaseExp` 和 `ArithExp` 首先声明了 `Rep[T]` 就是 `Exp[T]`，并定义了从 `Double` 到 `Exp[Double]` 的隐式转换规则。

接下来 `ArithExp` 定义了两个 Node，分别代表加法和乘法，同时它们也是加法和乘法这两个中缀操作符的运算结果。值得注意的是这两个 Node 是 `Def[Double]` 的子类，也就是说它们都有一个相关的 `Sym[Double]`，LMS 便是通过这种为计算结果命名，同时相同的计算结果使用相同的名字的方法来做到消除公共子表达的。

我们前面提过，LMS 的一个好处是，用户可以自由组合一些 trait 来对生成的代码进行优化，我们前面已经看到用字符串写的代码中有一个冗余的 `*1.0`，虽然无伤大雅，但我们不妨通过展示如何来消除这个强迫症的死敌来展示 LMS 的这一特性。我们可以编写如下函数：

{% highlight scala %}
    trait ArithExpOpt extends ArithExp {
        override def infix_*(x:Exp[Int],y:Exp[Int]) = (x,y) match {
            case (Const(x), Const(y)) => Const(x * y)
            case (x, Const(1)) => x
            case (Const(1), y) => y
            case _ => super.infix_*(x, y)
        }
    }
{% endhighlight %}

我们新的 trait `ArithExpOpt` 重载了乘法，对某些特定的 Node 进行了重写——因为我们使用图作为代码的中间表示形式，才使方便地进行重写优化成为可能。这里定义的重写规则很简单，当我们看到两个常数相乘时，就直接在生成代码时就把它们乘起来；如果两个因数中存在1，那么就省略掉1 [^4]。

下面我们来尝试下新的实现：

{% highlight scala %}
    trait PowerA2 extends PowerA { this: Compile =>
        val p4 = compile { x: Rep[Double] =>
            power(x + x, 4)
        }
        // use compiled function p4 ...
    }

    new PowerA2 with CompileScala
        with ArithExpOpt with ScalaGenArith
{% endhighlight %}

生成的代码如下：

{% highlight scala %}
    class Anon$1 extends ((Double)=>(Double)) {
        def apply(x0:Double): Double = {
            val x1 = x0+x0
            val x2 = x1*x1
            val x3 = x1*x2
            val x4 = x1*x3
            x4
        }
    }
{% endhighlight %}

我们确实成功消除了公共子表达式和冗余的常数1.

[^4]: 原文中第五行的代码为 `case (Const(1), y) => x`，我觉得应该是写错了。

## 其它功能

对于副作用、控制流、函数定义和递归、跨 stage 一致性的问题，请参见原论文。

# 脚注
