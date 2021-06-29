---
title: 使用Hexo搭建博客
date: 2019-05-24 16:11:31
categories: Hexo
tags: Hexo
---

捣鼓了一个下午，终于使用hexo搭完了自己的博客，有必要记录一下。

## 一、创建GitHub仓库

仓库的名称有格式要求。比如我的github仓库名是“Work4CP”，那么创建的仓库名就是Work4CP.github.io。
![](http://ww1.sinaimg.cn/large/005Mjp9lly1g3dgzbpcxaj311j0gj75b.jpg)

## 二、下载node.js，安装git

## 三、安装hexo

官方中文文档<https://hexo.io/zh-cn/docs/index.html>

### hexo常用命令
```bash
hexo s #完整命令为hexo server，用于启动服务器，主要用来本地预览
hexo g #完整命令为hexo generate，用于生成静态文件
hexo d #完整命令为hexo deploy，用于将本地文件发布到github上
hexo new "博客的名字" #用于新建一篇文章
```

1. 新建文件夹 D:\MyBlogs，进入该文件夹，右键选择 Git Bash here (提前安装好git bash工具)

2. 运行命令
   ```bash
   npm install -g hexo-cli
   ```
   
3. 执行以下命令，初始化hexo，这时候会在该文件夹中创建网站所需要的文件
   ```bash
   hexo init
   ```
   
4. 安装依赖包
   ```bash
   npm install
   ```
   
5. 生成静态文件
   ```bash
   hexo g
   ```
   
6. 此时最基本的网站已经搭建好了，只不过是在本地放着，我们可以先预览一下
   ```bash
   hexo s
   ```
   这时，会在本地开启一个http服务，监听4000端口，可以在浏览器输入http://localhost:4000访问

## 四、部署本地文件到github

1. 配置Git的用户信息
   **git config --global user.name "GitHub用户名"**（git config --global user.name "Work4CP"）

   **git config --global user.email "GitHub邮箱"**（git config --global user.email "daiqianjieaixia@126.com"）

2. 生成ssh密钥文件
   **ssh-keygen -t rsa -C "Github邮箱”**（ssh-keygen -t rsa -C "daiqianjieaixia@126.com"）
   这一步会让你设置私钥密码。

3. 把本机生成的公钥添加到github中
   将刚新生成的公钥 **id_rsa.pub** 添加到Github中
   ![](http://ww1.sinaimg.cn/large/005Mjp9lly1g3dh0c60bvj311q0hn75w.jpg)

4. 使用私人密钥与GitHub进行认证和通信
   ![](http://ww1.sinaimg.cn/large/005Mjp9lly1g3dh0nr4gzj30ic0axjrf.jpg)

5. 将hexo和github进行关联
   在D:\MyBlogs\config.yml最下方，添加如下配置：
   
   ```yml
   deploy:
    type: git
    repository: git@github.com:Work4CP/Work4CP.github.io
    branch: master
   ```
   
6. 将本地文件同步到GitHub
   ```bash
   hexo g
   hexo d
   ```
   
7. 访问<https://work4cp.github.io/>可以看到自己的博客了。

## 五、发表一篇文章

1. 在Git Bash中执行命令：
   ```bash
   hexo new "使用Hexo搭建博客"
   ```
   
2. 这时会在*D:\MyBlogs\source\_posts*中，生成一个***使用Hexo搭建博客.md***文件，我们就在里面编写自己的内容。

3. 写完后，可以现在本地预览一下生成的博客：
   ```bash
   hexo g
   hexo s
   ```
   
   访问http://localhost:4000
   
4. 同步到GitHub上
   ```bash
   hexo d
   ```
   
5. 访问自己的网站即可查看<https://work4cp.github.io/>

关于博客中主题的更改和其它的设置，可以查看中文官方文档：<https://hexo.io/zh-cn/docs/index.html>

hexo主题库：<https://hexo.io/themes/>

知乎：有哪些好看的hexo主题：<https://www.zhihu.com/question/24422335>



## 六、关于博客图片显示的问题

开始图片显示不了，后来网上说用插件的方法，还是不行。最后用了新浪微博的图床，只要在Chrome浏览器上添加**新浪微博图床**插件，然后再把图片添加上去即可，它会自动生成图片的地址：
![](http://ww1.sinaimg.cn/large/005Mjp9lly1g3dh5axjfvj30lt0f1q3w.jpg)

把这个Markdown地址添加到博客中就行了。

## 七、关于github改用户名之后博客无法访问的问题

今天手贱把用户名给改了，原来的博客域名无法访问了。解决方法是新建一个仓库，仓库名命名为`新的用户名.github.io`。再用`git`把博客 push 到仓库。新的博客域名就是`http://新的用户名.github.io`。

注意一定要新的仓库名的第一个字段和你的用户名一致，比如我的是**CaryDai.github.io**，我改的用户名就是CaryDai。在看到[这个提问](https://segmentfault.com/q/1010000010264567)的第一个解答后发现的问题。

## 参考文档

1. “Windows下，Hexo+GitHub搭建博客”<https://blog.csdn.net/qq_27754983/article/details/76143478>
2. “win10 如何搭建hexo博客”<https://blog.csdn.net/xiaoritai7803/article/details/79325017>
3. “windows使用hexo搭建博客”<https://jingyan.baidu.com/article/e8cdb32b0ce12137042bad51.html>

