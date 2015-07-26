# 嵌入式 Julia

我们已经知道 调用 [C 和 Fortran 代码](use-c-fortran.md)Julia 可以用简单有效的方式调用 C 函数。但是有很多情况下正好相反:需要从C 调用 Julia 函数。这可以把 Julia 代码整合到更大型的 C/C++ 项目中去， 而不需要重新把所有都用C/C++写一遍。 Julia提供了给C的API来实现这一点。正如大多数语言都有方法调用 C 函数一样， Julia的 API 也可以用于搭建和其他语言之间的桥梁。

## 高级嵌入

我们从一个简单的C程序入手，它初始化Julia并且调用一些Julia的代码::

```
  #include <julia.h>

  int main(int argc, char *argv[])
  {
      jl_init(NULL);
      JL_SET_STACK_BASE;

      jl_eval_string("print(sqrt(2.0))");

      return 0;
  }
```

编译这个程序你需要把 Julia的头文件包含在路径内并且链接函数库 ``libjulia``。 比方说 Julia安装在 ``$JULIA_DIR``， 就可以用gcc编译::

```
    gcc -o test -I$JULIA_DIR/include/julia -L$JULIA_DIR/usr/lib -ljulia test.c
```

或者可以看看 Julia 源码里 ``example/`` 下的 ``embedding.c``。

调用Julia函数之前要先初始化Julia， 可以用 ``jl_init`` 完成，这个函数的参数是Julia安装路径，类型是 ``const char*`` 。如果没有任何参数，Julia会自动寻找Julia的安装路径。 

第二个语句初始化了 Julia 的任务调度系统。这条语句必须在一个不返回的函数中出现，只要 Julia 被调用（``main`` 运行得很好）。严格地讲，这条语句是可以选择的，但是转换任务的操作将引起问题，如果它被省略的话。  

测试程序中的第三条语句使用 ``jl_eval_string`` 的调用评估了 Julia 语句。  

## 类型转换

真正的应用程序不仅仅需要执行表达式，而且返回主程序的值。``jl_eval_string`` 返回 ``jl_value_t``，它是一个指向堆上分配的 Julia 对象的指针。用这种方式存储简单的数据类型比如 ``Float64`` 叫做 ``boxing``，提取存储的原始数据叫做 ``unboxing``。我们提升过的用 Julia 计算 2 的平方根和用 C 语言读取结果的样本程序如下所示：

```
    jl_value_t *ret = jl_eval_string("sqrt(2.0)");

    if (jl_is_float64(ret)) {
        double ret_unboxed = jl_unbox_float64(ret);
        printf("sqrt(2.0) in C: %e \n", ret_unboxed);
    }
```

为了检查 ``ret`` 是否是一个指定的 Julia 类型，我们可以使用 ``jl_is_...`` 函数。通过将 ``typeof(sqrt(2.0))`` 输入进 Julia shell，我们可以看到返回类型为 ``Float64``（C 中的 double）。为了将装好的 Julia 的值转换成 C 语言中的 double,``jl_unbox_float64`` 功能在上面的代码片段中被使用。

相应的 ``jl_box_...`` 功能被用来用另一种方式转换：

```
    jl_value_t *a = jl_box_float64(3.0);
    jl_value_t *b = jl_box_float32(3.0f);
    jl_value_t *c = jl_box_int32(3);
```

正如我们下面将看到的，调用带有指定参数的 Julia 函数装箱(boxing)是需要的。

## 调用 Julia 的函数

当 ``jl_eval_string`` 允许 C 语言来获得 Julia 表达式的结果时，它不允许传递在 C 中计算的参数到 Julia 中。对于这个，你需要直接调用 Julia 函数，使用 ``jl_call``:

```
    jl_function_t *func = jl_get_function(jl_base_module, "sqrt");
    jl_value_t *argument = jl_box_float64(2.0);
    jl_value_t *ret = jl_call1(func, argument);
```

在第一步中，Julia 函数 ``sqrt`` 的处理通过调用 ``jl_get_function`` 检索。第一个传递给 ``jl_get_function`` 的参数是一个指向 ``Base`` 模块的指针，在那里 ``sqrt`` 被定义。然后，double 值使用 ``jl_box_float64`` 封装。最后，在最后一步中，函数使用 ``jl_call1`` 被调用。``jl_call0``,``jl_call2`` 和 ``jl_call3`` 函数也存在，来方便地处理不同参数的数量。为了传递更多的参数，使用 ``jl_call``:  

```
    jl_value_t *jl_call(jl_function_t *f, jl_value_t **args, int32_t nargs)
```

第二个参数 ``args`` 是一个 ``jl_value_t*`` 参数的数组而且 ``nargs`` 是参数的数字。

## 内存管理

正如我们已经看见的，Julia 对象作为指针在 C 中呈现。这引出了一个问题，谁应该释放这些对象。

通常情况下，Julia 对象通过一个 garbage collector(GC) 来释放，但是 GC 不会自动地知道我们在 C 中有一个对 Julia 值的引用。这意味着 GC 可以释放指针，使指针无效。

GC 仅能在 Julia 对象被分配时运行。像 ``jl_box_float64`` 的调用运行分配，而且分配也能在任何运行 Julia 代码的指针中发生。在 ``jl_...`` 调用间使用指针通常是安全的。但是为了确认值能使 ``jl_...`` 调用生存，我们不得不告诉 Julia 我们有一个对 Julia 值的引用。这可以使用 ``JL_GC_PUSH`` 宏指令完成。This can be done using the ``JL_GC_PUSH`` macros:

```
    jl_value_t *ret = jl_eval_string("sqrt(2.0)");
    JL_GC_PUSH1(&ret);
    // Do something with ret
    JL_GC_POP();
```

``JL_GC_POP`` 调用释放了之前 ``JL_GC_PUSH`` 建立的引用。注意到 ``JL_GC_PUSH`` 在栈上工作，所以它在栈帧被销毁之前必须准确地和 ``JL_GC_POP`` 成组。

几个 Julia 值能使用 ``JL_GC_PUSH2``，``JL_GC_PUSH3``，和 ``JL_GC_PUSH4`` 宏指令被立刻 push。为了 push 一个 Julia 值的数组我们可以使用 ``JL_GC_PUSHARGS`` 宏指令，它能向以下那样使用：cro, which can be used as follows:

```
    jl_value_t **args;
    JL_GC_PUSHARGS(args, 2); // args can now hold 2 `jl_value_t*` objects
    args[0] = some_value;
    args[1] = some_other_value;
    // Do something with args (e.g. call jl_... functions)
    JL_GC_POP();
```

### 控制垃圾回收

有一些函数来控制 GC。在普通的使用案例中，这些不应该是必需的。

|``void jl_gc_collect()``  | Force a GC run|
| ------------- |:-------------| 
|``void jl_gc_disable()``  | Disable the GC|
|``void jl_gc_enable()`` |   Enable the GC|

## 处理数组

Julia 和 C 能不用复制而分享数组数据。下一个例子将展示这是如此工作的。

Julia 数组通过数据类型 ``jl_array_t*`` 用 C 语言显示。基本上，``jl_array_t`` 是一个包含以下的结构：

- 有关数据类型的信息
- 指向数据块的指针
- 有关数组大小的信息

为了使事情简单，我们用一个 1D 数组开始。创建一个包含长度为 10 个元素的 Float64 数组通过以下完成：

```
    jl_value_t* array_type = jl_apply_array_type(jl_float64_type, 1);
    jl_array_t* x          = jl_alloc_array_1d(array_type, 10);
```

或者，如果你已经分配了数组你能生成一个对数据的简单包装：

```
double *existingArray = (double*)malloc(sizeof(double)*10);
jl_array_t *x = jl_ptr_to_array_1d(array_type, existingArray, 10, 0);
```
    
最后一个参数是一个表明 Julia 是否应该获取数据所有权的布尔值。如果这个参数非零，GC 将在数组不再引用时在数据指针上调用 ``free``。

为了获取 x 的数据，我们可以使用 ``jl_array_data``:

```
    double *xData = (double*)jl_array_data(x);
```
    
现在我们可以填写数组：

```
    for(size_t i=0; i<jl_array_len(x); i++)
        xData[i] = i;
```
      
现在让我们调用一个在 ``x`` 上运行操作的 Julia 函数：

```
    jl_function_t *func  = jl_get_function(jl_base_module, "reverse!");
    jl_call1(func, (jl_value_t*)x);
```

通过打印数组，我们可以核实 ``x`` 的元素现在被颠倒了。

### 访问返回的数组

如果一个 Julia 函数返回一个数组，``jl_eval_string`` 和 ``jl_call`` 的返回值能被转换成一个 ``jl_array_t*``:

```
    jl_function_t *func  = jl_get_function(jl_base_module, "reverse");
    jl_array_t *y = (jl_array_t*)jl_call1(func, (jl_value_t*)x);
```

现在 ``y`` 的内容能在使用 ``jl_array_data`` 前被获取。一如往常，当它在使用中时确保保持对数组的引用。

## 高维数组

Julia 的多维数组在内存中以列的顺序被存储。这儿是一些创建二维数组和获取属性的代码：

```
    // Create 2D array of float64 type
    jl_value_t *array_type = jl_apply_array_type(jl_float64_type, 2);
    jl_array_t *x  = jl_alloc_array_2d(array_type, 10, 5);

    // Get array pointer
    double *p = (double*)jl_array_data(x);
    // Get number of dimensions
    int ndims = jl_array_ndims(x);
    // Get the size of the i-th dim
    size_t size0 = jl_array_dim(x,0);
    size_t size1 = jl_array_dim(x,1);

    // Fill array with data
    for(size_t i=0; i<size1; i++)
        for(size_t j=0; j<size0; j++)
            p[j + size0*i] = i + j;
```

注意到当 Julia 数组使用 1-based 的索引，C 的 API 使用 0-based 的索引（例如在调用 ``jl_array_dim`` 时）以作为惯用的 C 代码读取。

## 异常

Julia 代码能抛出异常。比如，考虑以下：

```
      jl_eval_string("this_function_does_not_exist()");
```

这个调用将什么都不做。但是，检查一个异常是否抛出是可能的。

```
    if (jl_exception_occurred())
        printf("%s \n", jl_typeof_str(jl_exception_occurred()));
```

如果你用一个支持异常的语言（比如，Python，C#，C++）使用 Julia C API，用一个检查异常是否被抛出的函数包装每一个调用 libjulia 的调用是有道理的，而且它用主语言重新抛出异常。

### 抛出 Julia 异常

当写一个可调用的 Julia 函数时，验证参数和抛出异常来指出错误是必要的。一个典型的类型检查像这样：

```
    if (!jl_is_float64(val)) {
        jl_type_error(function_name, (jl_value_t*)jl_float64_type, val);
    }
```

通常的异常能使用函数来引起：

```
    void jl_error(const char *str);
    void jl_errorf(const char *fmt, ...);
```

``jl_error`` 使用一个 C 的字符串，``jl_errorf`` 像 ``printf`` 一样被调用：

```
    jl_errorf("argument x = %d is too large", x);
```

在这个例子中 ``x`` 被假设为一个整型。
