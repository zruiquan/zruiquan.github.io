title: Hexo 多环境部署
tags:
  - 博客
categories:
  - Hexo
date: 2021-08-18 15:35:00
---



## 1、安装git

```
sudo apt-get install git
```

## 2、设置git全局邮箱和用户名

```
git config --global user.name "yourgithubname"
git config --global user.email "yourgithubemail"
```

## 3、设置ssh key

```
ssh-keygen -t rsa -C "youremail"
```

## 4、安装nodejs

```
sudo apt-get install nodejs
sudo apt-get install npm
```

## 5、安装hexo

```
sudo npm install hexo-cli -g
```

## 6、克隆hexo源文件分支

```
git clone git@………………
cd xxx.github.io
npm install
npm install hexo-deployer-git --save
```

## 7、生成，部署：

```
hexo g
hexo d
```

## 8、新建博文

```
hexo new newpage
```

## Tips:

```
不要忘了，每次写完最好都把源文件上传一下
git add .
git commit –m "xxxx"
git push 
```

如果是在已经编辑过的电脑上，已经有clone文件夹了，那么，每次只要和远端同步一下就行了

```
git pull
```

## 参考博文

[]: https://blog.csdn.net/sinat_37781304/article/details/82729029(https://blog.csdn.net/sinat_37781304/article/details/82729029)

