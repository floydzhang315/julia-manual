
# 开发扩展包

Julia 中设有包管理器，当你安装了扩展包时，你可以看到它的源代码和完整的开发历史。你也可以修改扩展包，并使用 git 提交它们，为修复和增加扩展包功能做贡献。相似地，这个系统设计用来当你想要创建一个新扩展包时，最简单的方法就是利用包管理器中提供的基础设施。  

## 初始化设置

由于扩展包存储于 git 仓库中，所以在做扩展包开发之前，你需要先设置如下全局 git 配置：  

```

    $ git config --global user.name "FULL NAME"
    $ git config --global user.email "EMAIL"

```

``FULL NAME`` 是你真实的全名(双引号之间允许有空格)并且 ``EMAIL`` 是你真实的邮箱地址。  
尽管创建和发布 Julia 扩展包时使用 [GitHub](https://github.com/) 并不是必要的，然而大多数 Julia 扩展包都存在 GitHub 上并且包管理器知道如何正确地格式化源 URL，并在其他方面上顺利的使用服务。我们建议你创建一个 [免费账号](https://github.com/join) 在 GitHub 上然后做：

```

    $ git config --global github.user "USERNAME"

```

在这里 ``USERNAME`` 是你 GitHub 上正确的用户名。只要你做了这一点，包管理器就知道你的 GitHub 用户名然后可以配置相关事项。你还需要  [上传](https://github.com/settings/ssh) 你的SSH公钥到 GitHub 上并设置一个 [SSH 代理](http://linux.die.net/man/1/ssh-agent>) 在你的开发机器上，这样你可以最简单的推送你的修改。在将来，我们会让这个系统具有扩展性，支持更多其它的常见 git 工具例如 [BitBucket](https://bitbucket.org) 并且允许开发者选择他们所喜欢的。

## 生成新扩展包


假如你想创建一个新的 Julia 扩展包，名为 ``FooBar``。首先，你需要 ``Pkg.generate(pkg,license)``，其中 ``pkg`` 是新扩展包的名字并且 ``license`` 是生成器知晓的许可的名字：  

```

    julia> Pkg.generate("FooBar","MIT")
    INFO: Initializing FooBar repo: /Users/stefan/.julia/v0.3/FooBar
    INFO: Origin: git://github.com/StefanKarpinski/FooBar.jl.git
    INFO: Generating LICENSE.md
    INFO: Generating README.md
    INFO: Generating src/FooBar.jl
    INFO: Generating test/runtests.jl
    INFO: Generating .travis.yml
    INFO: Committing FooBar generated files

```

这样创建了一个目录 ``~/.julia/v0.3/FooBar``，将它初始化为一个 git 仓库，生成所有包需要有的一系列文件，并把它们提交到仓库：  

```

    $ cd ~/.julia/v0.3/FooBar && git show --stat

    commit 84b8e266dae6de30ab9703150b3bf771ec7b6285
    Author: Stefan Karpinski <stefan@karpinski.org>
    Date:   Wed Oct 16 17:57:58 2013 -0400

        FooBar.jl generated files.

            license: MIT
            authors: Stefan Karpinski
            years:   2013
            user: StefanKarpinski

        Julia Version 0.3.0-prerelease+3217 [5fcfb13*]

     .travis.yml      | 16 +++++++++++++
     LICENSE.md       | 22 +++++++++++++++++++++++
     README.md        |  3 +++
     src/FooBar.jl    |  5 +++++
     test/runtests.jl |  5 +++++
     5 files changed, 51 insertions(+)

```

此时，包管理器知道 MIT "Expat" 证书用 ``"MIT"`` 表示，Simplified BSD 证书用 ``"BSD"`` 表示，2.0版本的 Apache 软件证书用 ``"ASL"`` 表示。如果你想要使用不同的证书，你可以让我们把它添加到扩展包生成器上，或者就选这三者之一然后在生成之后修改 ``~/.julia/v0.3/PACKAGE/LICENSE.md`` 文件。  
如果你创建了一个 GitHub 账户并且配置了 git,``Pkg.generate`` 将会设置一个合适的源 URL 给你。它还会自动生成 ``.travis.yml`` 文件来使用 [Travis](https://travis-ci.org) 自动测试服务。你可以在 Travis website 上测试你的扩展包仓库，但是只要你做了这个它就已经开始测试了。当然，所有的默认测试是查证 ``using FooBar`` 能否在 Julia 上工作。  

## 使你的扩展包具有可用性

只要你提交了一些内容，那么你会为测试 ``FooBar`` 是否可以工作而感到高兴，你可能想要一些其他人来测试一下。首先，你需要创建一个远程仓库并把你的代码推送进去；我们不会自动的为你做这件事，但是未来将会，这配置起来并不难[3]。只要你完成了这个，只需将发布的仓库的 URL 发给他们就可以请让他们来试一下你的代码 - 像这样：  

```

    git://github.com/StefanKarpinski/FooBar.jl.git

```

对于你的扩展包而言，它将具有你的 GitHub 用户名和你的扩展包名，但是你明白是什么意思。收到你发的 URL 的人们可以使用 ``Pkg.clone`` 来安装扩展包并测试它:  

```

    julia> Pkg.clone("git://github.com/StefanKarpinski/FooBar.jl.git")
    INFO: Cloning FooBar from git@github.com:StefanKarpinski/FooBar.jl.git

```

>[3]: 极度推荐安装并使用 GitHub 的 ["hub" 工具](https://github.com/github/hub)。它允许你在扩展包仓库中像运行 ``hub create`` 那样做事，然后它会通过 GitHub 的 API 自动创建。

## 发布你的扩展包

一旦你决定 ``FooBar`` 已经准备好注册成为一个官方正式扩展包，你可以把它添加到你的本地 ``METADATA`` 的拷贝，并命名为 ``Pkg.register``:  

```

    julia> Pkg.register("FooBar")
    INFO: Registering FooBar at git://github.com/StefanKarpinski/FooBar.jl.git
    INFO: Committing METADATA for FooBar

```

这会在 ``~/.julia/v0.3/METADATA`` 仓库中创建一次提交:  

```

    $ cd ~/.julia/v0.3/METADATA && git show

    commit 9f71f4becb05cadacb983c54a72eed744e5c019d
    Author: Stefan Karpinski <stefan@karpinski.org>
    Date:   Wed Oct 16 18:46:02 2013 -0400

        Register FooBar

    diff --git a/FooBar/url b/FooBar/url
    new file mode 100644
    index 0000000..30e525e
    --- /dev/null
    +++ b/FooBar/url
    @@ -0,0 +1 @@
    +git://github.com/StefanKarpinski/FooBar.jl.git

```

然而，这次提交只是本地可见的。为了能将它公诸于世，你需要将你的本地 ``METADATA`` 上传到正式库中合并。``Pkg.publish()`` 命令将在 GitHub 上创建 ``METADATA`` 仓库的分支，并将你的修改提交到分支上，并打开一个拉取请求：  

```

  julia> Pkg.publish()
  INFO: Validating METADATA
  INFO: No new package versions to publish
  INFO: Submitting METADATA changes
  INFO: Forking JuliaLang/METADATA.jl to StefanKarpinski
  INFO: Pushing changes as branch pull-request/ef45f54b
  INFO: To create a pull-request open:  

    https://github.com/StefanKarpinski/METADATA.jl/compare/pull-request/ef45f54b

```

由于各种各样的原因 ``Pkg.publish()`` 有时并不会成功。在那些情况下，你可能在 GitHub 上做了一个拉取请求，这并[不难](https://help.github.com/articles/creating-a-pull-request)。  
只要 ``FooBar`` 扩展包的 URL 在正式 ``METADATA`` 仓库中注册，人们就知道从哪里克隆这个扩展包，但是这并没有一些注册过的版本可供下载。这意味着 ``Pkg.add("FooBar")`` 在只安装正式版本时并没有工作。``Pkg.clone("FooBar")`` 没有一个指定的 URL 指向它。此外，当他们运行 ``Pkg.update()``，他们将会得到你上传到仓库中最新版本的 ``FooBar``。当你还在修改它，在它没有成为正式版之前这是一个比较好的方式测试你的扩展包。  

## 扩展包版本号标签

当你准备好为你的扩展包制作一个正式版本时，你可以使用 ``Pkg.tag`` 命令为它添加版本号并注册：

```

    julia> Pkg.tag("FooBar")
    INFO: Tagging FooBar v0.0.1
    INFO: Committing METADATA for FooBar

```

这个 ``v0.0.1`` 标签在 ``FooBar`` 仓库中：  

```

    $ cd ~/.julia/v0.3/FooBar && git tag
    v0.0.1

```

它也可以为 ``FooBar`` 在你的本地 ``METADATA`` 仓库中创建一个新的版本入口：  

```

    $ cd ~/.julia/v0.3/FooBar && git show
    commit de77ee4dc0689b12c5e8b574aef7f70e8b311b0e
    Author: Stefan Karpinski <stefan@karpinski.org>
    Date:   Wed Oct 16 23:06:18 2013 -0400

        Tag FooBar v0.0.1

    diff --git a/FooBar/versions/0.0.1/sha1 b/FooBar/versions/0.0.1/sha1
    new file mode 100644
    index 0000000..c1cb1c1
    --- /dev/null
    +++ b/FooBar/versions/0.0.1/sha1
    @@ -0,0 +1 @@
    +84b8e266dae6de30ab9703150b3bf771ec7b6285

```

如果在你的扩展包仓库中有一个 ``REQUIRE`` 文件，它将会在你标记版本时拷贝到 ``METADATA`` 中适当的位置。扩展包开发者们需要确定他们的扩展包中的 ``REQUIRE`` 文件确实反应他们扩展包的需求，如果你使用 ``Pkg.tag`` 命令，这将自动进入你的正式版。看 [Requirements Specification](#man-package-requirements) 来了解完整格式的 ``REQUIRE``。  
``Pkg.tag`` 命令有第二个可选参数是一个显示的版本号对象如 ``v"0.0.1"`` 或者一个标志 ``:patch``， ``:minor`` 或者 ``:major``。这会智能的添加你的扩展包的补丁、副本或者主版本号。  
正如使用 ``Pkg.register``，这些对于 ``METADATA`` 的修改不会对其它任何人可见直到这些修改被上传。再一次使用 ``Pkg.publish()`` 命令行，它第一次使用的时候要确定每个独立的扩展包仓库已经被标记，如果它们没有被标记要提交它们，然后打开一个到 ``METADATA`` 的拉取请求：  

```

  julia> Pkg.publish()
  INFO: Validating METADATA
  INFO: Pushing FooBar permanent tags: v0.0.1
  INFO: Submitting METADATA changes
  INFO: Forking JuliaLang/METADATA.jl to StefanKarpinski
  INFO: Pushing changes as branch pull-request/3ef4f5c4
  INFO: To create a pull-request open:  

    https://github.com/StefanKarpinski/METADATA.jl/compare/pull-request/3ef4f5c4

```

## 修改扩展包需求

如果你需要修改一个已发布扩展包版本的注册需求，你只需要修改这个版本的 metadata 即可，这样可以保持相同的提交散列值 – 散列值与一个版本永久相关:  

```

  $ cd ~/.julia/v0.3/METADATA/FooBar/versions/0.0.1 && cat requires
  julia 0.3-

  $ vi requires

```

为了保持提交的散列值保持一致，需要检验仓库中的 ``REQUIRE`` 文件的内容是否与在 ``METADATA`` 中的在修改之后**不**匹配；这是不可避免的。
尽管当你在 ``METADATA`` 中为之前版本的扩展包修改了需求，你仍需要在当前版本的扩展包中修改 ``REQUIRE`` 文件。  

## 依赖关系

在扩展包中的 ``~/.julia/v0.3/REQUIRE`` 文件, ``REQUIRE`` 文件，和 ``METADATA`` 包 ``requires`` 文件使用一个简单的基于行的格式来显示需要安装的扩展包版本的范围。包 ``REQUIRE`` 和 ``METADATA requires`` 文件也需要包括扩展包兼容的 ``julia`` 的版本范围。

这里是这些包如何被解析和解释的。  

* 所有在 ``#`` 号后的内容被从行中剥离成为注释。
* 如果出了空白什么都没有，那么这一行被忽略。
* 如果剩下的都是非空字符，那么这一行是一个依赖关系，并且需要用空格分开每个单词。

最简单的有可能的依赖关系是这一行只有扩展包的名字：  

```

    Distributions

```

这个依赖将被任何版本的 ``Distributions`` 扩展包满足。这个扩展包的名字可以紧随零活更多升序版本号之后，指明可以接受的那个扩展包的版本间隔。一个版本号开始一个间距，下一个是这个间距的结束，然后下一个又是一个新的开始，然后继续；如果出现了一个奇怪的版本号，那么任意更高的版本都将兼容；如果给出了一个相同的版本号，那么后一个是可以兼容的最高版本。举个例子，这一行：  

```

    Distributions 0.1

```

``0.1.0`` 及其之后的版本的 ``Distributions`` 都将被兼容。一个版本号以 `-` 作为后缀也允许任何相同前缀的发布版本兼容。例如：  

```

    Distributions 0.1-

```

兼容相同前缀的版本例如 ``0.1-dev`` 或 ``0.1-rc1``，或 ``0.1.0`` 及其之后的任何版本。  
这个依赖条目：  

```

    Distributions 0.1 0.2.5

```

兼容从 ``0.1.0`` 起的任何版本，但是不包括 ``0.2.5``。  
如果你想要表明任何 ``0.1.x`` 版本被兼容，你可以这样写：

```

    Distributions 0.1 0.2-

```

如果你想要兼容在 ``0.2.7`` 之后的版本，你可以这样写：  

```

    Distributions 0.1 0.2- 0.2.7

```

如果一个依赖行以引导字符 ``@`` 开始，这是一个系统依赖关系。如果你的系统匹配这些系统环境，依赖关系就会被包含，否则将被忽略。例如：  

```

    @osx Homebrew

```

将仅在操作系统是 OS X 时需要 ``Homebrew`` 扩展包。当前支持的系统环境包括：  

```

    @windows
    @unix
    @osx
    @linux

```

``@unix`` 环境适应于所有的 UNIX 操作系统，包括 OS X, Linux 和 FreeBSD。在引导字符 ``@`` 后添加 ``!`` 表示否定的操作系统。例子：  

```

    @!windows
    @unix @!osx

```

第一个环境应用于任何系统除了 Windows ，第二个环境应用于任何 UNIX 系统除了 OS X。  
运行时检查 Julia 的当前版本可以应用在内置 ``VERSION`` 变量，这是一种  ``VersionNumber``。这些代码偶尔是必要的用来跟踪在发布的 Julia 版本之间的新功能或弃用的功能。运行时检查的例子：  

```

    VERSION < v"0.3-" #exclude all pre-release versions of 0.3

    v"0.2-" <= VERSION < v"0.3-" #get all 0.2 versions, including pre-releases, up to the above

    v"0.2" <= VERSION < v"0.3-" #To get only stable 0.2 versions (Note v"0.2" == v"0.2.0")

    VERSION >= v"0.2.1" #get at least version 0.2.1

```

到 `version number literals <man-version-number-literals>` 查看跟过更完整的描述细节。

