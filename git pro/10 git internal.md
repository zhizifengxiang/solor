10.1
1, Git是一个内容寻址型文件系统（content addressable file system）—— key-value(content)
2，其提供了大量底层的shell语句，来作为version control system的工具链。
3，plumbing：底层的命令——用于脚本编程；porcelain:高级命令

git init初始化.git目录，该目录包含所有的信息。下面是.git目录中的文件结构：

description: 用于gitWeb显示
config： 含有项目相关的配置选项
info/: 包含了.gitignore文件中指出的一些需要全局忽略的文件
hooks/:可以进一步执行的程序

下面是git的核心
HEAD:指向当前被checkout的branch
index:存储stage的信息
objects/:存储了数据库中的所有内容
refs/:存储指向commit的对象，包括branch, tag, remote


10.2
plumbing命令：git hash-object：take some data, store into .git/objects 目录， return unique key which can-be used to refer data-object.

1, 初始化git init
2, 创建一个内容对象：
echo "test content " | git hash-object -w --stdin
d670460b4b4aece5915caf5c68d12f560a9fe3e4 返回的一串40字节的SHA-1)
此时，在objects目录下生成一个文件：objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4

3, git cat-file -p d670460b4b4aece5915caf5c68d12f560a9fe3e4
打印刚才存的内容

4，创建一个文件，写入内容，并存起来，然后再追加内容，再次保存。
$ echo 'version 1' > test.txt
$ git hash-object -w test.txt
83baae61804e65cc73a7201a7252750c76066a30

$ echo 'version 2' > test.txt
$ git hash-object -w test.txt
1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
经过上面两个操作，生成两个文件。使用git cat-file就可以查看两次提交的内容。

5，上面生成的所有SHA-1文件，我们称作blob。可以查看对应类型：
$git cat-file -t 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
blob

Tree
git 中tree类似文件系统中的目录，blob类似文件系统中的inode。每个tree包含多个SHA-1条目，来索引blob，或者subtree，并记录相关的mode, type, filename.可以查看对应的tree
git cat-file -p master^{tree}
100644 blob a906cb2a4a904a152e80877d4088654daad0c859      README
100644 blob 8f94139338f9404f26296befa88755fc2598c289      Rakefile
040000 tree 99f1a6d12cb4b6f19c8655fca46c3ecf317074e0      lib

master^{tree}指向master branch的最后一次commit的tree object。注意，上面的lib指向一个subtree。
$ git cat-file -p 99f1a6d12cb4b6f19c8655fca46c3ecf317074e0
100644 blob 47c6340d6459e05787f644c2447d2595f5d3a54b      simplegit.rb


Commit Objects

Objects Storage
