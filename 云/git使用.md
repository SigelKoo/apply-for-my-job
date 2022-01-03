# 数据模型

### 初始化仓库

我们首先要在自己的工作目录下初始化一个空的git仓库

```text
git init
```

git会告知我们已经在当前的目录下创建了一个.git目录，我们来看看这个.git长什么样子。

```text
$ tree .git/
.git
|-- HEAD
|-- config
|-- description
|-- hooks
|   |-- applypatch-msg.sample
|   |-- commit-msg.sample
|   |-- fsmonitor-watchman.sample
|   |-- post-update.sample
|   |-- pre-applypatch.sample
|   |-- pre-commit.sample
|   |-- pre-push.sample
|   |-- pre-rebase.sample
|   |-- pre-receive.sample
|   |-- prepare-commit-msg.sample
|   |-- update.sample
|-- info
|   |-- exclude
|-- objects
|   |-- info
|   |-- pack
|-- refs
    |-- heads
    |-- tags

8 directories， 15 files
```

其中一些文件和目录是不是看着有些熟悉，现在我们主要还是看`objects`这个目录，现在它是空的，但是一会儿我们就会改变它。

### 提交文件

首先我们创建一个`Main.java`文件

```text
touch Main.java
```

然后输入一部分内容

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello World");
    }
}
```

然后以同样的方式在准备一个README.md文件

```text
touch README.md
```

向文件中输入以下内容

```text
this is my first java project!
```

现在add并且commit他们到仓库

```text
git add .
git commit -m 'Initial Commit'
```

### 模型的创建

现在看上去没啥特殊的，现在我们回过头来在看看`.git/objects`目录下已经存在了一些子文件夹以及文件了

```text
.git/objects
|-- 84
|   -- 705622ee44f2afbb21087ca7d81fda01fccded
|-- 95
|   -- fc1236534b6f73930367f02895467040f47d4a
|-- b0
|   -- 81e51f448387e72a3e3551ba8610eedc172e60
|-- f1
|   -- a8b89f50a2fd8287578daa2b0374adf3cad8aa
|-- info
|-- pack
6 directories， 4 files
```

需要注意的是在你的电脑上目录和文件名称和我这里是不一样的。

### blob object的创建

在`.git/objects`下我们注意到每个目录的名称只有2个字符长度，Git为每个对象生成一个40个字符的校验和（SHA-1）哈希，该校验和的前两个字符用作目录名，另外38个字符用作文件（对象）名。当我们提交一些文件时，git创建的第一类对象是**blob object**，在我们的例子中是两个，每一个**blob object**对应我们提交的每一个文件:

![img](https://pic4.zhimg.com/80/v2-ad1674d1c404535aa13f3e25667a149f_720w.jpg)

blob object包含文件的快照以及拥有文件校验和。

### tree object的创建

git创建的另外一种对象是`tree object`，在我们的例子中只有一个，它包含我们项目中所有文件的列表，其中包含分配给它们的blob object的指针（这就是git如何将文件与blob object相关联）

![img](https://pic2.zhimg.com/80/v2-f4bfd58c97fb639953887000cc63a1bd_720w.jpg)

### commit object的创建

最后git还创建了一个commit object，该对象具有指向它的tree object的指针（以及一些其他信息）

![img](https://pic2.zhimg.com/80/v2-231f6226fc7c6a9bf310d8c12eaa9ea5_720w.jpg)

这个时候在来看以下objects目录下的结构就清晰多了

```text
.git/objects
|-- 84
|   -- 705622ee44f2afbb21087ca7d81fda01fccded
|-- 95
|   -- fc1236534b6f73930367f02895467040f47d4a
|-- b0
|   -- 81e51f448387e72a3e3551ba8610eedc172e60
|-- f1
|   -- a8b89f50a2fd8287578daa2b0374adf3cad8aa
|-- info
|-- pack
```

### 验证模型的准确性

上面画出了模型图，但是你以为我这个模型是自己猜的吗？我又是如何确定哪个是blob object？哪个是tree object？哪个是commit object的呢？接下来就是见证奇迹的时刻了。

使用`git log`命令我们可以查看我们的提交历史

```text
commit f1a8b89f50a2fd8287578daa2b0374adf3cad8aa (HEAD -> master)
Author: zhu.yang <zhu.yang@xxx.com>
Date:   Tue Jan 8 10:12:06 2019 +0800
    Initial Commit
```

根据我们前面说的命名约定，我们可以在objects中发现`f1a8b89f50a2fd8287578daa2b0374adf3cad8aa`这个对象。想要查看文件内容我们不能简单的使用`cat`命令，因为这些不是纯文本文件，但是好在git给我们提供了一个cat-file命令。

```text
git cat-file commit f1a8b89f50a2fd8287578daa2b0374adf3cad8aa
```

可以通过它获取到commit object中的内容

```text
tree 95fc1236534b6f73930367f02895467040f47d4a
author zhu.yang <zhu.yang@xxx.com> 1546913526 +0800
committer zhu.yang <zhu.yang@xxx.com> 1546913526 +0800
Initial Commit
```

从上面可以看到commit指向tree object并且我们可以使用`git ls-tree`命令来检查下其中的内容

```text
git ls-tree 95fc1236534b6f73930367f02895467040f47d4a
```

正如我们说预料的一样，其中包含了指向blob object的文件列表

```text
100644 blob 84705622ee44f2afbb21087ca7d81fda01fccded    Main.java
100644 blob b081e51f448387e72a3e3551ba8610eedc172e60    README.md
```

如果想要查看Main.java中的内容则使用`cat-file`命令即可

```text
git cat-file blob 84705622ee44f2afbb21087ca7d81fda01fccded
```

我们可以看到其中返回了Main.java文件的内容

```text
public class Main {
        public static void main(String[] args) {
                System.out.println("Hello World");
        }
}
```

### 修改文件时模型的改变

现在我们修改一下main.java然后重新提交一下

![img](https://pic3.zhimg.com/80/v2-08846718f84c7b10f484b73adc05b3b6_720w.jpg)

正如我们看到的一样，git以快照的方式为`Main.java`新建了一个blob object，由于`README.md`没有被修改，因此不会为其创建新的blob object。而且**git会重用现有的blob object**。

现在，当git创建一个tree object时，分配给`Main.java`的blob指针会被更新，并且分配给`README.md`的blob指针将保持与前一个提交树中的相同。

![img](https://pic4.zhimg.com/80/v2-722290cfc7fd2e050d4668d7fccd2717_720w.jpg)

在最后，git创建一个commit object并指向它的tree object。同时还有一个指向它的父提交对象的指针(每个提交除了第一个提交至少还有一个父提交)

![img](https://pic2.zhimg.com/80/v2-5235d65d461504d5f8ca3c66b6046199_720w.jpg)

到现在为止我们已经知道了git是如何处理文件的新增以及编辑的，唯一还遗留的就是如何处理删除了，我们先删除Main.java：

![img](https://pic2.zhimg.com/80/v2-62066040a92365f5d67521f893fda8e5_720w.jpg)

请注意上图中红色的连线，我们发现删除同样也是非常简单，只需要删除tree object指向blob object的指针即可。在这种情况下我们在新的提交中删除了Main.java，因此我们的提交的树对象不再具有指向表示Main.java的blob object的指针。

### 模型对文件夹的处理

我们提供的这个数据模型还有一个附加功能-tree object是可以被嵌套的（它们可以指向其他树对象），你可以这样想：每个blob object代表一个文件，每个树对象代表一个目录，所以如果我们有嵌套目录，我们就有嵌套的tree object。

由于上面的图已经是提交多次结果画出来的了，再在上面的基础上画结构就不是那么清晰了，这次我重新初始化一个仓库来演示，现在该仓库下存在存在的数据如下：

```text
|-- README.md
`-- app
    `-- user.json
```

然后提交，最后可以看到如下的数据模型

![img](https://pic4.zhimg.com/80/v2-ccfb5235811990b05dfaee8126213f3f_720w.jpg)

Git使用blob object以及tree object来重现项目的文件夹结构。到这里我相信你肯定对git的数据模型有了较为深入的了解，它真的是很简单，我相信基于它再去学习Git一定会是事半功倍。

# 分支是如何实现的？

在开发软件的时候，可能很多人会同时为同一个软件开发功能或者修复bug，但是如果都在主分支来进行开发，引起冲突的概率将会大大增加，而且也不利于维护，如果你同时修改多个bug该怎么办？所幸，git的分支功能很好的帮助我们解决了这个问题，它可以帮助我们同时进行多个功能的开发和版本管理。

这次我会只显示commit objects来简化它，并且为了让它更容易理解我会给他们取个别名来代替原本的检验和，所以对于提交记录，我们会得到一个像下面这样的图。

![img](https://pic4.zhimg.com/80/v2-6cb1f7e48de4f5ee5ca812d1b584cc53_720w.jpg)

熟悉图论的应该注意到了上面的是一个有向无环图(DAG)，这意味着从一个节点开始沿着边的方向不会经过相同的节点。在我们的例图中可以清晰的发现存在三个不同的分支，我们分别用红色(包含A，B，C，D，E)，蓝色(A，B，F，G)，以及绿色(A，B，H，I，J)来标记它们

![img](https://pic1.zhimg.com/80/v2-e4fbe2370eaad412a44aa565f2d4ff24_720w.jpg)

这就是定义分支的一种方式-包含所有的提交列表。但是这不是git使用的方式，git使用更简单更便宜的方式，git只跟踪分支上的最后一次提交，而不是持有某个分支的所有列表并更新它们，只需要知道分支的最后一次提交，然后根据图的有向边就可以获取整个提交列表。例如要定义我们的蓝色分支，只需要知道蓝色分支的最后一次提交是G，如果我们需要蓝色分支包含的所有提交的列表，就从G沿着图有向边遍历即可。

![img](https://pic1.zhimg.com/80/v2-a9254fc9aed16ebd3888a958cc7c99f4_720w.jpg)

这就是git管理分支的方式，通过保持执行提交记录的指针即可，接下来我们会进行一个演示。首先通过`git init`初始化一个空仓库，然后查看.git目录下存在的文件

```text
.git
|-- HEAD
|-- config
|-- description
|-- hooks
|   |-- applypatch-msg.sample
|   |-- commit-msg.sample
|   |-- fsmonitor-watchman.sample
|   |-- post-update.sample
|   |-- pre-applypatch.sample
|   |-- pre-commit.sample
|   |-- pre-push.sample
|   |-- pre-rebase.sample
|   |-- pre-receive.sample
|   |-- prepare-commit-msg.sample
|   |-- update.sample
|-- info
|   -- exclude
|-- objects
|   |-- info
|   |-- pack
|-- refs
    |-- heads
    |-- tags
```

这次我们关注`refs`这个子目录，这个地方是git保留分支指针的地儿。当我们没有提交任何东西的时候，`refs`目录下只存在两个空目录，现在我们提交几个文件。

```
echo "Hello Java" > helloJava.txt
git add .
git commit -m "Hello Java Commit"
echo "Hello Php" > helloPhp.txt
git add .
git commit -m "Hello Php Commit"
echo "Hello Python" > helloPython.txt
git add .
git commit -m "Hello Python Commit"
```

当我们执行`git branch`的时候我们可以看到下面这样的输出

```text
* master
```

意味着我们现在处于master分支上（这个是当我们第一次提交的时候git自动给我们创建的），此时`refs`目录下是这样

```text
.git/refs
|-- heads
|   `-- master
`-- tags
```

我们看到refs/heads子目录中有一个文件，它就像我们的分支一样被命名为master，我们使用cat命令查看下文件内容

```text
$ cat .git/refs/heads/master
49cd903b2bf247de040118ce60d1931ff587e801
```

而使用`git log`命令我们可以看到我们的提交记录是这样的

```text
commit 49cd903b2bf247de040118ce60d1931ff587e801 (HEAD -> master)
Author: zhu.yang <zhu.yang@xxx.com>
Date:   Tue Jan 8 17:48:36 2019 +0800
    Hello Python Commit

commit dd7c1bc9c125067f5658bcc6bc35567d07bc4f35
Author: zhu.yang <zhu.yang@xxx.com>
Date:   Tue Jan 8 17:48:31 2019 +0800
    Hello Php Commit

commit c6bd5c991dbcf9c50bbab682796ab3e06672f5a7
Author: zhu.yang <zhu.yang@xxx.com>
Date:   Tue Jan 8 17:48:30 2019 +0800
    Hello Java Commit
```

从上面可以看出来一个分支仅仅只是一个文本文件，其中记录了这个分支最后一次提交的校验和。也就是指向commit的一个指针

![img](https://pic2.zhimg.com/80/v2-e0f0b02375e51191c3f47079c987b4ed_720w.jpg)

现在我们新建一个`feature`分支并切换到新建的这个分支上面

```text
git checkout -b feature
```

使用tree命令在来看看.git/refs的样子

```text
.git/refs
|-- heads
|   |-- feature
|   |-- master
|-- tags
```

同样的我们使用cat命令查看下.git/refs/heads/feature文件的校验和

```text
$ cat  .git/refs/heads/feature
49cd903b2bf247de040118ce60d1931ff587e801
```

我们会发现和master文件中的内容一致，现在为止我们没有往feature分支提交任何内容

![img](https://pic3.zhimg.com/80/v2-549c8425077b73b62b62e5070c4177c6_720w.jpg)

这就是git创建一个分支那么快以及方便的原因所在，git仅仅只是创建了一个包含最近一次提交校验和的文件而已。

现在我们的仓库里面就有2个分支了，但是git怎么知道我们当前检出的分支是哪个分支呢？这里其实存在一个特殊的指针叫做**HEAD**，它之所以特殊是因为它并不指向具体的commit object，而是指向分支，git使用它来跟踪最近检出的分支。

```text
$ cat .git/HEAD
ref: refs/heads/feature
```

![img](https://pic2.zhimg.com/80/v2-3ea08c43eea50f2d7103a9ca6728c319_720w.jpg)

如果我们执行

```text
git checkout master
```

然后查看HEAD，会发现当前分支是master，然后HEAD会指向master

```text
$ cat .git/HEAD
ref: refs/heads/master
```

![img](https://pic2.zhimg.com/80/v2-0fa67e096499ad0b653bf30599c8c881_720w.jpg)

这就是git的分支模型，很简单但是很重要，了解它有助于理解在这个图上的其他操作

# 索引是如何实现的

从git的角度来看，文件的修改涉及到以下三个区域：工作目录，stage区（暂存区）以及本地仓库。

![preview](https://pic1.zhimg.com/v2-d7a6e44979626f7bbe0ab985c1d90b1c_r.jpg)

当我们对我们的项目做了一些修改（新增文件，删除文件，修改文件等），我们处理的就是我们的工作目录。这个目录是存在于我们电脑的文件系统上的。所有的修改都会保留在工作目录直到我们把它们加入到暂存区（通过git add命令）。

暂存区这是对下一次提交最好的表示方式，当我们执行`git commit`，git会获取暂存区中的修改，并将这些修改作为下一次的提交内容。暂存区的一个实际作用就是允许你调整你的提交，你可以向暂存区新增和删除修改直到你对你下一次的提交满意，这个时候你就可以用`git commit`提交你的内容了。

在提交修改后，它们就会进入`.git/objects`目录，在其中被保存为commit，blob以及tree objects

把暂存区认为是一个存储修改的真实区域并不准确，git没有专门的stage目录来存放这些文件的修改(blobs)，git有一个名为index的文件来跟踪这三个区域的修改：工作目录、暂存区以及本地仓库。

当我们添加修改到暂存区的时候，git会更新index文件中的信息，并且创建一个新的blob object，然后将它们放到与之前提交的记录所产生的其他blob相同的.git/objects目录中。

### index的变化

接下来我们就通过一个正常的git流程来演示下git如何使用的index。

首先在我们的仓库里面有master以及feature两个分支，如果我们执行下面的命令，会有三件事情发生。

```text
git checkout feature
```

**第一**.git会移动HEAD指针来指向feature分支，为了更加便于理解，我们只显示功能分支的最后一次提交。

![img](https://pic2.zhimg.com/80/v2-a123377d46ef46f0b7a22cf7bccfb0c9_720w.jpg)

**第二**.git将获取feautre分支指向的提交内容并将其添加到索引中

![img](https://pic3.zhimg.com/80/v2-8edeb4473b13d874ac1e05cb15718ae2_720w.jpg)

我们注意到index是一个文件而不是目录，所以git是没有往其中存储内容的，git只是存储我们仓库中每个文件的信息而已，类似于上面这样

```text
mtime : 上次更新时间
file : 文件名称
wdir : 工作目录中文件版本
stage : index中文件版本
repo : 仓库中的文件版本
```

文件版本以校验和来标识，如果两个文件有相同的校验和，那么它们就有一样的内容以及版本。

最后，git会将你的工作目录和HEAD指向的内容相匹配（它将使用树和blob对象重新创建项目目录的内容）

![img](https://pic3.zhimg.com/80/v2-4443c6b0f595844b600c6802beb864b6_720w.jpg)

所以，当你使用checkout的时候，工作目录，暂存区以及仓库都是相同的版本。我们来看看当我们编辑Main.java的时候会发生什么？

![img](https://pic1.zhimg.com/80/v2-099037c6988bfa0dab3a0083f4bb8378_720w.jpg)

现在仅仅只影响了我们的工作目录，但是我们运行下面的命令的时候

```text
git status
```

git 首先会更新index文件中Main.java的工作目录的版本

![img](https://pic3.zhimg.com/80/v2-bb8cf93cd9f43766009f90d2b9bdca2e_720w.jpg)

然后我们看到Main.java在工作目录和暂存区有不同的版本

![img](https://pic1.zhimg.com/80/v2-c6914168b8d3448d5a819e5ffcb309d0_720w.jpg)

然后git会提示我们

```text
On branch feature
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)
modified:   Main.java
no changes added to commit (use "git add" and/or "git commit -a")
```

这就表明工作目录的修改不在暂存区中（那么下一次的提交就不会包含Main.java的修改）

所以，执行以下命令将Main.java加入到暂存区

```text
git add Main.java
```

执行了上面这条命令，就又会发生两件事儿，第一.git会为Main.java创建一个blob object然后存储在.git/objects目录下，第二.会再次更新index文件

![img](https://pic2.zhimg.com/80/v2-9e8dbb1d4907e447ec0ed4e5d43357b5_720w.jpg)

这个时候我们再次执行命令

```text
git status
```

git会发现Main.java的暂存区的版本和工作目录版本一致，但是和仓库的版本不一致

![img](https://pic4.zhimg.com/80/v2-4f6018eac3627d743468fb11f625ae33_720w.jpg)

所以git就告知我们

```text
On branch feature
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)
modified:   Main.java
```

证明Main.java已经在暂存区，但是还没有提交到仓库。现在我们就可以提交我们的修改了

```text
git commit -m "add some code to Main.java"
```

git会做下面几件事儿: 1. 新增commit object和tree object，并把它们和执行git add时创建的blob object连接起来 2. 移动feature的指针到新的commit object 3. 更新index

![img](https://pic3.zhimg.com/80/v2-148689a835c93718d3bf571ba03e5846_720w.jpg)

好啦，现在我们的Main.java在所有区域都有相同的版本了

无论执行 `git add`还是`git commit`index文件都会变更，这也更好的证明了我们上述模型，当然index文件中的内容肯定没有那么清晰，它是一个二进制文件，如果想要查看它的内容就需要借助其他工具来实现。

# merge和rebase如何理解？

![img](https://pic3.zhimg.com/80/v2-1dab0facb508094cae00c4d555360b26_720w.jpg)

我们按照一个正常的开发流程来学习，我们现在要开始开发了，首先我们需要把远程仓库同步到本地

```text
git clone https://github.com/generalthink/git_learn.git
```

运行完上面的命令，在我们的当前工作目录就会有一个git_learn的文件夹，其中存在.git目录（git维护仓库的基本）以及和远程仓库一样的文件，现在我们的代码环境就和远程仓库一致了，我们就可以开始我们的开发流程了。

### 开发流程

一般来说，无论是开发新功能(feature)还是修改bug(issue)我们都不会直接在主干上进行开发，而是会在分支上进行开发，然后验证无误之后在同步到master。 现在突然有了一个bug我们要新开一个分支来解决这个bug，那么我们应该怎么做呢?

首先，新建一个bugFix分支，并切换到这个分支

```text
git branch bugFix
git checkout bugFix
或者
git checkout -b bugFix
```

![img](https://pic1.zhimg.com/80/v2-d87ffc1c46a9bfc1d866f140ccb58cc8_720w.jpg)

新建一个分支，只是多了一个指针指向当前最新的提交而已，后面所有的提交都基于当前的分支，星号代表当前分支。

然后，找到bug的原因，修改对应的文件，现在我们修改了几个文件，需要提交将它加入到我们本地仓库，同步到远程仓库会在后面的文章讲解。

1. 加入修改的文件到暂存区

   ```text
   git add *.java
   ```

   上面的操作实际上是使文件工作目录和暂存区的版本一致，但是现在和仓库的版本还不一致。想要查看具体存在哪些文件需要提交可以使用`git status`查看。

2. 提交文件到本地仓库

   ```text
   git commit -m "fix bug"
   ```

   `commit`命令实际上是让index中文件在暂存区和仓库的版本保持一致，经过这个步骤，工作目录、暂存区以及仓库的版本都是一致了，下面的图是提交前后commit objects的变化。

   ![img](https://pic4.zhimg.com/80/v2-af1e21eb8e7539e4c2f17f0eb1b3530f_720w.jpg)

   你在修改bug的同时，其他人已经往主干分支上提交了其他功能的代码或者你本地本来就存在两个不同的分支，所以修复了这个bug之后，我们的仓库看上去是这样的

   ![img](https://pic3.zhimg.com/80/v2-4e423d0f73aac9f970040c1ba7505d46_720w.jpg)

### 合并分支

这个bug修复后，被测试验证通过了，然后接下来想要把这个分支的代码合并到主干分支（或者合并两个不同的分支），常用的合并方式有2种

#### git merge

将bugFix合并到master分支上

```text
//切换到master分支
git checkout master

git merge bugFix
```

![img](https://pic3.zhimg.com/80/v2-e6287c2acdef25d0de2759dd371fbd72_720w.jpg)

看到了没有，master指向了一个拥有两个父节点的提交记录，如果从master开始沿着箭头向上看，在到达起点的路上会经过所有的提交记录，这意味着 master 包含了对代码库的所有修改。

这个时候如果你还想把master分支合并到bugFix分支也是可以的

```text
git checkout bugFix
git merge master
```

![img](https://pic1.zhimg.com/80/v2-027609c25e55af885e487d8e530f6908_720w.jpg)

因为 master 继承自 bugFix，Git 什么都不用做，只是简单地把 bugFix 移动到 master 所指向的那个提交记录。 现在所有提交记录的颜色都一样了，这表明每一个分支都包含了代码库的所有修改！

#### git rebase

第二种合并分支的方法是 git rebase。Rebase 实际上就是取出一系列的提交记录，“复制”它们，然后在另外一个地方逐个的放下去。 Rebase 的优势就是可以创造更线性的提交历史。

```text
git rebase master
```

![img](https://pic2.zhimg.com/80/v2-b9d34cc579cae4f8142ea5fef898d379_720w.jpg)

现在 bugFix 分支上的工作在 master 的最顶端，同时我们也得到了一个更线性的提交序列。 注意，提交记录 C3 依然存在（树上那个虚线节点），而 C3' 是我们 Rebase 到 master 分支上的 C3 的副本。 现在唯一的问题就是 master 还没有更新，下面咱们就来更新它吧。

```text
git checkout master
git rebase bugFix
```

![img](https://pic4.zhimg.com/80/v2-d867a2bc53ec96a2d0fc8884d65e8ca3_720w.jpg)

由于 bugFix 继承自 master，所以 Git 只是简单的把 master 分支的引用向前移动了一下而已。

# 如何自由的修改提交记录

讲了merge和rebase，我们已经可以在commit object构成的图（当然我更愿意把它看成一棵树）上面进行分支的合并了，在图上我们可以新增节点(git commit)，合并节点(merge或者rebase)，这篇我们就来讲解下移动和删除节点。

在讲移动和删除之前，我们先来认识下HEAD的分离。

### **分离的HEAD**

我们都知道，HEAD是指向当前分支的，而分离的HEAD就是让其指向了某个具体的提交记录而不是分支名。

现在我本地的提交记录是这样的

![img](https://pic4.zhimg.com/v2-c76ae1566001d8f680a787c0f2402927_b.webp)

当我们执行`git checkout 062704b1c3a814dfd95695aba3684c22e3f3fa85`之后HEAD就处于分离状态。

![img](https://pic2.zhimg.com/v2-2bf6b9b58b10d5456aabf2574ae3de29_b.webp)

![img](https://pic4.zhimg.com/v2-aacaba0323e3bb215f929892912f043f_b.webp)

### 移动节点

我们可以通过指定提交记录hash的方式移动指针的位置（无论是分支还是HEAD），然而实际中并没有那么直观的图给我们看，就不得不使用`git log`来查看提交记录的hash，然而hash又比较长，幸好git对hash的处理比较智能，我们只需要提供唯一标识提交记录的前几个字符就可以了，因此我们可以只输入`git checkout 0627`就可以检出提交记录了。

通过hash值来移动节点显然并不方便，所以git提供了相对引用，这样我们就可以从一个易于记忆的地方（比如bugFix分支或者HEAD）开始计算。

### **相对引用**

相对引用很给力，常用的两种用法

1. 使用`^`向上移动1个提交记录
2. 使用`~num`向上移动多个提交记录

![img](https://pic1.zhimg.com/v2-5595347c73ed5a5a38fb24b7fa3d8f3c_b.webp)

![img](https://pic2.zhimg.com/v2-c4e9d88f8ca11d82baadc3e82ad7a2a1_b.webp)

当我们执行`git checkout master^`的时候，我们的HEAD就指向了上一个提交记录（当前记录的是一个记录），注意这里移动的是提交记录（commit object），同理移动多个记录也是一样的。

使用相对引用最多的就是移动分支，我们可以命令直接让分支指向另外一个提交。

![img](https://pic2.zhimg.com/v2-dfb1b64c53ad53abe954f6b7a25390e5_b.webp)

现在master分支就指向了第一个提交，需要注意的是不能在当前分支操作当前分支的移动，否则你会有这样的错误

```text
fatal: Cannot force update the current branch.
```

完成移动之后并不会切换分支，仍然处于之前的分支。

### **任意移动**

如何能将提交树的commit object任意的移动？让我们的修改可以更加的随意，`git cherry-pick`就能做到。 现在我们想把bugFix分支上C2，C4的提交记录移动到master分支上，只需要执行

```text
git cherry-pick C2 C4
```

这个命令可以"复制"提交节点并在当前分支做一次完全一样的新提交

![img](https://pic1.zhimg.com/v2-b09d94415c131d1248dff5a799a83bb0_b.webp)

### **回退代码**

有的时候我们的代码提交错了，但是已经提交到git上去了，我想要回退怎么办？还好git提供了两种方法用来撤销变更----`git reset`以及`git revert`。

### **git reset**

`git reset`通过把分支记录回退几个提交记录来实现撤销改动，其实就是移动在图上的指针。

![img](https://pic3.zhimg.com/v2-4c52ac06bfc839a83d98858b2a4ed762_b.webp)

### git revert

![img](https://pic4.zhimg.com/v2-30064bbc531ef4c512ad20c9e7f8795b_b.webp)

我们本来是要撤销C2提交的，但是为什么还多了一个C2'提交呢？这是因为新提交记录C2'引入了更改--这个更改又是用来撤销C2这个提交的，也就是说C2'的状态于C1是相同的。revert之后就可以把更改push到远程仓库与别人分享了。

# 这些远端命令一定要记住

### git clone --- 克隆远端代码到本地

当我们进行开发的时候，开发流程是这样的：首先将远程仓库(中央仓库)的代码clone到本地，在本地进行开发，开发完成之后将代码提交到远程仓库。

远程仓库并不复杂，实际上它们只是你的仓库在另外一台计算机上的拷贝，我们可以通过网络和这台计算机通信--也就是增加或是获取提交记录。我们先通过命令将远端仓库clone到本地

```text
git clone https://github.com/generalthink/git_learn.git
```

执行命令之后git仓库就从远端clone到本地了，此时本地和远端的代码一致。执行了这个命令之后我们本地有什么变化呢？ 先查看我们现在存在哪些分支

```text
$ git branch -a
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/master
```

当我们执行`git clone`的时候， Git 会为远程仓库中的每个分支在本地仓库中创建一个远程分支（比如 origin/master）。然后再**创建一个跟踪远程仓库中活动分支的本地分支**，默认情况下这个本地分支会被命名为 master。

你可能注意到了我们除了本地的master分支，还多了origin/master的分支，这种类型的分支就叫远程分支，它反映了远程仓库在你上次和它通信的状态。还记得index那篇文章吗？index文件中记录了工作目录、暂存区、本地仓库的版本用于跟踪文件状态，那么远程仓库的状态由谁来维护呢？没错就是这个origin/master分支。

需要注意的是远程分支有一个特别的属性，当我们检出时，自动进入分离HEAD状态（此种状态下提交并不能影响origin/master分支）.git这么做是出于不能直接在这些分支上进行操作的原因，你必须在别的地方完成你的工作。

![img](https://pic2.zhimg.com/v2-f29666df1dfe0596dad7b36eb91f950d_b.jpg)



远程分支的命令规范是这样的：`<remote name>/<branch name>`，当我们使用git clone某个仓库的时候，git已经帮我们把远程仓库的名称设置为origin了。

可以使用下面的命令来查看远程库对应的简短名称

```text
$ git remote -v
origin  https://github.com/generalthink/git_learn.git (fetch)
origin  https://github.com/generalthink/git_learn.git (push)
```

上面我们把远端仓库同步到了本地，远端和本地的代码就是一致的了（本地仓库中两个分支都指向的最新的提交记录）。

### **分支跟踪**

当我们将本地master分支的代码push到远程的master分支（同时会更新远程分支origin/master）的时候，我们只需要执行`git push`就可以了，就好像git知道我们它们是关联起来的！

其实master和origin/master的关联关系是由分支的"remote tracking"属性决定的，master被设定为跟踪origin/master -- 表示master指定了推送的目的地以及拉取后合并的目标。

可以让任意分支跟踪 origin/master， 然后该分支会像 master 分支一样得到隐含的 push 目的地以及 merge 的目标。 这意味着你可以在分支 bugFix上执行 git push，将工作推送到远程仓库的 master 分支上。我们可以通过下面的两种方法创建一个bugFix的分支，它跟踪远程分支origin/master

- `git checkout`
  git checkout -b bugFix origin/master

- `git branch`
  需要保证bugFix分支已经存在
  git branch -u origin/master bugFix

  如果当前就在bugFix分支上，命令可以优化成为

  git branch -u origin/master

这样bugFix就会跟踪origin/master了，当我们推送代码到远端的时候就可以不用指定目的地了，直接执行`git push`就可以将bugFix分支的代码推送到远端的master分支了。

通过`git branch -vv`命令可以查看本地分支关联的远程分支的对应关系

```text
$ git branch -vv
* bugFix     215d0ff [origin/master] add bugFix.md
  foo        e2240d6 [origin/master: behind 2] add foo.md
  master     7b7adf6 [origin/master: behind 5] Revert "bugFix"
  newFeature 3136c72 [origin/master: behind 3] add test2.md
```

当你通过上面的命令设置了跟踪关系之后执行`git pull`的时候你可能会有这样的报错信息：

```text
fatal: The upstream branch of your current branch does not match
the name of your current branch.  To push to the upstream branch
on the remote， use
    git push origin HEAD:master
To push to the branch of the same name on the remote， use
    git push origin newFeature
To choose either option permanently， see push.default in 'git help config'.
```

这全是因为`git config push.default`设置，默认是simple(从git 2.0开始)，这表示当本地分支和远端分支的名称不一样的时候git会拒绝提交。为了让其允许push到它跟踪的分支，需要重新设置这个参数

```text
git config --global push.default upstream
```

`--global`只改变当前git仓库的配置。关于push.default有哪些值可以通过`git help config`命令查看。

设置完成之后，在执行`git push`命令就可以直接将bugFix分支的内容提交到master分支上了。

```text
$ git push
Enumerating objects: 4， done.
Counting objects: 100% (4/4)， done.
Delta compression using up to 8 threads.
Compressing objects: 100% (2/2)， done.
Writing objects: 100% (2/2)， 327 bytes | 327.00 KiB/s， done.
Total 2 (delta 1)， reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1)， completed with 1 local object.
To https://github.com/generalthink/git_learn.git
   e2240d6..215d0ff  bugFix -> master
```

### **开发流程**

上面我们的仓库已经和远端一致之后，我们就可以开发了，现在我们要修改一个bug，做法就是本地新建一个bugFix（命名规范各个公司是不一样的，一般配合bts工具）分支， 然后在这个分支上面修改，修改完成之后将修改提交到线上服务器，然后线上jekins会自动跑一些脚本，验证你提交的代码，或者检测冲突，有冲突就需要合并。等到一切没有问题之后就可以合并master去了，当然我们自己开发是没有这么复杂的，因此我们就通过直接将bugFix分支的代码推送到远端master分支就可以了

### **提交代码到远程仓库**

`git push`命令负责将我们的变更上传到指定的远程仓库，现在直接将我们的代码推送到远程分支

```text
$ git push origin bugFix:master
To https://github.com/generalthink/git_learn.git
! [rejected]        bugFix -> master (fetch first)
error: failed to push some refs to 'https://github.com/generalthink/git_learn.git'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally。 This is usually caused by another repository pushing
hint: to the same ref。 You may want to first integrate the remote changes
hint: (e。g。， 'git pull 。。。') before pushing again。
hint: See the 'Note about fast-forwards' in 'git push --help' for details。
```

执行命令发现报错了，为什么会这样呢？是因为你在开发的过程中，你的同事也在开发，并且他开发的代码已经合并到了主干上，这个时候你本地的代码就不是最新的了，这个时候如果你往远端push代码，那么git就会给你抛出这个错误提示。

此时，远端和本地分支的情况是这样的。

![img](https://pic1.zhimg.com/v2-c71d255a3288b24aa4e50d2fbaa865b4_b.jpg)



可以看到远端master节点和本地的origin/master指向的并不是同一个commit object，而我们执行的git push命令显然不能智能的帮助我们合并。此时我们应该先同步远端更改到本地，合并这些修改，然后在push到主干。

### **git fetch -- 同步代码到本地**

下面的命令用来和远端进行通信，把远端的代码先同步到本地

```text
git fetch origin master
```

![img](https://pic4.zhimg.com/v2-e1b25175be59f3c7d9e74651a62610cb_b.jpg)

git fetch完成了仅有的但是特别重要的两步 **1. 从远程仓库下载本地仓库中缺失的提交记录** **2. 更新远程分支指针(如 origin/master)**

现在本地仓库的远程分支更新成了远程仓库相应分支最新的状态。它通常通过互联网(http://或者git://协议)与远程仓库通信。 需要注意的是**git fetch 并不会改变你本地仓库的状态。它不会更新你的 master 分支，也不会修改你磁盘上的文件。**

### **合并代码**

现在我们已经获取到了远程的数据，只需要将这些变化更新到我们的工作目录中就可以了，我们只需要像合并本地分支那样来合并远程分支就可以了，我们可以通过以下三种方式来完成合并 1. git cherry-pick origin/master 2. git rebase origin/master 3. git merge origin/master

实际上，由于先抓取更新再合并到本地分支这个流程很常用，因此 Git 提供了一个专门的命令来完成这两个操作。它就是我们的 git pull。

```text
git pull ===== git fetch;  git merge origin/master

git pull --rebase == git fetch;git rebase origin/master
```

现在本地的远程分支和远程仓库的代码保持了一致，我们终于可以使用服务器`git push origin bugFix:master`提交我们的代码了。

看着还挺不错，现在我们可以开心的使用git工作了，但是需要记住的是git push之前一定要保证要本地的远程指针一定要和远端一致，要不然你就只有等着报错吧。

### **远程命令语法**

上面我们看到了和远程仓库交互的命令主要有git fetch/pull/push这个三个，有人经常使用的可能就只有get pull，git push这样的，可能第一次看到`git push orgin bugFix:master`这样的命令很惊奇，所以这里对这几个命令的语法做一些简介，如果有了解过的就可以不用看下面的文章了。

### **git push语法**

```text
git push <remote> <localPlace:remotePlace>
```

例子：

```text
git push origin master:master
```

表示切换到本地的master分支，获取所有提交，再到远程仓库"origin"中找到"master"分支(如果没有会新建一个)，将远程仓库中没有的提交记录添加上去。 通过"localPlace"参数来告诉git提交记录来自master，在推送到远程仓库中的master，后面的两个参数实际上是要同步的两个仓库的位置。 当只指定localPlace的时候remotePlace的值默认是我们跟踪的分支名称(需要注意push.default参数的值)，如果当前分支就是你想要提交的分支，那么你可以直接写成`git push`

这里的localPlace和remotePlace按照官方说明是一个refspec，“refspec” 是一个自造的词，意思是 git 能识别的位置（比如分支 bugFix或者 HEAD~1）。

### **git fetch**

git fetch和git push的参数及其类似，它们概念相同，只是方向相反(因为你现在是下载，而非上传)

```text
git fetch <remote> <remotePlace:localPlace>
```

举个例子

```text
git fetch origin master
```

git会到远程仓库的master分支上，然后获取所有本地不存在的提交，放到本地的origin/master上，注意fetch并不会更新本地的非远程分支，而是下载提交记录。

如果想要直接更新本地master分支也不是不可以，运行下面的命令即可

```text
git fetch origin master:master
```

**理论上是可以的，但是强烈建议不要那么做。**

当我们只执行`git fetch`不带任何参数的时候，它就会下载所有的提交记录到各个远程分支。

### **git pull**

学会了git fetch，那么git pull就很简单了，git pull唯一关注的是提交最终合并到哪里。之前说过

```text
git pull == git fetch;git merge
```

那么

```text
git pull origin bugFix ====  git fetch orign master; git merge origin/bugFix
```

![img](https://pic3.zhimg.com/v2-e656fc75ebf60b3f9cd5e7eb076c6002_b.jpg)

上图可以看到当前分支是bugFix，执行pull命令之后origin/master的指向改变了，bugFix分支的内容和远端master分支的内容进行了合并，把这条命令拆解为2条来记忆我相信更容易让人理解。
