# 0 序言
公司上线git管理代码仓库，半年有余。由于长久没用，渐渐生疏。虽然之前略有接触，但是终归浅尝辄止。

本篇及后续几篇，将对《git pro》中的知识进行总结归纳，力求让故事浅显易懂，凸显重点。

# 1 暂存（stage），提交（commit)
暂存（stage）操作，计算当前目录下，所有子目录中，每个改动文件的校验和（使用SHA-1校验码）；然后，针对每个文件，生成一个叫做“blob”的数据结构。这个结构就是当前版本下，对应文件的快照（snapshot）。

若提交(commit)被暂存的文件，则git会生成两个结构，一个是“树结构(tree)”，该数据对象存储指针，指向“blob”；另一个是“commit”结构，该数据对象指向“树结构”。这样，我们就把当前版本下，所有的文件改动，按照结构化方式保存起来。

图片pic-1展示了blob，tree，commit三层数据结构之间的关系，其记录了一次提交操作。该提交操作记录了三个文件被修改。

如果我们多次提交，将commit抽出来看，每个提交对象（commit对象）有一个指针，指向上一次的提交对象（父对象），这样就得到pic-2的一条链表，链表的每个元素就是一个commit。

# 2 分支（branch）

每一个分支（branch），实际上是一个指针，指向不同的commit对象，就出现了不同的版本。git初始化代码库时，会自动创建一个叫做master的branch（主分支），我们也可以将master分支改为其他名字，比如main分支。

图pic-3显示，在一个commit对象的链表上，三个分支master, v1.0，HEAD都指向了commit-“f30ab”。

由于我们可以创建许多分支，而某一时刻，我们只能在一个分支上工作，所以，“HEAD”分支指示了我们当前工作的分支。

现在我们来练习一下。

首先，如图pic-4，我们正处在master分支上，此时，创建一个新分支testing。执行下面命令，就会有master和testing同时指向commit-f30ab。

> $ git branch testing

但是，branch只是创建了一个新分支，由于HEAD指针仍然指向master分支，因此，执行下面命令，我们就将HEAD切换到testing分支。

> $ git checkout testing

如果需要查看所有的commit和分支情况，则使用下面命令：

> $ git log --oneline --decorate
f30ab (HEAD, master, testing) add feature \#32 - ability to add new
34ac2 fixed bug \#1328 - stack overflow under certain conditions
98ca9 initial commit of my project

所以，切换branch的本质，是在各个branch之间，移动HEAD指针。每次移动过程中，git做了两件事：HEAD指针指向目标branch；修改涉及到的所有文件。

# 3 出现分歧
在图pic-2中，我们假设用户只对一个branch进行操作，所以，我们得到一个线性结构。但是如果有多个branch，就会出现树形结构。

在pic-5中，分支master和分支testing在commit-f30ab的时候，还是一样的。但是，两个branch修改了不同的文件，或者对同一文件做了不同的修改，导致链表在commit-f30ab的后面，出现了分歧。这样，因为不同branch的提交操作，出现了不同的commit路线。

执行下面命令，可以查看当前的commit情况。

> $ git log --oneline --decorate --graph --all
* c2b9e (HEAD, master) made other changes
| * 87ab2 (testing) made a change
|/
* f30ab add feature \#32 - ability to add new formats to the
* 34ac2 fixed bug \#1328 - stack overflow under certain conditions
* 98ca9 initial commit of my project
