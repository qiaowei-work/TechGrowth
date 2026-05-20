# GIt

常见情形 & 操作过程 & 原理



## 情形一：本地仓库  ==》 空远程仓库

### 基本原则

1. 绑定远程分支的前提是本地创建分支 =》**空文件夹是不会创建分支的**

新建`文件夹`，创建 & 复制文件到该`文件夹`

```shell
git add . 
git commit -m "提交的备注"
```

绑定远程分支

```SHELL
git remote add origin 'git url'  # 绑定远程分支
git remote -v # 查看远程分支
```

推送并绑定远程仓库

```SHELL
git push -u origin main  # 绑定到远程仓库 'main' 
```

📢注意：

1. 这个命令中，`-u ` 表示绑定，`origin` 是远程仓库别名，`main` 表示绑定到远程仓库的该分支
2. 如果远程仓库没有这个分支，会创建一个名为 `main` 的分支

## 情形二：本地空文件夹 =》拉取非空项目

初始化仓库
```shell
git init
```
绑定远程分支
```shell
git remote add origin [git地址]
```
查看远程分支
```shell
git fetch origin
```
切换&创建&绑定远程分支
```shell
git checkout -b [远程分支名称] origin/[远程分支名称]
```