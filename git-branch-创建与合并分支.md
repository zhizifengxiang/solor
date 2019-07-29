# 0 序言
上一篇文章介绍了commit、branch和当前branch的概念。本篇将重点介绍分支，commit由一个线性结构，变成树形结构，并重新回到线性结构的操作过程。


# 1 出现分歧
在图pic-1中，我们假设用户只对一个branch进行操作，所以，我们得到一个线性结构。但是如果有多个branch提交commit，就会出现树形结构。

在pic-2中，分支master和分支testing在commit-f30ab的时候，还是一样的。但是，两个branch修改了不同的文件，或者对同一文件做了不同的修改，导致链表在commit-f30ab的后面，出现了分歧。这样，因为不同branch的提交操作，出现了不同的commit路线。

执行下面命令，可以查看当前的commit情况。

> $ git log --oneline --decorate --graph --all
\* c2b9e (HEAD, master) made other changes
| * 87ab2 (testing) made a change
|/
\* f30ab add feature \#32 - ability to add new formats to the
\* 34ac2 fixed bug \#1328 - stack overflow under certain conditions
\* 98ca9 initial commit of my project

# 2 背景
现在，我们正编写一个软件。出现了如下情景：

1. 我们将master分支认为是“主干”，每次软件发布，都会从master中抽取代码。
2. 为了不影响主干代码，我们创建了一个工作分支myBranch，用来添加新功能。
3. QA（quality assurance）测试发现master中有bug，需要紧急修复。于是，我们创建一个hotfix分支。

这样，如图pic-3所示，我们目前有三个分支指针：
1. master指针指向较旧的C2.
2. myBranch指针指向较新的C3.1。
3. hotfix指向较新的C3.2。

由于C3.1和C3.2出现了分歧，因此myBranch和hotfix出现了不同分支。

# 3 创建分支
为了实现上面的分支结构，我们需要执行下面的步骤。假设现在我们在master分支上（HEAD指向master），master指向C2。

1. 创建并切换myBranch分支。
> $ git checkout -b myBranch

2. 在myBranch提交一些修改，myBranch移动到C3.1。
> $ git add myFile.txt
> $ git commit

3. 修复hotfix。由于是master分支上出现的问题，所以首先要移动到master分支，再从master分支处，创建新的分支hotfix。
> $ git checkout master
> $ git checkout -b hotfix
> // modify something
> $ git add hotfix.txt
> $ git commit

4. 此时，HEAD指针指向hotfix，版本结构呈现树状结构。

# 4 合并分支
hotfix是一个应急的临时分支，在修复完问题后，应该马上执行下面命令，将hotfix合并到master分支。
> $ git checkout master
> $ git merge hotfix
