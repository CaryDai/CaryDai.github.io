---
title: idea常用技巧
date: 2019-05-26 17:30:58
categories: IDE
tags: 
	- IDE常用技巧
	- idea
---

## idea 中复制一个 Maven 工程

1. 将原工程整个文件夹复制，重新命名，**并删除掉里面的 .idea 文件夹**；
2. 用 idea 打开刚才重命名的文件夹，在 Project Structure ——> Project Settings ——> Project 和 Modules 里重新选择 Project SDK。
3. Artifacts 里面：
   ![](http://ww1.sinaimg.cn/large/005Mjp9lly1g3euu3su7mj30sj0kdmxw.jpg)
   选择刚刚打开的项目，再点 Apply。
4. 配置 Tomcat
   Run ——> Edit Configurations，点左侧绿色的 + 号，选择本地 tomcat，并自己命名。
   选择 Deployment，点 + 号，选择“Artifact”，再点 Apply，OK。启动项目即可打开网页。

## idea中复制一个普通工程

https://www.cnblogs.com/kaiwen/p/10232908.html

在上面的基础上，还需要在`Project Structure`---`Project Settings`---`Project`里面制定输出的文件夹路径，如：`G:\JavaWorkspace\RPC\RPC_V2\out`。

## 导入项目后右键无法运行Main方法

https://blog.csdn.net/yyj108317/article/details/89182332

## 解决Tomcat控制台乱码问题

在`Run/Edit Configurations...`里面输入下图红框中的内容：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/%E8%A7%A3%E5%86%B3%E6%8E%A7%E5%88%B6%E5%8F%B0%E4%B9%B1%E7%A0%81%E9%97%AE%E9%A2%98.png)

## idea 运行单个程序

有的时候只想运行某份代码，不想编译其它的代码，这是需要设置一下：

Run - Edit Configuration - Before launch 里面，把 **Build** 换成**Build, no error check**，Apply 之后就行了。当然，前提是这个文件中 main 函数所依赖的所有 class 文件都没有错误。

## 解除Git关联

由[这篇博客](https://blog.csdn.net/Alex_81D/article/details/80392935)可知：

file ->settings->version control 选中这一栏,右边有个

[![img](https://gss0.baidu.com/7Po3dSag_xI4khGko9WTAnF6hhy/zhidao/wh%3D600%2C800/sign=33f9d77f55df8db1bc7b74623913f16c/d439b6003af33a8786c9d6fdcd5c10385243b5b5.jpg)](https://gss0.baidu.com/7Po3dSag_xI4khGko9WTAnF6hhy/zhidao/pic/item/d439b6003af33a8786c9d6fdcd5c10385243b5b5.jpg)

点红色减号,就解除了,然后去项目目录下删除.git这个文件夹,你可以不删除,为了以后继续关联。

提交到远程仓库的时候，需要在 Settings->Version Control 里检查是否是“Unregistered roots”，若是的话，就需要选中该路径，如`G:\JavaWorkspace\RPC\RPC_V4`，点击左上角的“+”号即可，然后Apply。