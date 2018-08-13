---
title: hexo 帮助
date: 2018-08-13 9:30:58
tags: hexo
---
- [指令](#%E6%8C%87%E4%BB%A4)
    - [init 新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站。](#init-%E6%96%B0%E5%BB%BA%E4%B8%80%E4%B8%AA%E7%BD%91%E7%AB%99%E3%80%82%E5%A6%82%E6%9E%9C%E6%B2%A1%E6%9C%89%E8%AE%BE%E7%BD%AE-folder-%EF%BC%8Chexo-%E9%BB%98%E8%AE%A4%E5%9C%A8%E7%9B%AE%E5%89%8D%E7%9A%84%E6%96%87%E4%BB%B6%E5%A4%B9%E5%BB%BA%E7%AB%8B%E7%BD%91%E7%AB%99%E3%80%82)
    - [new 新建一篇文章。](#new-%E6%96%B0%E5%BB%BA%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E3%80%82)
    - [发布到github pagers](#%E5%8F%91%E5%B8%83%E5%88%B0github-pagers)
    - [generate 生成静态文件](#generate-%E7%94%9F%E6%88%90%E9%9D%99%E6%80%81%E6%96%87%E4%BB%B6)
    - [publish 发表草稿。](#publish-%E5%8F%91%E8%A1%A8%E8%8D%89%E7%A8%BF%E3%80%82)
    - [server 启动服务器。默认情况下，访问网址为：http://localhost:4000/。](#server-%E5%90%AF%E5%8A%A8%E6%9C%8D%E5%8A%A1%E5%99%A8%E3%80%82%E9%BB%98%E8%AE%A4%E6%83%85%E5%86%B5%E4%B8%8B%EF%BC%8C%E8%AE%BF%E9%97%AE%E7%BD%91%E5%9D%80%E4%B8%BA%EF%BC%9Ahttplocalhost4000%E3%80%82)
    - [deploy 部署网站。](#deploy-%E9%83%A8%E7%BD%B2%E7%BD%91%E7%AB%99%E3%80%82)
    - [migrate 从其他博客系统](#migrate-%E4%BB%8E%E5%85%B6%E4%BB%96%E5%8D%9A%E5%AE%A2%E7%B3%BB%E7%BB%9F)
    - [clean 清除缓存文件 (db.json) 和已生成的静态文件 (public)。](#clean-%E6%B8%85%E9%99%A4%E7%BC%93%E5%AD%98%E6%96%87%E4%BB%B6-dbjson-%E5%92%8C%E5%B7%B2%E7%94%9F%E6%88%90%E7%9A%84%E9%9D%99%E6%80%81%E6%96%87%E4%BB%B6-public%E3%80%82)
    - [安全模式 在安全模式下，不会载入插件和脚本。](#%E5%AE%89%E5%85%A8%E6%A8%A1%E5%BC%8F-%E5%9C%A8%E5%AE%89%E5%85%A8%E6%A8%A1%E5%BC%8F%E4%B8%8B%EF%BC%8C%E4%B8%8D%E4%BC%9A%E8%BD%BD%E5%85%A5%E6%8F%92%E4%BB%B6%E5%92%8C%E8%84%9A%E6%9C%AC%E3%80%82)
    - [调试模式 在终端中显示调试信息并记录到 debug.log。](#%E8%B0%83%E8%AF%95%E6%A8%A1%E5%BC%8F-%E5%9C%A8%E7%BB%88%E7%AB%AF%E4%B8%AD%E6%98%BE%E7%A4%BA%E8%B0%83%E8%AF%95%E4%BF%A1%E6%81%AF%E5%B9%B6%E8%AE%B0%E5%BD%95%E5%88%B0-debuglog%E3%80%82)
- [问题以及解决方法](#%E9%97%AE%E9%A2%98%E4%BB%A5%E5%8F%8A%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95)

## 指令

### init 新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站。
`$ hexo init [folder]`

### new 新建一篇文章。
`$ hexo new [layout] <title>`

### 发布到github pagers
```
hexo clean
hexo g -d
https://beilo.github.io/
```

### generate 生成静态文件 
`$ hexo generate` 或者 `$ hexo g`
| 选项            | 描述                   |
| --------------- | ---------------------- |
| `-d`,`--deploy` | 文件生成后立即部署网站 |
| `-w`,`--watch`  | 监视文件变动           |

### publish 发表草稿。
`$ hexo publish [layout] <filename>`

### server 启动服务器。默认情况下，访问网址为：http://localhost:4000/。
`$ hexo server`
| 选项               | 描述                     |
| ------------------ | ------------------------ |
| `-g`, `--generate` | 部署之前预先生成静态文件 |

### deploy 部署网站。
`$ hexo deploy` 或者 `$ hexo d`
| 选项             | 描述                           |
| ---------------- | ------------------------------ |
| `-p`, `--port`   | 重设端口                       |
| `-s`, `--static` | 只使用静态文件                 |
| `-l`, `--log`    | 启动日记记录，使用覆盖记录格式 |

### migrate 从其他博客系统
`$ hexo migrate <type>`

### clean 清除缓存文件 (db.json) 和已生成的静态文件 (public)。
`$ hexo clean`

### 安全模式 在安全模式下，不会载入插件和脚本。
`$ hexo --safe`

### 调试模式 在终端中显示调试信息并记录到 debug.log。
`$ hexo --debug`

## 问题以及解决方法
```
1. 
JS-YAML: can not read a block mapping entry; a multiline key may not be an implicit key at line 8, column 12:
原因是最上面的 title:等等的冒号后面格式要正常,比如 所有:后面加个空格。

2.
could not read Username for 'https://github.com': Invalid argument
关联 github 失败,后从https -> ssh.

3.
npm install hexo-deployer-git --save
git 报错
```
