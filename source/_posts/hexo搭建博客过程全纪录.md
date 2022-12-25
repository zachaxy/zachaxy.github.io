---
title: hexo搭建博客过程全纪录
date: 2017-03-30 17:30:53
tags: 工具
mathjax: true
---

# 准备工作

1. 安装[Node.js](https://nodejs.org/en/download/)一路next即可

2. 安装[git](http://git-scm.com/download/),一路next即可

3. 注册github账号,这个步骤就不说了吧...

4. 生成ssh key(这一步非必须)

   1. 检测之前是否有生成过ssh没

      ```
      cd  ~/.ssh   //注意 ~/.ssh之间没有空格
      ```

      如果提示：`No such file or directory` 说明你还未生成`ssh key`

   2. 生成新的ssh key

      ```
      $ ssh-keygen -t rsa -C "邮件地址@youremail.com"  //这个邮箱地址就是你注册github使用的邮箱
      Generating public/private rsa key pair.
      Enter file in which to save the key (/c/Users/xxx/.ssh/id_rsa):
      ```

      第三行是在询问你将生成的ssh key放在哪里默认是你的用户目录,这里直接回车就好

   3. 接下来或让你创建一个密码,并再次确认

      ```
      Created directory '/c/Users/zhangxin/.ssh'.
      Enter passphrase (empty for no passphrase):
      Enter same passphrase again:
      ```

   4. 添加 ssh key 到 github

      打开本地`/c/Users/zhangxin/.ssh`,你的肯定不是我这个文件,改成你在是第二步保存的文件位置,将`id_rsa.pub`文件用记事本打开,将此文件里面内容为刚才生成人密钥。如果看不到这个文件，你需要设置显示隐藏文件。复制这个文件的内容,登陆你的github,点击右上角头像处的下拉列表 `Settings—>SSH and GPG keys —> 右上角 New SSH key`,把你本地生成的密钥复制到里面（key文本框中）， 点击` add key `就 ok 了.同时你也可以设置title用来为这个 key 做一个标示,因为我们很可能在多台电脑上都写博客并推送,不同的电脑需要按照相同的步骤,当然生成的ssh key是不同的,如果你需要同时在另一台电脑上工作,就需要把另一台电脑的ssh key 也添加到你的github中,title所以,你自己可以区分开就好,比如单位的,用一个`work`,在家的用一个`home`,随你喜欢.

   5. 测试 ssh 是否正确设置

      ```
      $ ssh -T git@github.com

      The authenticity of host 'github.com (207.97.227.239)' can't be established.
      RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
      Are you sure you want to continue connecting (yes/no)?

      //这一步输入yes

      	
      Hi cnfeat! You've successfully authenticated, but GitHub does not provide shell access.
      ```

      如果正常的话,就是显示这些内容了.

5. 设置用户信息

   ```
   $ git config --global user.name "userName"             //你的用户名,要加双引号的啊
   $ git config --global user.email  "userName@xxx.com"  //填写自己的邮箱 ,也要加双引号的啊
   ```

   查看用户设置

   ```
   $ git config --list
   ```

   ​


# 建立博客

1. 登陆 github ,创建一个新的仓库,名字叫做 `xxx.github.io`,这里xxx要换成你的 github 的用户名,点击「Create Repository」 完成创建。

2. 创建一个文件夹来保存你写的博客,例如在 E 盘下创建文件夹 `blogs`

3. 进入该文件夹,鼠标右键,打开`git bash`

4. 安装`hexo`,在bash中输入`npm install -g hexo`

5. `bash`中进入`bolgs`文件夹下,`cd E:/blogs`

6. 输入`hexo init`

7. 现在已经搭建起来一个本地博客了 , 输入以下命令验证

   `$ hexo g` -生成

   `$ hexo s` -启动服务本地预览 

   然后到浏览器输入localhost:4000进行预览(ctrl + c 停止本地预览)

   ​

   ​

# 更换主题

   目前使用的是`hexo`默认的主题,其实也很好看的,如果你不喜欢,可以更换主题,这里推荐`jacman`,

1. 下载主题

   将主题下载到`blogs/theme`目录下在`bash`中执行` git clone https://github.com/wuchong/jacman.git E:/blogs/themes/jacman`

2. 更换主题

   修改`blogs`目录下的config.yml配置文件中的theme属性，将其设置为jacman

3. 启用主题

   ```
   hexo clean --因为主题换了 你需要clean以下老主题生成的缓存
   cd themes/jacman
   git pull
   hexo g   --生成
   hexo s   --启动本地预览
   ```

   ​



# 上传博客

经过上面的步骤之后,就可以开始写博客并上传到github上了,步骤如下:

1. 进入到`blogs`文件加下,运行`hexo n "博客文件名"`
2. 找到`blogs/source/_posts/xxx`其中xxx是第一步新键的博客文件名,默认为`.md文件`
3. 打开该文件,书写博客,保存
4. `执行 ./ok.sh`,中间可能会遇到让你输入用户名和密码的情况,输入即可.（关于ok.sh，请看下面的快捷部署）




# 快捷部署

1. 进入之前创建的 blogs 的根目录 接着操作以下命令

   `$ cd blogs`

   注意 1：现在我们需要clone我们自己的GitHub仓库了

   注意 2：切记下面是**你自己的仓库名** , 把名字都改过来 , 下面我用的是我的仓库名字

   `$ git clone https://github.com/zachaxy/zachaxy.github.io.git .deploy/zachaxy.github.io`

   翻译下这条命令的意思

   将我们之前创建的GitHub 仓库克隆到本地 , 命令会新建一个目录叫做.deploy用于存放克隆的代码。

   然后会在.deploy文件夹下生成一个 **你的名字.github.io** 的文件夹用于存放文件

2. 接着在 Hexo **根目录**下创建一个 .txt 文件 , 把下面的命令复制进去

3. 注意 ：**你的GitHub名字**是什么就**把你的名字全部改到下面** , 细心点。稍微解释一下下面的命令，在部署文章之前，我们肯定已经使用 `hexo n "xxx" `产生了一个xxx.md的文件，并书写完博客了，那么接下来这个第一行就是生成博客对应的html文件等，这些文件都放在 `hexo/public` 路径下；第二条指令是将 `public` 下的所有文件 拷贝到 本地的`.deploy/zachaxy.github.io` 路径下（相同文件会覆盖）；第三条指令，进入`.deploy/zachaxy.github.io`路径；接下来第四条到第六条指令，就是普通的提交命令，不在解释。

```
hexo generate
cp -R public/* .deploy/zachaxy.github.io
cd .deploy/zachaxy.github.io
git add .
git commit -m  "update"
git push origin master
```

4. 将这个 **.txt 文件的后缀改成 .sh** , 它就变成了脚本文件 , 我们就将文件改成 **ok.sh** 。
5. 从此以后需要部署本地博客到 GitHub , 在 hexo 根目录下，直接`./ok.sh` , 会弹出提示 , 需要输入 GitHub 的用户名跟密码 , 按提示输入自己的用户名和密码即可

# jacman配置



[最适合新手的 GitHub + Hexo 「大话」博客搭建教程](https://smartbeng.github.io/2017/03/26/blogFinish/)

[Hexo新的一些优化](http://whatbeg.com/2017/04/13/hexosomeopt.html)

 [[小技巧\]让Hexo在使用Mathjax时支持多行公式](http://kubicode.me/2016/03/18/Hexo/The-Trick-about-Hexo-Support-MutliLine-Equation-using-Mathjax/)

[修复Hexo写Mathjax公式多个下标失效的问题](http://kubicode.me/2016/03/16/Hexo/Fix-Hexo-Bug-In-Mathjax/)

[用 Hexo 搭建个人博客-02：进阶试验](http://www.jianshu.com/p/6c1196f12302)

 [Hexo部署环境迁移--多台电脑搭建Hexo环境](http://whatbeg.com/2016/09/17/hexo-migrate.html)

 [多机更新 Hexo 博客](http://lowrank.science/Hexo-Migration/)