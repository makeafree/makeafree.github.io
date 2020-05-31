---
title: 修改Jupyter Notebook默认打开路径
date: 2019-12-8
tags: python文件
categories:
- python
---

# 修改Jupyter Notebook默认打开路径
当前jupyter notebook 的启动工作路径就是在  c:\Users\用户名，为了使每次都在一个指定的工作路径下打开，可按如下设置：
- 1、打开命令行界面,使用win+R然后输入cmd回车后，打开命令行界面， 然后输入jupyter notebook --gernerate-config，然后回车，如下图所示：
![生成config命令](https://img-blog.csdnimg.cn/20191208144552407.png)
- 2、生成之后在C:\Users\计算机名字\.jupyter\jupyter_notebook_config.py,使用编辑器(notepad++或者sublime text)打开,找到
#c.NotebookApp.notebook_dir = ''
在引号中添加你定义的路径，需要注意的是要用‘\\’或者‘/’，最后单引号前面加上r(后面学习Python过程中会学到r的意义)，自定义之后，将最前面的#号去掉，设置好后如下图所示：![路径设置示意图](https://img-blog.csdnimg.cn/20191208145207371.png)
- 3、保存文档后，再次在命令行输入jupyter notebook指令即可在指定的文件夹下启动。

注意：启动方式有两种：
第一：命令行输入jupyter notebook
第二：window应用程序中找到anaconda文件夹下的jupyter notebook.exe打开