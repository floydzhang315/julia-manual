# 扩展包

Julia 内置了一个包管理系统，可以用这个系统来完成包的管理，当然，你也可以用你的操作系统自带的，或者从源码编译。
你可以在 http://pkg.julialang.org  找到所有已注册（一种发布包的机制）的包的列表。
所有的包管理命令都包含在 ``Pkg`` 这个module里面，Julia的 Base install 引入了 ``Pkg`` 。

## 扩展包状态


可以通过 ``Pkg.status()`` 这个方程，打印出一个你所有安装的包的总结。

刚开始的时候，你没有安装任何包::

```
    julia> Pkg.status()
    INFO: Initializing package repository /Users/stefan/.julia/v0.3
    INFO: Cloning METADATA from git://github.com/JuliaLang/METADATA.jl
    No packages installed.
```

当你第一次运行 ``Pkg`` 的一个命令时， 你的包目录（所有的包被安装在一个统一的目录下）会自动被初始化，因为 ``Pkg`` 希望有这样一个目录，这个目录的信息被包含于 ``Pkg.status()`` 中。

这里是一个简单的，已经有少量被安装的包的例子:

```
    julia> Pkg.status()
    Required packages:
     - Distributions                 0.2.8
     - UTF16                         0.2.0
    Additional packages:
     - NumericExtensions             0.2.17
     - Stats                         0.2.6
```

这些包，都是已注册了的版本，并且通过 ``Pkg`` 管理。
安装了的包可以是一个更复杂的"状态"，通过"注释"来表明正确的版本；当我们遇到这些“状态”和“注释”时我们会解释的。

为了编程需要，``Pkg.installed()`` 返回一个字典，这个字典对应了安装了的包的名字和其现在使用的版本:

```
    julia> Pkg.installed()

    ["Distributions"=>v"0.2.8","Stats"=>v"0.2.6","UTF16"=>v"0.2.0","NumericExtensions"=>v"0.2.17"]
```

## 添加和删除扩展包

Julia 的包管理有一点不同这是因为它是生命而不是必要。这意味着你告诉它你想要什么，它就会知道安装什么版本（或移除）来有选择地满足那些需求 - 最低程度下地。所以不是安装一个包，你只是添加它到需求列表然后“解决”什么需要被安装。特别的，这意味着如果一些包因为它被你想要东西的前一个版本所需要而已经被安装，而且一个更新的版本不再有那个需求了，更新将真正移除那个包。

你的包需求在文件 ``~/.julia/v0.3/REQUIRE`` 中。你可以手动编辑这个文件，然后调用 ``Pkg.resolve()`` 方法来安装，升级或者移除包来有选择地满足需求，或者你可以做 ``Pkg.edit()``，它将在你的编辑器中打开 ``REQUIRE``（通过 ``EDITOR`` 或者 ``VISUAL`` 环境变量配置），然后之后自动调用 ``Pkg.resolve()``，如果有必要的话。如果你仅仅想要添加或者移除一个单一包的需求，你也可以使用非交互的 ``Pkg.add`` 和 ``Pkg.rm`` 命令，它添加或移除一个单一的需求来 ``REQUIRE``，然后调用 ``Pkg.resolve()``。

你可以用 ``Pkg.add`` 函数添加一个包到需求列表，这个包和所有它所依赖的包都将被安装：

```
    julia> Pkg.status()
    No packages installed.

    julia> Pkg.add("Distributions")
    INFO: Cloning cache of Distributions from git://github.com/JuliaStats/Distributions.jl.git
    INFO: Cloning cache of NumericExtensions from git://github.com/lindahua/NumericExtensions.jl.git
    INFO: Cloning cache of Stats from git://github.com/JuliaStats/Stats.jl.git
    INFO: Installing Distributions v0.2.7
    INFO: Installing NumericExtensions v0.2.17
    INFO: Installing Stats v0.2.6
    INFO: REQUIRE updated.

    julia> Pkg.status()
    Required packages:
     - Distributions                 0.2.7
    Additional packages:
     - NumericExtensions             0.2.17
     - Stats                         0.2.6
```

这所做的事情首先是添加 ``Distributions`` 到你的 ``~/.julia/v0.3/REQUIRE`` 文件：

```
    $ cat ~/.julia/v0.3/REQUIRE
    Distributions
```

然后它使用这些新的需求运行 ``Pkg.resolve()``，它导向了 ``Distributions`` 包应该被安装因为它是必需的而且没有被安装的结论。正如之前所声明的，你可以通过手动编辑你的 ``~/.julia/v0.3/REQUIRE`` 文件完成相同的事情然后自己运行 ``Pkg.resolve()``。

```
    $ echo UTF16 >> ~/.julia/v0.3/REQUIRE

    julia> Pkg.resolve()
    INFO: Cloning cache of UTF16 from git://github.com/nolta/UTF16.jl.git
    INFO: Installing UTF16 v0.2.0

    julia> Pkg.status()
    Required packages:
     - Distributions                 0.2.7
     - UTF16                         0.2.0
    Additional packages:
     - NumericExtensions             0.2.17
     - Stats                         0.2.6
 ```

这和调用 ``Pkg.add("UTF16")``功能相同，除了 ``Pkg.add`` 直到在安装完成之后才改变 ``REQUIRE``，所以如果有问题的话，``REQUIRE`` 将被剩下，正如在调用 ``Pkg.add`` 之前。``REQUIRE`` 文件的格式在 [Requirements Specification](http://julia-cn.readthedocs.org/zh_CN/latest/manual/packages/#man-package-requirements)中被描述；它允许在其他事物中获得特定包版本的范围。  

当你决定你不想再拥有一个包，你可以使用 ``Pkg.rm`` 来从 ``REQUIRE`` 文件移除它的需求：  

```
    julia> Pkg.rm("Distributions")
    INFO: Removing Distributions v0.2.7
    INFO: Removing Stats v0.2.6
    INFO: Removing NumericExtensions v0.2.17
    INFO: REQUIRE updated.

    julia> Pkg.status()
    Required packages:
     - UTF16                         0.2.0

    julia> Pkg.rm("UTF16")
    INFO: Removing UTF16 v0.2.0
    INFO: REQUIRE updated.

    julia> Pkg.status()
    No packages installed.
```

再一次，这和编辑 ``REQUIRE`` 文件来移除有着包名的那一行然后运行 ``Pkg.resolve()``来更改安装包的集合来匹配相类似。尽管 ``Pkg.add`` 和 ``Pkg.rm`` 对于添加和移除单个包的需求来说是方便的，当你想要添加或移除多个包时，你可以调用 ``Pkg.edit()``来手动地改变 ``REQUIRE`` 的内容然后根据情况更新你的包。``Pkg.edit()``不回滚 ``REQUIRE`` 的内容如果 ``Pkg.resolve()``失效 - 不如说，你不得不再一次运行 ``Pkg.edit()``来修改文档内容。

因为包管理内部使用 git 来管理包 git 仓库，当运行 ``Pkg.add`` 时，用户可能会碰上协议的问题（比如在一个防火墙后）。接下来的命令可在命令行中被运行来告诉 git 当克隆仓库时使用 'https' 而不是 'git' 协议。

```
    git config --global url."https://".insteadOf git://
```

## 安装未注册的扩展包

Julia 包仅仅是 git 仓库，在任何 git 支持的[协议](https://www.kernel.org/pub/software/scm/git/docs/git-clone.html#URLS)上都是可克隆的，而且包含遵循特定布局惯例的 Julia 代码。官方的 Julia 包在 [METADATA.jl](https://github.com/JuliaLang/METADATA.jl) 仓库中注册，在可以著名的地方可获得。在之前的段落中，``Pkg.add`` 和 ``Pkg.rm`` 命令和注册的包交互，但是包管理也能安装并使用未注册的包。为了安装未注册的包，使用 ``Pkg.clone(url)``，在那里 ``url`` 是一个包能被克隆的 git URL： 

```
    julia> Pkg.clone("git://example.com/path/to/Package.jl.git")
    INFO: Cloning Package from git://example.com/path/to/Package.jl.git
    Cloning into 'Package'...
    remote: Counting objects: 22, done.
    remote: Compressing objects: 100% (10/10), done.
    remote: Total 22 (delta 8), reused 22 (delta 8)
    Receiving objects: 100% (22/22), 2.64 KiB, done.
    Resolving deltas: 100% (8/8), done.
```

按照惯例，Julia 仓库用一个 ``.jl`` 的结尾命名（附加的 ``.git`` 指示了一个“裸” git 仓库），这防止它们和其他语言的仓库碰撞，也使得 Julia 包在搜索引擎中方便找到。当包在你的 ``.julia/v0.3`` 目录下安装时，然而，扩展是多余的，所以我们将它留下。

如果未注册的包在它们的资源树的顶部包含 ``REQUIRE`` 文件，那这个文件将被用来决定未注册的包依赖于哪些注册的包，而且它们将自动被安装。未注册的包和注册的包一样，具有相同版本的解决逻辑，所以安装过的包版本将在必要时调整来满足注册过的和未注册过的包的需求。

[1] 官方的包集在 [https://github.com/JuliaLang/METADATA.jl](https://github.com/JuliaLang/METADATA.jl)，但是个人和组织能简单地使用一个不同的元数据仓库。这允许包可以自动安装的控制。我们可以仅允许审计通过的和批准的包版本，并使得私人的包和 fork 可被获得。

## 更新扩展包

当包开发者发布你正在使用的新的注册的包版本时，你当然，想要新的版本。为了获得最新和最棒的包版本，只要 ``Pkg.update()``:

```
    julia> Pkg.update()
    INFO: Updating METADATA...
    INFO: Computing changes...
    INFO: Upgrading Distributions: v0.2.8 => v0.2.10
    INFO: Upgrading Stats: v0.2.7 => v0.2.8
```

更新包的第一步是将新的改变放入 ``~/.julia/v0.3/METADATA`` 并看看是否有新的注册包版本已经被发布了。在这之后，``Pkg.update()``通过从包的上游库 pull 一些更改会更新在一个分支上被检查且不 dirty（比如，在 git 下没有对文件更改）的更新包。上游的改变仅仅在如果没有合并或重定基地址是有必要的情况下应用 - 比如，如果分支是 ["fast-forwarded"](http://git-scm.com/book/en/Git-Branching-Basic-Branching-and-Merging)。如果分支不是 fast-forwarded，就假设你正在使用它而且将自己更改仓库。 

最后，更新的过程重新计算了一个最佳的包版本的集合来安装以满足你顶级的需求和 “fix” 包的需求。包被认为是 fixed 如果它是下面几条之一：

1.**未注册：**包不在 ``METADATA`` 中 - 你用 ``Pkg.clone`` 安装过它。
2.**被检出:**包仓库在一个开发分支上。
3.**Dirty:**在仓库中对文件进行过了修改。

如果这些中的任何一项出现，包管理者不能自由地更改安装好的包版本，所以它的需求必须被满足，无论它所选择的其他包版本是怎样的。在 ``~/.julia/v0.3/REQUIRE`` 中的顶层需求的组合和修改过的包的需求被用来决定应该安装什么。

## Checkout, Pin and Free

你可能想要使用包的 ``master`` 版本而不是注册版本中的一个。在 master 上可能有修改或功能,它们是你所需要的且没有在任何注册版本上发布，或者你可能是一个包的开发者且想要改变 ``master`` 或一些其他的开发分支。在这些例子中，你能通过 ``Pkg.checkout(pkg)``来检查 ``pkg`` 或 ``Pkg.checkout(pkg,branch)``的 ``master`` 分支以检查一些其他的分支：

```
    julia> Pkg.add("Distributions")
    INFO: Installing Distributions v0.2.9
    INFO: Installing NumericExtensions v0.2.17
    INFO: Installing Stats v0.2.7
    INFO: REQUIRE updated.

    julia> Pkg.status()
    Required packages:
     - Distributions                 0.2.9
    Additional packages:
     - NumericExtensions             0.2.17
     - Stats                         0.2.7

    julia> Pkg.checkout("Distributions")
    INFO: Checking out Distributions master...
    INFO: No packages to install, update or remove.

    julia> Pkg.status()
    Required packages:
     - Distributions                 0.2.9+             master
    Additional packages:
     - NumericExtensions             0.2.17
     - Stats                         0.2.7
 ```

一旦在用 ``Pkg.add`` 安装 ``Distributions`` 之后，在写完的同时它就位于最新的注册版本上 - ``0.2.9``。然后在运行 ``Pkg.checkout("Distributions")``之后，你可以从 ``Pkg.status()``的输出中看到 ``Distributions`` 比起 ``0.2.9`` 在一个未注册的版本上更佳。由 “pseudo-version” 数字 ``0.2.9+`` 指示。

当你检查一个未注册的包版本时，包仓库中 ``REQUIRE`` 文件的副本地位高于任何其他在 ``METADATA`` 中注册的需求，所以开发者保持这个文件的正确性和及时性是很重要的，这反映了目前包版本的真正需求。如果在包仓库中的 ``REQUIRE`` 文件是不正确的或者遗失了，当包被检出时依赖性可能会被移除。这个文件也被用来填充新发布的包版本，如果你使用了 ``Pkg`` 为此提供的 API（在下面描述）。

当你决定你不再想要让一个包在分支上被检出，你能使用 ``Pkg`` “释放”它回到包管理者的控制之下。

```
    julia> Pkg.free("Distributions")
    INFO: Freeing Distributions...
    INFO: No packages to install, update or remove.

    julia> Pkg.status()
    Required packages:
     - Distributions                 0.2.9
    Additional packages:
     - NumericExtensions             0.2.17
     - Stats                         0.2.7
```

在这之后，因为包是在一个注册版本之上而且不在一个分支上，它的版本将被更新作为包的注册版本被发布。

如果你想要在一个指定的版本上 pin 一个包以使调用 ``Pkg.update()``不会改变包所在的版本，你可以使用 ``Pkg.pin`` 功能：

```
    julia> Pkg.pin("Stats")
    INFO: Creating Stats branch pinned.47c198b1.tmp

    julia> Pkg.status()
    Required packages:
     - Distributions                 0.2.9
    Additional packages:
     - NumericExtensions             0.2.17
     - Stats                         0.2.7              pinned.47c198b1.tmp
```

在这之后，``Stats`` 包将以版本 ``0.2.7`` 保持 pin 的状态 - 或者更具体地说，在提交 ``47c198b1``时，但是自从版本被永久地和一个给定的 git hash 连接后，这就一样了。``Pkg.pin`` 通过为你想要 pin 包的提交创建一个 throw-away 分支而运行。默认下，它在当前的提交下 pin 了一个包，但是你能通过传递第二个参数选择一个不同的版本：

```
    julia> Pkg.pin("Stats",v"0.2.5")
    INFO: Creating Stats branch pinned.1fd0983b.tmp
    INFO: No packages to install, update or remove.

    julia> Pkg.status()
    Required packages:
     - Distributions                 0.2.9
    Additional packages:
     - NumericExtensions             0.2.17
     - Stats                         0.2.5              pinned.1fd0983b.tmp
```

现在 ``Stats`` 包在提交 ``1fd0983b`` 时被 pin 了，它和 ``0.2.5`` 版本相一致。当你决定 “unpin” 一个包且让包管理者再一次更新它时，你可以使用 ``Pkg.free`` 就像你想要离开任何分支一样：

```
    julia> Pkg.free("Stats")
    INFO: Freeing Stats...
    INFO: No packages to install, update or remove.

    julia> Pkg.status()
    Required packages:
     - Distributions                 0.2.9
    Additional packages:
     - NumericExtensions             0.2.17
     - Stats                         0.2.7
```

Julia 的包管理者被设计以让当你有一个包需要安装时，你就可以查看它的源代码和完整的开发历史。你也可以对包做出更改，使用 git 提交它们，并能简单地作出修改和增强。相类似的，系统被设计以让如果你想要创建一个新的包，这么做最简单的方法就是在由包管理者提供的基础设施内部。

[2]:不在分支上的包也将被标记为 dirty，如果你在仓库中作出改变，但是那是一件比较少见的事。
