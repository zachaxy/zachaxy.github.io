---
title: hexo搭建改良
date: 2017-12-05 19:20:39
tags: 工具
---

hexo 搭建改良
之前使用[hexo 搭建博客过程全纪录](https://zachaxy.github.io/2017/03/30/hexo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E8%BF%87%E7%A8%8B%E5%85%A8%E7%BA%AA%E5%BD%95/)已经可以成功的搭建一个 hexo 的博客了，使用了一段时间之后，发现了一个问题，就是博客只能在当前的这台 pc 上发布，但是有时候想在家也写写博客并立即发布，之前的博客只能基于单终端的发布。因此在多个端上进行 hexo 的同步成为当前需要解决的问题，为此，我这里进行了三次改进。通过不断的改良，也使我对 hexo 有了进一步的了解。

<!-- more -->

# GitPage + Hexo
相信大家刚开始用 hexo 搭建博客的时，搜索出现最多的关键词就是 GitPage + Hexo 了
这里用大白话说一下二者的基本原理：

不知道大家想过这个问题没，就是如果我们想做一个博客，还想让外网的其他人也看到，需要哪些准备条件：

1. 首先，我们肯定需要一个服务器，来保存我们的博客。这样外网才能访问的到。
2. 其次，我们在浏览器浏览的博客都是 html 的网页，因此我们的博客页面在服务器上必须保存的是 html，css，javascript 等着些文件。

接下来依次解决上面的问题：

问题一：有多种解决方法，可以购买阿里云，腾讯云等，来构建自己的博客系统，但是需要付费。那就寻找一个免费的平台，好在 github 提供了类似的功能，我们的代码仓库放在 github 上，外网都可以看到，这不就是一个免费的服务器嘛。但是如果我们直接把博客的 html 代码放到代码仓库，那么 github 会认为你创建的是一个 html 的代码库，这样查看例如 index.html 的文件并不是一个网页，而是代码。因此还需要一个转换器，该转换器的工作是如果我们打开的是例如 index.html 这样的文件，github 不是展示其代码，而是将其渲染成对应的网页展示出来。这一使命由 GitPage 来完成。

其实要完成上述的需求，还必须遵从 GitPage 规定的两个要求：
- 每个人至多有一个这样的代码库，其仓库名必须是 `username.github.io`。这里的 username 就是你 github 的用户名。这里只是第一步，此时这任然是一个普通的代码库
- 该代码库中的文件必须按照一定的要求保存。例如在指定的文件夹中放 html 文件，在指定的文件夹放 css 文件等。

只要我们按照 GitPage 的要求，将你写好的博客的 html 等文件放到指定的文件夹中。其实你就可以创建博客了。

问题二：解决了服务器的存储问题，接下来解决博客的网页生成问题了，并不是每个人都会前端的（即使会，我相信他们也不愿意真的用 html 来写博客，效率太低！），如果有这么一个转换器，可以将我写的文档转换成网页，那将极大的提高我们的工作效率。还真有这么一个转换器——Hexo。它可以将 markdown 的文件转换为 html 文件。并且配有多种主题，只需简单的配置，就可以轻松的切换风格不同的网页。而且其转换的文件夹的结构是完全符合 GitPage 的要求的。其对应于 `/public`文件夹。该文件夹下的所有文件只需要全部上传到`username.github.io`仓库中，我们就可以通过 `https://username.github.io/`来访问了。


这里强调一点：我们所写的博客，**最重要的就是原始的 markdown 文件！** 而不是其对应的 html 文件。因为这些文件随时可以用 hexo 来再次生成。但是如果 markdown 文件丢失了，可能意味着我们要重写写文章了。



# 快速搭建
参考之前写的这篇[hexo 搭建博客过程全纪录](https://zachaxy.github.io/2017/03/30/hexo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E8%BF%87%E7%A8%8B%E5%85%A8%E7%BA%AA%E5%BD%95/)就可以顺利跑通了。可以通过它进行一次练手。其主要是针对单平台的发布。如果你就是在单一的 pc 上写博客并发布，那它很适合你。但是如果你的需求是在 pcA 上写两篇，又在 pcB 上写两篇。那么请往下看。



# 探索改进之旅

这里的改进主要是解决多终端同步问题。

## 最原始最简单的方法
在了解了上面 GitPage 和 Hexo 所做的工作之后，我萌生了一个最简单的方法：在另一台电脑 pcB 上安装 Node.js。然后直接将 pcA 上的执行`hexo init`的文件夹(例如我的是：`E:/blogs`)拷贝到另一台 pcB 上，这样就可以直接用了，可以说是无缝连接。

缺点：如果你两台电脑交互发布文章的太频繁，老是这样将文件夹拷贝来拷贝去，也很麻烦。

适用情况：如果你之前长期 pcA 上发布文章，后来换了台电脑，以后准备长期在 pcB 上发布文章，那么这种方法最适合不过了。


## 两个仓库
前面也说过博客中最重要的是 markdown 的源文件。而在`username.github.io`的代码库中保存的是博客文件夹的 public 下的内容。之前也想过用两个分支来解决。但是[hexo 搭建博客过程全纪录](https://zachaxy.github.io/2017/03/30/hexo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E8%BF%87%E7%A8%8B%E5%85%A8%E7%BA%AA%E5%BD%95/)的做法，二者并不在同一目录级别。无法实现用一个仓库再将博客文件夹下的 source 文件进行上传。故采用了两个仓库的做法。即新建一个仓库`MyBlogBackup`用来保存 markdown 源文件。

注意：这种解决方案是针对使用[hexo 搭建博客过程全纪录](https://zachaxy.github.io/2017/03/30/hexo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E8%BF%87%E7%A8%8B%E5%85%A8%E7%BA%AA%E5%BD%95/)搭建博客为前提的。其它情况则不适用。这里贴出所写的脚本文件：

发布文章的脚本：
```shell
# this time,we are in blogs dir,we need sync the source file
git pull
# generate the new file and the old files from another machine
hexo generate
cd .deploy/zachaxy.github.io
git pull
# copy web pages from public/
cp -R ../../public/* .
git add .
git commit -m "update"
git push origin master
cd ../..
# this time,you are in blogs dir
git add .
git commit -m "upload source file"
git push origin master
```

删除文章的脚本：运行该脚本，必须传入一个参数，指定要删除的文章的名称（要带 md 后缀）
```shell
if [ $# -ne 1 ]
then
	echo "the args count must equal 1,the arg we need is the name of the file you want to delete\n";
elif [ ! -d "source/_posts/$1" ]
then
	echo "No such file or directory"
	exit 2;
else
#first rm the file in both file system and local git
	rm "source/_posts/$1"
	git add .
	git commit -m "delete file"
	git push origin master
#second cover the change to GitPage
	hexo generate
	cp -R public/* .deploy/zachaxy.github.io
	cd .deploy/zachaxy.github.io
	git add .
	git commit -m  "update"
	git push origin master
fi
```

再次回顾下用到的两个仓库：`/MyBlog/.deploy/username.github.io`以及`MyBlogBackup`

简单说一下思路：在新的终端，先用`hexo n`创建并写文章，然后执行发布脚本，该脚本会先将`MyBlogBackup`中的 markdown 同步到本地，然后用`hexo g`生成 html 文件。接下来再进入`.deploy`下的`username.github.io`的仓库,将代码 pull 下来，然后将刚刚生成的 public 下的 html 文件将仓库的文件强行覆盖。
按说相同的 markdown 文件用相同的主题生成的 html 文件是一样的，但是为了避免平台差异，hexo 版本的差异，这里还是要进行覆盖吧。


缺点：两个仓库的隔离性很高，而且目前的脚本很难称得上自动化，每次在另一个终端写博客时，都要谨记脚本的执行顺序，这些都需要人工记忆，而无法用代码约束。同时删除文件的维护成本也很高。但是作为一次尝试，它也解决了多终端同步的问题。

## 一个仓库，两个分支
这是我目前使用的解决方案。下面进行详细说明。

刚开始自己是按照[hexo 搭建博客过程全纪录](https://zachaxy.github.io/2017/03/30/hexo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E8%BF%87%E7%A8%8B%E5%85%A8%E7%BA%AA%E5%BD%95/)来搭建的，也是参考的别人的文章。一切顺利，但是和网上其它的教程有点不一样。不一样的地方在于发布文章的时候，这篇文章的做法并没有使用`hexo d`,而在博客文件夹下再创建一个 .deploy 的文件夹(当然文件夹的名字随意)，并创建一个 git 仓库，然后把用 hexo 生成的 html 文件(在 public 文件夹下)全部拷贝到.deploy 下，然后将直接把.deploy 下的代码全部 push。

刚开始我以为这种操作才是正统，后来看文章，发现更多的人用的是 `hexo d` 。其实这里并没有什么正统不正统，只是经历过这个事情之后，我们可以认定，`hexo d`干的事情就是把 public 中的文件 push 到远程。
在使用两个仓库的方法中，我也想过用两个分支的思路，但是不在同一个目录中，似乎不能作为两个分支。

```
Blogs
	public
		需要向 username.github.io push 的所有文件
	source
		_post
			*.md
```

这种目录结构创建两个分支，一个分支保存 _post 下的 md 文件，一个分支保存 public 下的文件。那么在初始化 git 仓库时在那个目录下？
这里肯定是要找二者的公共目录 Blogs，但是如果我们直接在 Blogs 下进行 git init，那么在上传 public 下的文件时是不符合 GitPage 的要求的，因为 GitPage 要求 public 下的文件必须作为仓库的一级目录。而我们这样做的话，一级目录就是 public 了。


后来经过试验，发现了使用多分支进行 hexo 同步的方法。
两个分支：一个分支我们可以叫 hexo，该分支主要是保存 markdown 源文件，hexo 配置文件。该分支需要我们手动创建，并把该分支设为默认分支。具体步骤如下：
1. 在 github 创建`username.github.io`的仓库
2. 在本地创建一个同名的文件夹`username.github.io`
3. 在本地文件夹下执行 `git init`,`git checkout -b hexo`；初始化代码库，这里目前是空的，然后创建名为 hexo 的分支，其实这里并不是创建分支，而是把默认的 master 分支改名为 hexo。不信你这时候执行`git branch`，并不显示任何分支。但是 git 的命令行已经显示当前分支为 hexo 分支了。
4. 在本地文件夹下执行`npm install hexo`、`hexo init`、`npm install` 和 `npm install hexo-deployer-git`
5. 因为我使用的主题里有一个搜索功能，所以还要额外执行`npm install hexo-generator-search -S`
6. 接下来用 hexo n xxx`创建并写文章。
7. 写好文章后先不着急发布，先添加以下远程代码库 `git remote add origin git@github.com:zachaxy/zachaxy.github.io.git`
8. 将 markdown 源文件发布上去，具体的 ignore 文件如下。注意发布的远程仓库名 `git push -u origin hexo`。其实 master 只是默认的主分支，这个名字完全可以修改，而且在刚创建的仓库第一次这样提交时，指明远程的仓库叫 hexo，那么其默认分支就是 hexo
9. 接下来生成并发布文章，使用`hexo g -d`，其会按照 _config.yml 中关于 deploy 的配置，自动创建 master 分支，并将 public 下的文件作为一级目录 push 到 master 分支中。

> git仓库的创建是不允许嵌套的，也就是在父目录下创建git，然后又在子目录下创建git仓库。如果子目录下也有一个仓库，对子目录是没有影响的，子目录只会维护自己目录下的文件修改，但是回到父目录，执行 `git add .` 的时候，就会提示当前路径下，包含一个子目录的仓库，无法执行。 那么换一个思路，我们将子目录添加在父目录的 gitignore 文件中，那么就有可以实现同一个仓库，两个分支，其中一个分支记录子目录下的文件变更，而父目录忽略该子目录，记录其他文件即可。猜想 hexo 也是这样做的，在执行 `hexo -d` 的时候，也是将 `hexo g` 生成的 public 下的文件，拷贝到 `.deploy_git` 文件夹下,同时父目录下又忽略该文件夹，即可实现上述操作。


ignore 文件
```
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
```

 _config.yml 中关于 deploy 的配置，注意这里的 branch 必须为 master。
```
deploy:
  type: git
  repo: git@github.com:zachaxy/zachaxy.github.io.git
  branch: master
```



以后我们在另一个终端进行发布时,同样创建一个`username.github.io`文件夹，然后依次执行
1. 安装 node.js
1. git init
1. git checkout -b hexo
1. git remote add origin git@github.com:zachaxy/zachaxy.github.io.git
1. npm install -g hexo
1. git pull origin hexo
1. npm install




要注意的地方:

这里并没有像第一次搭建的时候执行 hexo init, 这是反而是用 git pull origin hexo 来替代的,因为这个hexo init 的目的是为了在当前目录下建立一些目录结构,而这些结构我们已经上传到了 github 上.

既然目录结果一样,为什么一定要选远程仓库的,而不是用新的 hexo init, 这是因为我们的远程仓库上已经有了其它的主题配置,这个一会要用,还有一个重要的文件 `package.json`,这个文件里包含了我们创建博客时所需的额外的 node 插件,还记得我们在初始搭建环境的时候,安装的 `npm install hexo-deployer-git`和`npm install hexo-generator-search -S`吗?



这些插件的版本,都已经写在了`package.json`中,接下来我们要执行的 `npm install`命令就是安装`package.json`中所指定的插件,这样省的我们在之前另一台电脑上安装了很多插件的情况下,在新电脑上一个一个的再去重新安装.



同时在执行 npm install hexo 的时候,会生成一个多余的`package-lock.json`的文件,记得把这个文件添加到`.gitignore`文件中.

接下来正常的创建 md，写博客，发布即可。