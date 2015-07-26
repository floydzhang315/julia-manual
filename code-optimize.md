
# 代码性能优化

以下几节将描述一些提高 Julia 代码运行速度的技巧。

## 避免全局变量


全局变量的值、类型，都可能变化。这使得编译器很难优化使用全局变量的代码。应尽量使用局部变量，或者把变量当做参数传递给函数。

对性能至关重要的代码，应放入函数中。

声明全局变量为常量可以显著提高性能：

```
const DEFAULT_VAL = 0
```

使用非常量的全局变量时，最好在使用时指明其类型，这样也能帮助编译器优化： 

```
global x
y = f(x::Int + 1)
```

写函数是一种更好的风格，这会产生更多可重复和清晰的代码，也包括清晰的输入和输出。


## 使用 ``@time`` 来衡量性能并且留心内存分配


衡量计算性能最有用的工具是 ``@time`` 宏. 下面的例子展示了良好的使用方式 :

```
  julia> function f(n)
             s = 0
             for i = 1:n
                 s += i/2
             end
             s
          end
  f (generic function with 1 method)

  julia> @time f(1)
  elapsed time: 0.008217942 seconds (93784 bytes allocated)
  0.5

  julia> @time f(10^6)
  elapsed time: 0.063418472 seconds (32002136 bytes allocated)
  2.5000025e11
```

在第一次调用时 (``@time f(1)``), ``f`` 会被编译. (如果你在这次会话中还
没有使用过 ``@time``, 计时函数也会被编译.) 这时的结果没有那么重要. 在
第二次调用时, 函数打印了执行所耗费的时间, 同时请注意, 在这次执行过程中
分配了一大块的内存. 相对于函数形式的 ``tic`` 和 ``toc``, 这是
``@time`` 宏的一大优势.

出乎意料的大块内存分配往往意味着程序的某个部分存在问题, 通常是关于类型
稳定性. 因此, 除了关注内存分配本身的问题, 很可能 Julia 为你的函数生成
的代码存在很大的性能问题. 这时候要认真对待这些问题并遵循下面的一些个建
议.

另外, 作为一个引子, 上面的问题可以优化为无内存分配 (除了向 REPL 返回结
果), 计算速度提升 30 倍 ::

```
  julia> @time f_improved(10^6)
  elapsed time: 0.00253829 seconds (112 bytes allocated)
  2.5000025e11
```

你可以从下面的章节学到如何识别 ``f`` 存在的问题并解决.

在有些情况下, 你的函数可能需要为本身的操作分配内存, 这样会使得问题变得
复杂. 在这种情况下, 可以考虑使用下面的 :ref:`工具
<man-performance-tools>` 之一来甄别问题, 或者将函数拆分, 一部分处理内存分
配, 另一部分处理算法 (参见 :ref:`预分配内存 <man-preallocation>`).



## 工具

Julia 提供了一些工具包来鉴别性能问题所在 :

- [profiling](http://julia-cn.readthedocs.org/zh_CN/latest/stdlib/profile/#stdlib-profiling) 可以用来衡量代码的性能, 同时鉴别出瓶颈所在.
  对于复杂的项目, 可以使用 `ProfileView
  <https://github.com/timholy/ProfileView.jl>` 扩展包来直观的展示分析
  结果.

- 出乎意料的大块内存分配, -- ``@time``, ``@allocated``, 或者
  -profiler - 意味着你的代码可能存在问题. 如果你看不出内存分配的问题,
  -那么类型系统可能存在问题. 也可以使用 ``--track-allocation=user`` 来
  -启动 Julia, 然后查看 ``*.mem`` 文件来找出内存分配是在哪里出现的.

- `TypeCheck <https://github.com/astrieanna/TypeCheck.jl`_ 可以帮助找
  出部分类型系统相关的问题. 另一个更费力但是更全面的工具是
  ``code_typed``. 特别留意类型为 ``Any`` 的变量, 或者 ``Union`` 类型.
  这些问题可以使用下面的建议解决.

- `Lint <https://github.com/tonyhffong/Lint.jl>`_ 扩展包可以指出程序一
  些问题.

## 避免包含一些抽象类型参数

当运行参数化类型时候，比如 arrays，如果有可能最好去避免使用抽象类型参数。
思考下面的代码：

```
a = Real[]    # typeof(a) = Array{Real,1}
if (f = rand()) < .8
    push!(a, f)
end
```

因为`a`是一个抽象类型`Real`的 array，所以可以包含任何`Real`类型的值。既然`Real`对象可以是任意的大小和结构，`a`必须被解释为一个array数组指向所有可能的对象。所以我们应该用确定的类型代替，比如`Float64`:

```
a = Float64[] # typeof(a) = Array{Float64,1}
```

这样会建立大小为 64 位的浮点值，也会更有效率。


## 类型声明

在 Julia 中，编译器能推断出所有的函数参数与局部变量的类型，因此声名变量类型不能提高性能。然而在有些具体实例中，声明类型还是非常有用的。

## 给复合类型做类型声明


假如有一个如下的自定义类型： 

```
type Foo
    field
end
```

编译器推断不出 ``foo.field`` 的类型，因为当它指向另一个不同类型的值时，它的类型也会被修改。这时最好声明具体的类型，比如 ``field::Float64`` 或者 ``field::Array{Int64,1}`` 。

### 显式声明未提供类型的值的类型


我们经常使用含有不同数据类型的数据结构，比如上述的 ``Foo`` 类型，或者元胞数组（ ``Array{Any}`` 类型的数组）。如果你知道其中元素的类型，最好把它告诉编译器： ::

```
function foo(a::Array{Any,1})
    x = a[1]::Int32
    b = x+1
    ...
end
```

假如我们知道 ``a`` 的第一个元素是 ``Int32`` 类型的，那就添加上这样的类型声明吧。如果这个元素不是这个类型，在运行时就会报错，这有助于调试代码。

### 显式声明命名参数的值的类型


命名参数可以显式指定类型::

```
function with_keyword(x; name::Int = 1)
    ...
end
```

函数只处理指定类型的命名参数，因此这些声明不会对该函数内部代码的性能产生影响。
不过，这会减少此类包含命名参数的函数的调用开销。

与直接使用参数列表的函数相比，命名参数的函数调用新增的开销很少，基本上可算是零开销。

如果传入函数的是命名参数的动态列表，例如``f(x; keywords...)``，速度会比较慢，性能敏感的代码慎用。


## 把函数拆开

把一个函数拆为多个，有助于编译器调用最匹配的代码，甚至将它内联。

举个应该把“复合函数”写成多个小定义的例子： 

```
function norm(A)
    if isa(A, Vector)
        return sqrt(real(dot(A,A)))
    elseif isa(A, Matrix)
        return max(svd(A)[2])
    else
        error("norm: invalid argument")
    end
end
```

如下重写会更精确、高效： 

```
norm(x::Vector) = sqrt(real(dot(x,x)))
norm(A::Matrix) = max(svd(A)[2])
```

## 写“类型稳定”的函数


尽量确保函数返回同样类型的数值。考虑下面定义： 

```
pos(x) = x < 0 ? 0 : x
```

尽管看起来没问题，但是 ``0`` 是个整数（ ``Int`` 型）， ``x`` 可能是任意类型。因此，函数有返回两种类型的可能。这个是可以的，有时也很有用，但是最好如下重写： ::

```
pos(x) = x < 0 ? zero(x) : x
```

Julia 中还有 ``one`` 函数，以及更通用的 ``oftype(x,y)`` 函数，它将 ``y`` 转换为与 ``x`` 同样的类型，并返回。这仨函数的第一个参数，可以是一个值，也可以是一个类型。

## 避免改变变量类型


在一个函数中重复地使用变量，会导致类似于“类型稳定性”的问题： 

```
function foo()
    x = 1
    for i = 1:10
        x = x/bar()
    end
    return x
end
```

局部变量 ``x`` 开始为整数，循环一次后变成了浮点数（ ``/`` 运算符的结果）。这使得编译器很难优化循环体。可以修改为如下的任何一种：

-  用 ``x = 1.0`` 初始化 ``x``
-  声明 ``x`` 的类型： ``x::Float64 = 1``
-  使用显式转换: ``x = one(T)``

## 分离核心函数


很多函数都先做些初始化设置，然后开始很多次循环迭代去做核心计算。尽可能把这些核心计算放在单独的函数中。例如，下面的函数返回一个随机类型的数组： 

```
function strange_twos(n)
    a = Array(randbool() ? Int64 : Float64, n)
    for i = 1:n
        a[i] = 2
    end
    return a
end
```

应该写成： 

```
function fill_twos!(a)
    for i=1:length(a)
        a[i] = 2
    end
end

function strange_twos(n)
    a = Array(randbool() ? Int64 : Float64, n)
    fill_twos!(a)
    return a
end
```

Julia 的编译器依靠参数类型来优化代码。第一个实现中，编译器在循环时不知道 ``a`` 的类型（因为类型是随机的）。第二个实现中，内层循环使用 ``fill_twos!`` 对不同的类型 ``a`` 重新编译，因此运行速度更快。

第二种实现的代码更好，也更便于代码复用。

标准库中经常使用这种方法。如 [abstractarray.jl](https://github.com/JuliaLang/julia/blob/master/base/abstractarray.jl)  文件中的 ``hvcat_fill`` 和 ``fill!`` 函数。我们可以用这两个函数来替代这儿的 ``fill_twos!`` 函数。

形如 ``strange_twos`` 之类的函数经常用于处理未知类型的数据。比如，从文件载入的数据，可能包含整数、浮点数、字符串，或者其他类型。

## 内存列中的访问数组


Julia 中的多维数组是根据以列为主的顺序存储的。这意味着每次数组都占据了一列。我们可以通过如下所示的 ``vec`` 功能或者是 或是排列 ``[:]`` 来进行验证（  注意到数组的顺序是 ``[1 3 2 4]`` 而不是 ``[1 2 3 4]`` ）：

``` 
julia> x = [1 2; 3 4]  
2x2 Array{Int64,2}:
 1  2
 3  4

julia> x[:]
4-element Array{Int64,1}:
 1
 3
 2
 4
```

这种给数组排序的约定在许多语言中都是常见的，比如 Fortran ， Matlab ， 和 R 语言(举几个例子来说)。以列为主序的另一选择就是以行为主序，其它语言中的 C 语言和 Python 语言( ``numpy`` )就是选用了这种方式。记住数组的顺序对数组的查找有着至关重要的影响。要记住的一个查找规则就是对于基于列为顺序的数组，第一个指针是变化最快的。这基本上就意味着如果在一段代码中，循环指针是第一个，那么查找速度会更快。

我们来看一下下面这个人为的例子。假设我们想要实现一个功能，接收一个 ``Vector`` 并且返回一个方形的 ``Matrix`` ，且行或列为输入矢量的复制。我们假设是行还是列为数据的复制并不重要（或许剩下的代码可以相应地更容易的适应）。我们可以想到有至少四种方法可以实现这一点（除了建议的回访正建的 ``repmat`` 功能）： 

```
function copy_cols{T}(x::Vector{T})
    n = size(x, 1)
    out = Array(eltype(x), n, n)
    for i=1:n
        out[:, i] = x
    end
    out
end

function copy_rows{T}(x::Vector{T})
    n = size(x, 1)
    out = Array(eltype(x), n, n)
    for i=1:n
        out[i, :] = x
    end
    out
end

function copy_col_row{T}(x::Vector{T})
    n = size(x, 1)
    out = Array(T, n, n)
    for col=1:n, row=1:n
        out[row, col] = x[row]
    end
    out
end

function copy_row_col{T}(x::Vector{T})
    n = size(x, 1)
    out = Array(T, n, n)
    for row=1:n, col=1:n
        out[row, col] = x[col]
    end
    out
end
```

现在我们使用同样的输入向量 ``1`` 产生的随机数 ``10000`` 给每个功能计时：

```
julia> x = randn(10000);

julia> fmt(f) = println(rpad(string(f)*": ", 14, ' '), @elapsed f(x))

julia> map(fmt, {copy_cols, copy_rows, copy_col_row, copy_row_col});
copy_cols:    0.331706323
copy_rows:    1.799009911
copy_col_row: 0.415630047
copy_row_col: 1.721531501
```

注意到 ``copy_cols`` 比 ``copy_rows`` 快很多。这是意料之中的，因为 ``copy_cols``  遵守 ``Matrix`` 界面的基于列的存储，并且一次就填满一列。除此之外， ``copy_col_row`` 比 ``copy_row_col`` 快很多，因为它符合我们的查找规则，即在一段代码中第一个出现的元素应该是与最内部的循环相联系的。  


## 输出预先分配


如果你的功能返回了一个 Array 或其它复杂类型，它可能不得不分配内存。不幸的是，时常分配和它的相反事件，垃圾区收集，是有实质性瓶颈的。  

有时候，你可以在访问每个功能时通过预先分配输出来避开分配内存的需要。作为一个很小的例子，比较一下  

```
function xinc(x)
    return [x, x+1, x+2]
end

function loopinc()
    y = 0
    for i = 1:10^7
        ret = xinc(i)
        y += ret[2]
    end
    y
end
```

和

```
function xinc!{T}(ret::AbstractVector{T}, x::T)
    ret[1] = x
    ret[2] = x+1
    ret[3] = x+2
    nothing
end

function loopinc_prealloc()
    ret = Array(Int, 3)
    y = 0
    for i = 1:10^7
        xinc!(ret, i)
        y += ret[2]
    end
    y
end
```

计时结果：

```
    julia> @time loopinc()
    elapsed time: 1.955026528 seconds (1279975584 bytes allocated)
    50000015000000

    julia> @time loopinc_prealloc()
    elapsed time: 0.078639163 seconds (144 bytes allocated)
    50000015000000
```

预先分配有其他好处，比如，允许访问者通过算法控制“输出”类型。在上面的例子中，我们可以按照自己希望的，通过一个 ``SubArray`` 而不是 ``Array``。

按着最极端的来想，预先分配可以让你的代码看起来丑点，所以需要一些表达方式和判断。  

## 避免输入/输出时的串插入  


把数据写入文件（或者其他输入/输出设备）时，中间字符串的形成是额外的开销。而不是：  

```
    println(file, "$a $b")
```

使用：：  

```
    println(file, a, " ", b)
```

第一种代码形成了一个字符串，然后把它写入了文件，而第二种代码直接把值写入了文件。同样也注意到在某些情况下，字符串的插入很难读出来。考虑一下：  

```
    println(file, "$(f(a))$(f(b))")
```

对比：：  

```
    println(file, f(a), f(b))
```


## 处理有关舍弃的警告

被舍弃的函数，会查表并显示一次警告，而这会影响性能。建议按照警告的提示进行对应的修改。

## 小技巧


注意些有些小事项，能使内部循环更紧致。

-  避免不必要的数组。例如，不要使用 ``sum([x,y,z])`` ，而应使用 ``x+y+z``
-  对于较小的整数幂，使用 ``*`` 更好。如 ``x*x*x`` 比 ``x^3`` 好
-  针对复数 ``z`` ，使用 ``abs2(z)`` 代替 ``abs(z)^2`` 。一般情况下，对于复数参数，尽量用 ``abs2`` 代替 ``abs``
-  对于整数除法，使用 ``div(x,y)`` 而不是 ``trunc(x/y)``, 使用 ``fld(x,y)`` 而不是 ``floor(x/y)``, 使用 ``cld(x,y)`` 而不是 ``ceil(x/y)``.


## 性能注释


有时你可以设定某些项目属性来获得更好的优化。  

-  在检查公式时，使用 ``@inbounds`` 来消除数组界限。一定要在这之前完成。如果下标越界了，你可能会遇到崩溃或不执行的问题。
-  在 ``for`` 循环之前写上  ``@simd`` ，这个可以帮你检验。**这个特征是试验性的** 而且在之后的 Julia 版本中可能会改变会消失。

这里有一个包含两种形式审定的例子：  

```
    function inner( x, y )
        s = zero(eltype(x))
        for i=1:length(x)
            @inbounds s += x[i]*y[i]
        end
        s
    end

    function innersimd( x, y )
        s = zero(eltype(x))
        @simd for i=1:length(x)
            @inbounds s += x[i]*y[i]
        end
        s
    end

    function timeit( n, reps )
        x = rand(Float32,n)
        y = rand(Float32,n)
        s = zero(Float64)
        time = @elapsed for j in 1:reps
            s+=inner(x,y)
        end
        println("GFlop        = ",2.0*n*reps/time*1E-9)
        time = @elapsed for j in 1:reps
            s+=innersimd(x,y)
        end
        println("GFlop (SIMD) = ",2.0*n*reps/time*1E-9)
    end

    timeit(1000,1000)
```

在配有2.4GHz的Intel Core i5 处理器的电脑上，产生如下结果：  

```
    GFlop        = 1.9467069505224963
    GFlop (SIMD) = 17.578554163920018
```

``@simd for`` 循环应该是一维范围的。 *缩减变数* 是用于累积变量的，比如例子中的  ``s`` 。通过使用 ``@simd``， 你可以维护循环的几种性能：  

-  有缩减变数的特殊考虑后，在任意的或重叠的顺序中执行迭代都是安全的。
-  减少变量的浮点操作可以被重复执行，但是可能会比没有 ``@simd`` 产生不同的结果。  
-  不会有一个迭代在等待另一个迭代，以实现前进。  

使用 ``@simd`` 仅仅是给了编译器矢量化的通行证。它是不是真的会这样做还取决于编译器。要真正从当前的实现中获益，你的循环应该有如下额外的性能：   

-  循环必须是内部循环。
-  循环主题必须是无循环程序。这就是为什么当前所有的数组访问都需要 ``@inbounds`` 的原因了。
-  访问必须有一个跨越模式，而且不能“聚集”（随机指针读取）或者 “分散”（随机指针写入）。
-  跨越应该是单元跨越。
-  在一些简单的例子中，例如一个2-3数组访问的循环中， LLVM 自动矢量化可能会自动生效，导致无需 ``@simd`` 的进一步加速。
