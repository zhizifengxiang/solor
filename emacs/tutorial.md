# 1 前序
学生时代在Linux上实验，所以大致学习emacs的基本用法。工作后，转战qt creator，emacs也荒废掉。
这次重新捡起来的触动是，远程登陆工作服务器，IDE开启实在慢，所以再次重归emacs，希望能用新的角度使用学习Emacs的强大功能。

本篇首先介绍关键术语，光标移动组合快捷键，然后进一步介绍文本的删除、剪切、粘贴、撤销等基本功能。

# 2 术语与光标移动
所有组合快捷键，都是以ctrl 或 Alt开头，并加上字符。为了简便，下面 **C** 表示control，**M** 代表alt。

### 1 光标移动
光标移动共分：上、下、前（左）、后（右），四个方向。每个方向移动有不同粒度。比如向前移动，可以移动一个字符、一个单词，或者一个句子。

|键盘组合|功能|补充|
|--------|-------|-------|
|C-v|向上滚动页面|view|
|M-v|向下滚动页面|view|
|C-l（是L的小写）|（多按几次）将当前行移动到页面顶部/中部/底部|location|
||||
|C-p|移动光标到上一行|previous line|
|C-n|移动光标到下一行|next line|
|C-b|向左移动光标 **一个字符** |backwards|
|M-b|向左移动光标 **一个单词**|backwards|
|C-f|向右移动光标 **一个字符**|forwards|
|M-f|向右移动光标 **一个单词**|forwards|
||||
|C-a|移动光标到当前行首|向左移动|
|M-a|移动光标到当前句首|向左移动|
|M-<|移动光标到文档最开始|向左移动|
||||
|C-e|移动光标到当前行尾|向右移动|
|M-e|移动光标到当前句尾|向右移动|
|M->|移动光标到文档最末尾|向右移动|

### 2 辅助命令
|键盘组合|功能|补充|
|--------|---|----|
|C-u number otherOp|重复number次操作otherOp|比如 “C-u 2 C-f”：向右移动光标两次|
|C-g|中断当前命令|比如上面的"C-u 2 C-f", 我们只输入"C-u 2"，按下“C-g”，就会中断命令输入|
|C-x 1(数字1)|回到默认主会话窗口，并删除其他所有会话窗口||

# 3 编辑修改操作
对文档修改操作主要包括：插入、删除、剪切、粘贴、段选择、撤销等。

|键盘组合|功能|补充|
|-------|----|----|
|C-d|删除前面的字符，同backspace||
|M-d|剪切一个单词|emacs称剪切为kill|
|C-k|剪切一行||
|M-k|剪切一个句子||
|C-space|设置起始点，移动光标后选中一段文本||
|C-y|粘贴刚才剪切的内容|emacs称粘贴为yank|
|M-y|切换粘贴的内容|C-y粘贴最近的剪切操作，M-y进行切换|
|C-/|撤回上一次操作|该操作只针对两种情况：修改文本，每次撤销会累计20个字符。也可以：C-_, C-x u|

# 4 文件和缓存（buffer）

### 1 文件
通过下面命令，如果指定文件名存在，则打开文件。如果不存在，则创建文件。这个命令会让Emacs打开一个叫做minibuffer的窗口。将目标文件名输入到这个窗口中即可。
> C-x C-f

使用下面命令保存文件。 Emacs在保存文件的过程中，会在文件名后面加一个“~”。
> C-x C-s

### 2 缓存（buffers）
打开文件后，文件内容会在内存中缓存，缓存区域叫做buffer，我们通过下面命令显示所有buffer。
> C-x C-b

使用下面命令，可以切换不同的buffer。
> C-x b 输入相应的buffer

下面命令会将buffer中的内容保存到文件中。
> C-x s

# 5 扩展命令
emacs有两种方式触发拓展命令。
> C-x 后面为单字符命令
M-x 后面为多字符的命令

上面以进见过 "C-x C-f" 和 "C-x C-s"。下面介绍其他命令。
> C-x C-c 结束当前session，询问是否保存文件修改
> C-x C-z 将Emacs转入后台，只有在emacs运行在命令行时有用。可以在shell中使用"%emacs"或者"fg"来唤醒。

对于 "M-x"，replace-string 命令让用户以交互方式，替换某个字符。用户可以输入部分命令，并使用“tab”键来补全，显示提示等。
> M-x replace-string

# 6 自动保存
emacs会自动保存当前修改文档。其会在原始文件名两侧加“#”。比如：hello.txt，自动保存的文件名为：“#hello.txt#”。如果意外退出，我们使用下面命令，恢复之前修改。
>M-x recover-file

#7 屏幕的下半部分

### 1 命令提示
对于一些比较长的命令，emacs会在屏幕下方给出提示，回显出对应的命令。

### 2 模式
位于编辑阅读区的下方有一个显示类似如下信息的长条，
> -:\*\*-  TUTORIAL       63% L749    (Fundamental)

上面的“星号”表示当前文档被修改；后面是文件名；后面是阅读百分比和行号。
最后是编辑区所在的“模式”。

默认模式为“fundamental”，其为"major mode"的一种。通过下面命令可以切换不同的mode。每种mode命令集大致相同，在不同mode下，每个命令的行为稍有不同。
> M-x fundamental-mode
> M-x text-mode

查看当前major mode下的帮助文档，输入下面的命令。
> C-h m

还有一些minor mode，其是对环境进行修改。每个minor mode之间，以及minor mode和major mode之间没有任何联系。

Auto-fill-mode作为minor, 如果某行太长，其会让单词自动换行。打开和关闭都使用如下相同命令, 这种操作叫做“toggle the mode”
> M-x auto-fill-mode

auto-fill-mode通常每行长度为70个字符，使用下面命令，可以设置行所容纳的最大字符数量。
> C-x f

# 8 搜索和替换
> 增量式搜索：Emacs可以随着用户输入关键词，对当前文本进行动态搜索。

下面命令启动搜索，输入完关键词，按Enter或者移动光标，结束搜索。
> C-s 向文档末尾方向搜索
> C-r 向文档头部方向搜索

# 9 窗口系统

### 1 多个窗口（windows）
下面命令为多窗口操作命令：

|命令|功能|补充|
|-----|-----|----|
|C-x 2|在当前窗口分割出一个window||
|C-x o|将光别切换到另一个window中||
|C-M v|对另一个页面向下翻页|meta可以用esp键代替，但是按键顺序是esp-C v|
|C-x 4 C-f|在新的window中打开文件，且光标移动到新window中||

### 2 多个框架（frame）
一个frame包含多个window，以及window配套的菜单，滚动条等等。
> M-x make-frame # 创建新的frame

### 3 recursive edit
不明白怎么回事，只说下现象和解决方案。
当"(fundamental)"变成"[(fundamental)]",此时就进入了recursive edit模式。多次按“esc”即可退出。

# 10 寻找更多文档
下面命令可以获得针对某个命令的信息。

|命令|功能|补充
|----|-----|----|----|
|C-h 命令选项|显示某个命令的文档||
|C-h C-h|显示所有可以获得信息||
|C-h c 命令的描述|比如 "C-h c C-p"会显示C-p命令的简短描述||
|C-h k 命令的描述|获得某个命令的详细说明||
|C-h f 函数名称|某个函数的描述|比如"C-h f previous-line"|
|C-h v 某个变量|某个变量的描述||
|C-h a|输入某个字符，Emacs列出所有相关的命令|比如：C-h a file|
|C-h i|打开info文档||
