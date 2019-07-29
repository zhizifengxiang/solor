# 0 序言



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
