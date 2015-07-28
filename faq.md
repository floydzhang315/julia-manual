# 常见问题

## 会话和 REPL

### 如何删除内存中的对象？

Julia 没有 MATLAB 的 ``clear`` 函数；在 Julia 会话（准确来说，``Main`` 模块）中定义了一个名字的话，它就一直在啦。

如果你很关心内存使用，你可以用占内存的小的来替换大的。例如，如果 ``A`` 是个你不需要的大数组，可以先用 ``A = 0`` 来释放内存。下一次进行垃圾回收的时候，内存就会被释放了；你也可以直接调用 ``gc()`` 来回收。

### 如何在会话中修改 type/immutable 的声明？

有时候你定义了一种类型但是后来发现你需要添加一个新的域。当你尝试在 REPL 里这样做时就会出错

```
    ERROR: invalid redefinition of constant MyType
```

``Main`` 模块里的类型不能被重新定义。

当你在开发新代码时这会变得极其不方便，有一个很好的办法来处理。模块是可以用重新定义的办法来替换，所以把你的所有的代码封装在一个模块里就能够重新定义类型以及常数。你不能把类型名导入到 ``Main`` 里再去重新定义，但是你可以用模块名来解决这个问题。换句话说，当你开发的时候可以用这样的工作流

```
    include("mynewcode.jl")              # this defines a module MyModule
    obj1 = MyModule.ObjConstructor(a, b)
    obj2 = MyModule.somefunction(obj1)
    # Got an error. Change something in "mynewcode.jl"
    include("mynewcode.jl")              # reload the module
    obj1 = MyModule.ObjConstructor(a, b) # old objects are no longer valid, must reconstruct
    obj2 = MyModule.somefunction(obj1)   # this time it worked!
    obj3 = MyModule.someotherfunction(obj2, c)
    ...
```

## 函数

### 我把参数 ``x`` 传递给一个函数， 并在函数内修改它的值， 但是在函数外 ``x`` 的值并未发生变化， 为什么呢？

假设你像这样调用函数:

```
julia> x = 10
julia> function change_value!(y) # Create a new function
           y = 17
       end
julia> change_value!(x)
julia> x # x is unchanged!
10
```

在 Julia 里， 所有的函数(包括 ``change_value!()``) 都不能修改局部变量的所属的类。如果 ``x`` 被函数调用时被定义为一个不可变的对象(比如实数), 就不能修改; 同样地，如果 ``x`` 被定义为一个 ``Dict`` 对象，你不能把它改成 ASCIIString。但是需要主要的是: 假设 ``x`` 是一个数组(或者任何可变类型)。 你不能让 ``x`` 不再代表这个数组，但由于数组是可变的对象，你能修改数组的元素:

```
	julia> x = [1,2,3]
	3-element Array{Int64,1}:
	1
	2
	3

	julia> function change_array!(A) # Create a new function
	           A[1] = 5
	       end
	julia> change_array!(x)
	julia> x
	3-element Array{Int64,1}:
	5
	2
	3
```

这里我们定义了函数 ``change_array!()``, 把整数 ``5`` 分配给了数组的第一个元素。 当我们把 ``x`` 传读给这个函数时，注意到 ``x`` 依然是同一个数组，只是数组的元素发生了变化。

### 我能在函数中使用 ``using`` 或者 ``import`` 吗？

不行，在函数中不能使用 ``using`` 或 ``import``。如果你要导入一个模块但只是在某些函数里使用，你有两种方案::

1. 使用 ``import`` 

```        
        import Foo
        function bar(...)
            ... refer to Foo symbols via Foo.baz ...
        end
```

2. 把函数封装到模块里:

```
        module Bar
        export bar
        using Foo
        function bar(...)
            ... refer to Foo.baz as simply baz ....
        end
        end
        using Bar
```

## 类型，类型声明和构造方法

### 什么是“类型稳定”？

这意味着输出的类型是可以由输入类型预测出来。特别地，这表示输出的类型不能因输入的值的变化而变化。下面这段代码 *不是* 类型稳定的

```
    function unstable(flag::Bool)
        if flag
            return 1
        else
            return 1.0
        end
    end
```

这段代码视参数的值的不同而返回一个 ``Int`` 或是 ``Float64``。 因为 Julia 无法在编译时预测函数返回值类型，任何使用这个函数的计算都得考虑这两种可能的返回类型，这样很难生成快速的机器码。

### 为什么看似合理的运算 Julia 还是返回 ``DomainError``?

有些运算数学上讲得通但是会产生错误:

```
    julia> sqrt(-2.0)
    ERROR: DomainError
     in sqrt at math.jl:128

    julia> 2^-5
    ERROR: DomainError
     in power_by_squaring at intfuncs.jl:70
     in ^ at intfuncs.jl:84
```

这时由类型稳定造成的。对于 ``sqrt``， 大多数用户会用 ``sqrt(2.0)`` 得到一个实数而不是得到一个复数 ``1.4142135623730951 + 0.0im``。 也可以把 ``sqrt`` 写成当参数为负的时候返回复数，但是这将不再是 [类型稳定](http://julia-cn.readthedocs.org/zh_CN/latest/manual/faq/#man-type-stable)而且 ``sqrt`` 会变的很慢。

在这些情况下，你可以选择 *输入类型* 来得到想要的 *输出类型* :

```
    julia> sqrt(-2.0+0im)
    0.0 + 1.4142135623730951im

    julia> 2.0^-5
    0.03125
```

### Julia 为什么使用本机整数运算？

Julia 会应用机器运算的整数计算。这意味着 int 值的范围是有界的，是在两界之间取值的，所以添加，减去，乘以和除以一个整数都可能导致上溢或下溢，这可能会导致一些不好的后果，这种情况在一开始会让人感到很不安。

```
    julia> typemax(Int)
    9223372036854775807

    julia> ans+1
    -9223372036854775808

    julia> -ans
    -9223372036854775808

    julia> 2*ans
    0
```

显然，这远远不能用数学的方法来表现，您可能会认为 Julia 与一些高级编程语言会公开给用户这一情况相比来说不是那么理想。然而这对于效率和透明度都非常珍贵的数值工作来说，相比之下，替代品更是糟糕。

这里有一个选择是来检查每个整数操作的溢出情况，并且由于溢出情况而提高结果值到大一些的整数类型，例如 ``Int128`` 或 ``BigInt``。不幸的是，这就引进了在每个整数操作上都会有的主要负担（想想增加一个循环计数器） - 这需要发射代码在算术指令后执行程序时的溢出检查，并且需要一些分支来解决潜在溢出问题。更糟糕的是，这会导致每一个计算，在涉及整数时都是不稳定的。正如我们上面提到的，[类型的稳定性](http://julia-cn.readthedocs.org/zh_CN/latest/manual/faq/#man-type-stable)是有效的代码生成的关键。如果您不能指望整数运算的结果是整数，那么按 C 和 Fortran 编译器方式做的简单代码，想要生成速度快是不可能的。

这个方法还有一个可以避免不稳定类型外观的变化，就是把 ``Int`` 和 ``BigInt`` 合并成一个单一的混合整数类型，当结果不再适合机器整数的大小时，可以由内部改变来表示。然而这只是表面上解决了 Julia 语言的不稳定性水平问题，它也仅仅只是通过强硬地把所有相同的难题汇于 C 语言，使混合整数类型可以成功实现的方式，解决了几个小问题而已。这种方法基本上可以进行工作，甚至可以在许多情况下可以作出相当快的反应，但是还是有几个缺点的。其中一个问题是，在内存中整数和整数数组的表示方法，不再和 C，Fortran 等其它具有本地机器整数的语言的本地表示方法一一对应了。因此，对这些语言进行互操作，我们无论如何最终都需要引入本地的整数类型。任何无界表示的整数都没有一个固定的位，因此它们不能内联地被存储在有固定大小的槽的数组里，较大的整数的值会一直需要单独的堆分配来进行存储。当然，不管一个混合整数的实现有多精妙，总会有性能陷阱的情况或是性能下降的情况。复杂的表示的话，缺乏与 C 和 Fortran 语言的互操作性，不能代表没有额外堆存储的整数数组，并且不可预知的性能特点使即使最精妙的混合整数来实现高性能计算的工作不管怎样都不是个好办法。

还有一个在使用混合整数或是使其提高到 BigInts 的选择是用饱和的整数运算实现的，这个运算使即使把一个数添加到最大的整数值，值也不会变，同样的，从最小的整数值减去数值，值也不变。这恰恰就是 Matlab™ 可以实现的。

```
    >> int64(9223372036854775807)

    ans =

      9223372036854775807

    >> int64(9223372036854775807) + 1

    ans =

      9223372036854775807

    >> int64(-9223372036854775808)

    ans =

     -9223372036854775808

    >> int64(-9223372036854775808) - 1

    ans =

     -9223372036854775808
```

乍一看，这似乎很合理,因为 922337203685477580 是比 -922337203685477580 更要接近 922337203685477580 的，并且整数还是表现在一种用 C 语言和 Fortran 语言兼容的固定大小实现的本地的方式。然而，饱和的整数运算，是非常有问题的。首先的和最明显的问题是，它不是机器的整数算术操作方式，所以每台机器进行整数运算来检查下溢或上溢，并且用 typemin（int）或 typemax（int） 适当地取代结果之后，才可以实现发出饱和操作需要发出的指令。这就单独将每一个整数运算从一个单一的、快速的指令扩展到 6 个指令，还可能包括分支。但它会变得更糟–饱和的整数算术并不是联想的。来考虑这个 MATLAB 计算：

```
    >> n = int64(2)^62
    4611686018427387904

    >> n + (n - 1)
    9223372036854775807

    >> (n + n) - 1
    9223372036854775806
```

这使得它很难写很多基本的整数算法，因为很多常见的技术依赖于这样一个事实，即机器加成与溢出是联想的。考虑在 Julia 中利用 (lo + hi) >>> 1 表达式来找到整数值 lo 和 hi 的中间点：

```
    julia> n = 2^62
    4611686018427387904

    julia> (n + 2n) >>> 1
    6917529027641081856
```

看见了吗？没有问题。这是 2^62 和 2^63 之间正确的中点，尽管 ``n＋2n`` 实际应是 - 461168601842738790。现在尝试在 MATLAB 中：

```
    >> (n + 2*n)/2

    ans =

      4611686018427387904
```

这就出错了。添加一个  a >>> 运算元到 Matlab 上并不会有帮助。因为添加 n 和 2n 已经破坏了必要的计算正确的中点的信息时，饱和就发生了。

这不仅是程序员缺乏结合性而不幸不能依赖这样的技术,而且还打败几乎任何编译器可能想做的优化整数运算。例如，由于 Julia 的整数使用正常的机器整数运算，LLVM 是自由的积极简单的优化小函数如 f（k）= 5k-1。这个函数的机器码就是这样的：

```
    julia> code_native(f,(Int,))
        .section    __TEXT,__text,regular,pure_instructions
    Filename: none
    Source line: 1
        push    RBP
        mov RBP, RSP
    Source line: 1
        lea RAX, QWORD PTR [RDI + 4*RDI - 1]
        pop RBP
        ret
```

函数的实际体是一个单一的 ``lea`` 指令，计算整数时立刻进行乘，加运算。当 f 被嵌入另一个函数时，更加有利处：

```
    julia> function g(k,n)
             for i = 1:n
               k = f(k)
             end
             return k
           end
    g (generic function with 2 methods)

    julia> code_native(g,(Int,Int))
        .section    __TEXT,__text,regular,pure_instructions
    Filename: none
    Source line: 3
        push    RBP
        mov RBP, RSP
        test    RSI, RSI
        jle 22
        mov EAX, 1
    Source line: 3
        lea RDI, QWORD PTR [RDI + 4*RDI - 1]
        inc RAX
        cmp RAX, RSI
    Source line: 2
        jle -17
    Source line: 5
        mov RAX, RDI
        pop RBP
        ret
```

由于 ``f`` 调用被内联，循环体的结束时只是一个单一的 ``lea`` 指令。接下来，如果我们使循环迭代次数固定，我们可以来考虑发生了什么：

```
    julia> function g(k)
             for i = 1:10
               k = f(k)
             end
             return k
           end
    g (generic function with 2 methods)

    julia> code_native(g,(Int,))
        .section    __TEXT,__text,regular,pure_instructions
    Filename: none
    Source line: 3
        push    RBP
        mov RBP, RSP
    Source line: 3
        imul    RAX, RDI, 9765625
        add RAX, -2441406
    Source line: 5
        pop RBP
        ret
```

因为编译器知道整数的加法和乘法之间的联系并且乘法分配时优先级会高于除法 – 这两者都是真正的饱和运算 –  它们可以优化整个回路使之只留下来的只是乘法和加法。饱和算法完全地打败了这种最优化，这是因为结合性和分配性在每次在循环迭代都可能会失败，而所导致的不同后果取决于在哪次迭代会失败。

饱和整数算法只是一个真的很差的语言语义学选择的例子，它可以阻止所有有效的性能优化。在 C 语言编程中有很多事情是很难的，但整数溢出并不是其中之一，特别是在 64  位系统中。比如如果我用的整数可能会变得比 2^63-1 还要大，我可以很容易地预测到。您要问自己我是在遍历存储在计算机中的实际的东西么？之后我就可以确认数是不会变得那么大的。这点是可以保证的，因为我没那么大的存储空间。我是真的在数实际真实存在的东西么？除非它们是宇宙中的沙子或原子粒，否则2^63-1 已经足够大了。我是在计算阶乘么？之后就可以确认，它们可能变得特别大-我就应该用 BigInt 了。看懂了么？区分起来是很简单的。

### 类型的“抽象的”或者不明确的域如何与编译器进行交互？

类型可以在不指定字段的类型的情况下声明：

```
    julia> type MyAmbiguousType
               a
           end
```

这允许 `a` 是任何类型。这通常是非常有用的，但它有一个缺点：对于 `MyAmbiguousType` 类型的对象，编译器将无法生成高效的代码。原因是编译器使用对象的类型而不是值来决定如何构建代码。不幸的是，`MyAmbiguousType` 类型只能推断出很少的信息：

```
    julia> b = MyAmbiguousType("Hello")
    MyAmbiguousType("Hello")

    julia> c = MyAmbiguousType(17)
    MyAmbiguousType(17)

    julia> typeof(b)
    MyAmbiguousType (constructor with 1 method)

    julia> typeof(c)
    MyAmbiguousType (constructor with 1 method)
```

`b` 和 `c` 有着相同的类型，但是它们在内存中数据的基础表示是非常不同的。即使您只在 `a` 的域中储存数值，事实上 `Uint8` 和 `Float64` 的内存表示不同也意味着 CPU 需要用两种不同的指令来处理它们。由于类型中的所需信息是不可用，于是这样的决定不得不在运行时作出。这减缓了性能。  

您可以用声明 `a` 的类型的方法做得更好。在这里，我们注意到这样一种情况，就是 `a` 可能是几个类型中的任意一种，在这种情况下自然的解决办法是使用参数。例如：

```
    julia> type MyType{T<:FloatingPoint}
             a::T
           end
```

这相对以下代码是一个更好的选择

```
    julia> type MyStillAmbiguousType
             a::FloatingPoint
           end
```

因为第一个版本指定了包装对象的类型。例如：

```
    julia> m = MyType(3.2)
    MyType{Float64}(3.2)

    julia> t = MyStillAmbiguousType(3.2)
    MyStillAmbiguousType(3.2)

    julia> typeof(m)
    MyType{Float64} (constructor with 1 method)

    julia> typeof(t)
    MyStillAmbiguousType (constructor with 2 methods)
```

`a` 的域的类型可以轻而易举地由 `m` 的类型确定，但不是从 `t` 的类型确定。事实上，在 `t` 中是可以改变 `a` 的域的类型的：

```
    julia> typeof(t.a)
    Float64

    julia> t.a = 4.5f0
    4.5f0

    julia> typeof(t.a)
    Float32
```

相反，一旦 `m` 被构造，`m.a` 的类型就不能改变了：

```
    julia> m.a = 4.5f0
    4.5

    julia> typeof(m.a)
    Float64
```

`a` 的类型可以从 `m` 的类型知道的事实和 `m.a` 的类型不能在函数中修改的事实允许编译器为像 `m` 那样的类而不是像 `t` 那样的类生成高度优化的代码。

当然，只有当我们用具体类型来构造 `m` 时，这一切才是真实的。我们可以通过明确地用抽象类构造它的方法来打破之一点：

```
    julia> m = MyType{FloatingPoint}(3.2)
    MyType{FloatingPoint}(3.2)

    julia> typeof(m.a)
    Float64

    julia> m.a = 4.5f0
    4.5f0

    julia> typeof(m.a)
    Float32
```

对于一切实际目的，这些对象对 `MyStillAmbiguousType` 的行为相同。

对比一个简单程序所产生的全部代码是很有意义的：

```
    func(m::MyType) = m.a+1
```

使用：

```
    code_llvm(func,(MyType{Float64},))
    code_llvm(func,(MyType{FloatingPoint},))
    code_llvm(func,(MyType,))
```

由于长度的原因，结果并没有在这里显示，但您不妨自己尝试一下。因为在第一种情况下，该类型是完全指定的，编译器不需要在运行时生成任何代码来解决类型的问题。这就会有更短的代码更快的编码速度。

### 如何声明“抽象容器类型”的域

与应用在[上一章节](http://julia-cn.readthedocs.org/zh_CN/latest/manual/faq/#man-abstract-fields)中的最好的相同例子在容器类型中也适用：

```
    julia> type MySimpleContainer{A<:AbstractVector}
             a::A
           end

    julia> type MyAmbiguousContainer{T}
             a::AbstractVector{T}
           end
```

例如：

```
    julia> c = MySimpleContainer(1:3);

    julia> typeof(c)
    MySimpleContainer{UnitRange{Int64}} (constructor with 1 method)

    julia> c = MySimpleContainer([1:3]);

    julia> typeof(c)
    MySimpleContainer{Array{Int64,1}} (constructor with 1 method)

    julia> b = MyAmbiguousContainer(1:3);

    julia> typeof(b)
    MyAmbiguousContainer{Int64} (constructor with 1 method)

    julia> b = MyAmbiguousContainer([1:3]);

    julia> typeof(b)
    MyAmbiguousContainer{Int64} (constructor with 1 method)
```

对于 `MySimpleContainer`，对象是由其类型和参数完全指定的，所以编译器可以生成优化的功能。在大多数情况下，这可能就足够了。

虽然现在编译器可以完美地完成它的工作，但某些时候您可能希望您的代码能够根据 `a` 的*元素类型*做出不同的东西。通常，达到这一点最好的方法是把您的具体操作（这里是 `foo`）包在一个单独的函数里：

```
    function sumfoo(c::MySimpleContainer)
        s = 0
	for x in c.a
	    s += foo(x)
	end
	s
    end

    foo(x::Integer) = x
    foo(x::FloatingPoint) = round(x)
```

这在允许编译器在所有情况下都生成优化的代码，同时保持做起来很简单。

然而，有时候您需要根据 `a` 的不同的元素类型来声明外部函数的不同版本。您可以像这样来做：

```
    function myfun{T<:FloatingPoint}(c::MySimpleContainer{Vector{T}})
        ...
    end
    function myfun{T<:Integer}(c::MySimpleContainer{Vector{T}})
        ...
    end
```

这对于 `Vector{T}` 来讲不错，但是我们也要给 `UnitRange{T}` 或其他抽象类写明确的版本。为了防止这样单调乏味的情况，您可以在 `MyContainer` 的声明中来使用两个变量：

```
    type MyContainer{T, A<:AbstractVector}
        a::A
    end
    MyContainer(v::AbstractVector) = MyContainer{eltype(v), typeof(v)}(v)

    julia> b = MyContainer(1.3:5);

    julia> typeof(b)
    MyContainer{Float64,UnitRange{Float64}}
```

请注意一个有点令人惊讶的事实，`T` 没有在 `a` 的域中声明，一会之后我们将会回到这一点。用这种方法，一个人可以编写像这样的函数：

```
    function myfunc{T<:Integer, A<:AbstractArray}(c::MyContainer{T,A})
        return c.a[1]+1
    end
    # Note: because we can only define MyContainer for
    # A<:AbstractArray, and any unspecified parameters are arbitrary,
    # the previous could have been written more succinctly as
    #     function myfunc{T<:Integer}(c::MyContainer{T})

    function myfunc{T<:FloatingPoint}(c::MyContainer{T})
        return c.a[1]+2
    end

    function myfunc{T<:Integer}(c::MyContainer{T,Vector{T}})
        return c.a[1]+3
    end

    julia> myfunc(MyContainer(1:3))
    2

    julia> myfunc(MyContainer(1.0:3))
    3.0

    julia> myfunc(MyContainer([1:3]))
    4
```

正如您所看到的，用这种方法可以既专注于元素类型 `T` 也专注于数组类型 `A`。

然而还剩下一个问题：我们没有强制使 `A` 包括元素类型 `T`，所以完全有可能构造这样一个对象：

```
  julia> b = MyContainer{Int64, UnitRange{Float64}}(1.3:5);

  julia> typeof(b)
  MyContainer{Int64,UnitRange{Float64}}
```

为了防止这一点，我们可以添加一个内部构造函数：

```
    type MyBetterContainer{T<:Real, A<:AbstractVector}
        a::A

        MyBetterContainer(v::AbstractVector{T}) = new(v)
    end
    MyBetterContainer(v::AbstractVector) = MyBetterContainer{eltype(v),typeof(v)}(v)


    julia> b = MyBetterContainer(1.3:5);

    julia> typeof(b)
    MyBetterContainer{Float64,UnitRange{Float64}}

    julia> b = MyBetterContainer{Int64, UnitRange{Float64}}(1.3:5);
    ERROR: no method MyBetterContainer(UnitRange{Float64},)
```

内部构造函数要求 `A` 的元素类型为 `T`。

## 无和缺值

### Julia 中的“空（null）”和“无（nothingness）”如何工作？

不像许多其他语言（例如，C 和 Java）中的那样，Julia 中没有“空（null）”值。当引用（变量，对象的域，或者数组元素）是未初始化的，访问它就会立即抛出一个错误。这种情况可以通过 `isdefined` 函数检测。

有些函数只用于其副作用，不需要返回值。在这种情况下，惯例返回 `nothing`，它只是一个 `Nothing` 类型的对象。这是一个没有域的普通类型；它除了这个惯例之外，没有什么特殊的，并且 REPL 不会为它打印任何东西。一些不能有值的语言结构也统一为 `nothing`，例如 `if false; end`。

注意 `Nothing`（大写）是 `nothing` 的类型，并且只应该用在一个类型被需求环境中（例如一个声明）。

您可能偶尔看到 `None`，这是完全不同的。它是空（empty，或是“底” bottom）类型，一类没有值也没有子类型（subtypes，除了它本身）的类型。您一般不需要使用这种类型。

空元组（`()`）是另一种类型的无。但是它不应该真的被认为是什么都没有而是一个零值的元组。

## Julia 发行版

### 我想要使用一个 Julia 的发行版本（release），测试版（beta），或者是夜间版（nightly version）？

如果您想要一个稳定的代码基础，您可能更倾向于 Julia 的发行版本。一般情况下每 6 个月发布一次，给您一个稳定的写代码平台。

如果您不介意稍稍落后于最新的错误修正和更改的话，但是发现更具有吸引力的更改的更快一点的速度，您可能更喜欢 Julia 测试版本。此外，这些二进制文件在发布之前进行测试，以确保它们是具有完全功能的。

如果您想利用语言的最新更新，您可能更喜欢使用 Julia 的夜间版本，并且不介意这个版本偶尔不工作。

最后，您也可以考虑从源头上为自己建造 Julia。此选项主要是对那些对命令行感到舒适或对学习感兴趣的个人。如果这描述了您，您可能也会感兴趣在阅读我们[指导方针](https://github.com/JuliaLang/julia/blob/master/CONTRIBUTING.md)。

这些下载类型的链接可以在下载页面 [http://julialang.org/downloads/](http://julialang.org/downloads/) 找到。请注意，并非所有版本的 Julia 都可用于所有平台。

### 何时移除舍弃的函数？

过时的功能在随后的发行版本之后去除。例如，在 0.1 发行版本中被标记为过时的功能将不会在 0.2 发行版本中使用。

## 开发 Julia

### 我要如何调试 Julia 的 C 代码？（从一个像是 gdb 的调试器内部运行 Julia REPL）

首先您应该用 `make debug` 构建 Julia 调试版本。下面，以`(gdb)`开头的行意味着您需要在 gdb  prompt 下输入。

### 从 shell 开始

主要的挑战是 Julia 和 gdb 都需要有它们自己的终端，来允许您和它们交互。一个方法是使用 gdb 的 `attach` 功能来调试一个已经运行的 Julia session。然而，在许多系统中，您需要使用根访问（root access）来使这个工作。下面是一个可以只使用用户级别权限来实现的方法。

第一次做这种事时，您需要定义一个脚本，在这里被称为 `oterm`，包含以下几行：

```
    ps
    sleep 600000
```

让它用 `chmod +x oterm` 执行。

现在：

- 从一个 shell（被称为 shell 1）开始，类型 `xterm -e oterm &`。您会看到一个新的窗口弹出，这将被称为终端 2。
- 从 shell 1 之内，`gdb julia-debug`。您将会在 `julia/usr/bin` 里找到这个可执行文件。
- 从 shell 1 之内，`(gdb) tty /dev/pts/#` 里面的 `#` 是在 terminal 2 中 `pts/` 之后显示的数字。
- 从 shell 1 之内，`(gdb) run`
- 从 terminal 2 之内，在 Julia 中发布任何准备好的您需要的命令来进入您想调试的步骤
- 从 shell 1 之内，按 Ctrl-C
- 从 shell 1 之内，插入您的断点，例如 `(gdb) b codegen.cpp:2244`
- 从 shell 1 之内，`(gdb) c` 来继续 Julia 的执行
- 从 terminal 2 之内，发布您想要调试的命令，shell 1 将会停在您的断点处。

### 在 emacs 之内

- `M-x gdb`，然后进入 `julia-debug`（这可以最简单的从 julia/usr/bin 找到，或者您可以指定完整路径）
- `(gdb) run`
- 现在您将会看到 Julia prompt。在 Julia 中运行任何您需要的命令来达到您想要调试的步骤。
- 在 emacs 的 “Signals” 菜单下选择 BREAK——这将会使您返回到 `(gdb)` prompt
- 设置一个断点，例如，`(gdb) b codegen.cpp:2244`
- 通过 `(gdb) c` 返回到 Julia prompt
- 执行您想要运行的 Julia 命令。
