

#Interacting With Julia


Julia comes with a full-featured interactive command-line REPL (read-eval-print loop) built into the ``julia`` executable.  In addition to allowing quick and easy evaluation of Julia statements, it has a searchable history, tab-completion, many helpful keybindings, and dedicated help and shell modes.  The REPL can be started by simply calling julia with no arguments or double-clicking on the executable::

    $ julia
                   _
       _       _ _(_)_     |  A fresh approach to technical computing
      (_)     | (_) (_)    |  Documentation: http://docs.julialang.org
       _ _   _| |_  __ _   |  Type "help()" to list help topics
      | | | | | | |/ _` |  |
      | | |_| | | | (_| |  |  Version 0.3.0-prerelease+2834 (2014-04-30 03:13 UTC)
     _/ |\__'_|_|_|\__'_|  |  Commit 64f437b (0 days old master)
    |__/                   |  x86_64-apple-darwin13.1.0

    julia>

To exit the interactive session, type ``^D`` — the control key together with the ``d`` key on a blank line — or type ``quit()`` followed by the return or enter key. The REPL greets you with a banner and a ``julia>`` prompt.

The different prompt modes
--------------------------

###The Julian mode


The REPL has four main modes of operation.  The first and most common is the Julian prompt.  It is the default mode of operation; each new line initially starts with ``julia>``.  It is here that you can enter Julia expressions.  Hitting return or enter after a complete expression has been entered will evaluate the entry and show the result of the last expression.


    julia> string(1 + 2)
    "3"

There are a number useful features unique to interactive work. In addition to showing the result, the REPL also binds the result to the variable ``ans``.  A trailing semicolon on the line can be used as a flag to suppress showing the result.



    julia> string(3 * 4);

    julia> ans
    "12"

###Help mode


When the cursor is at the beginning of the line, the prompt can be changed to a help mode by typing ``?``.  Julia will attempt to print help or documentation for anything entered in help mode::

    julia> ? # upon typing ?, the prompt changes (in place) to: help>

    help> string
    Base.string(xs...)

       Create a string from any values using the "print" function.

In addition to function names, complete function calls may be entered to see which method is called for the given argument(s).  Macros, types and variables can also be queried::

    help> string(1)
    string(x::Union(Int16,Int128,Int8,Int32,Int64)) at string.jl:1553

    help> @printf
    Base.@printf([io::IOStream], "%Fmt", args...)

       Print arg(s) using C "printf()" style format specification
       string. Optionally, an IOStream may be passed as the first argument
       to redirect output.

    help> String
    DataType   : String
      supertype: Any
      subtypes : {DirectIndexString,GenericString,RepString,RevString{T<:String},RopeString,SubString{T<:String},UTF16String,UTF8String}

Help mode can be exited by pressing backspace at the beginning of the line.

###Shell mode


Just as help mode is useful for quick access to documentation, another common task is to use the system shell to execute system commands.  Just as ``?`` entered help mode when at the beginning of the line, a semicolon (``;``) will enter the shell mode.  And it can be exited by pressing backspace at the beginning of the line.


    julia> ; # upon typing ;, the prompt changes (in place) to: shell>

    shell> echo hello
    hello

###Search modes


In all of the above modes, the executed lines get saved to a history file, which can be searched.  To initiate an incremental search through the previous history, type ``^R`` — the control key together with the ``r`` key.  The prompt will change to ``(reverse-i-search)`':``, and as you type the search query will appear in the quotes.  The most recent result that matches the query will dynamically update to the right of the colon as more is typed.  To find an older result using the same query, simply type ``^R`` again.

Just as ``^R`` is a reverse search, ``^S`` is a forward search, with the prompt ``(i-search)`':``.  The two may be used in conjunction with each other to move through the previous or next matching results, respectively.


Key bindings
------------

The Julia REPL makes great use of key bindings.  Several control-key bindings were already introduced above (``^D`` to exit, ``^R`` and ``^S`` for searching), but there are many more.  In addition to the control-key, there are also meta-key bindings.  These vary more by platform, but most terminals  default to using alt- or option- held down with a key to send the meta-key (or can be configured to do so).

|Program control| |
|:----|:---|
|^D	|Exit (when buffer is empty)|
|^C	|Interrupt or cancel|
|Return/Enter, ^J|	New line, executing if it is complete|
|meta-Return/Enter|	Insert new line without executing it|
|? or ;	|Enter help or shell mode (when at start of a line)|
|^R, ^S|	Incremental history search, described above|
|**Cursor movement**||
|Right arrow, ^F	|Move right one character|
|Left arrow, ^B|	Move left one character|
|Home, ^A|	Move to beginning of line|
|End, ^E|	Move to end of line|
|^P	|Change to the previous or next history entry|
|^N	|Change to the next history entry|
|Up arrow|	Move up one line (or to the previous history entry)|
|Down arrow|	Move down one line (or to the next history entry)|
|Page-up	|Change to the previous history entry that matches the text before the cursor|
|Page-down|	Change to the next history entry that matches the text before the cursor|
|meta-F|	Move right one word|
|meta-B	|Move left one word|
|**Editing**|
|Backspace, ^H|	Delete the previous character|
|Delete, ^D	|Forward delete one character (when buffer has text)|
|meta-Backspace|	Delete the previous word|
|meta-D|	Forward delete the next word|
|^W	|Delete previous text up to the nearest whitespace|
|^K	|“Kill” to end of line, placing the text in a buffer|
|^Y	|“Yank” insert the text from the kill buffer|
|^T	|Transpose the characters about the cursor|
|Delete, ^D|	Forward delete one character (when buffer has text)|

###自定义快捷键


Julia REPL 的快捷键可以通过向 ``REPL.setup_interface()`` 传入字典类型的数据来实现自定义. 字典的关键字可以是字符, 也可以是字符串. 字符 ``*`` 代表默认默认操作. ``^x`` 代表快捷键 Control 键加 ``x`` 键. Meta 键加 ``x`` 键可以写作 ``"\\Mx"``.  字典的数据必须是 ``nothing`` (代表忽略该操作), 或者参数为 ``(PromptState, AbstractREPL, Char`` 的函数. 例如, 为了实现绑定上下键到搜索历史记录, 可以把下面的代码加入到 ``.juliarc.jl`` :

 

```
 import Base: LineEdit, REPL

  const mykeys = {
    # Up Arrow
    "\e[A" => (s,o...)->(LineEdit.edit_move_up(s) || LineEdit.history_prev(s, LineEdit.mode(s).hist)),
    # Down Arrow
    "\e[B" => (s,o...)->(LineEdit.edit_move_up(s) || LineEdit.history_next(s, LineEdit.mode(s).hist))
  }

  Base.active_repl.interface = REPL.setup_interface(Base.active_repl; extra_repl_keymap = mykeys)
```

可供使用的按键和操作请参阅 ``base/LineEdit.jl``.

Tab 补全
-------

在 Julia REPL (或者帮助模式下的 REPL), 可以输入函数或者类型名的前几个字符, 然后按 Tab 键来显示可能的选项::

```
  julia> stri
  stride     strides     string      stringmime  strip

  julia> Stri
  StridedArray    StridedVecOrMat  String
  StridedMatrix   StridedVector
```

Tab 键也可以使 LaTeX 数学字符替换成 Unicode 并且显示可能的选项::

 

```
 julia> \pi[TAB]
  julia> π
  π = 3.1415926535897...

  julia> e\_1[TAB] = [1,0]
  julia> e₁ = [1,0]
  2-element Array{Int64,1}:
   1
   0

  julia> e\^1[TAB] = [1 0]
  julia> e¹ = [1 0]
  1x2 Array{Int64,2}:
   1  0

  julia> \sqrt[TAB]2     # √ is equivalent to the sqrt() function
  julia> √2
  1.4142135623730951

  julia> \hbar[TAB](h) = h / 2\pi[TAB]
  julia> ħ(h) = h / 2π
  ħ (generic function with 1 method)

  julia> \h[TAB]
  \hat              \heartsuit         \hksearow          \hookleftarrow     \hslash
  \hbar             \hermitconjmatrix  \hkswarow          \hookrightarrow    \hspace
```
