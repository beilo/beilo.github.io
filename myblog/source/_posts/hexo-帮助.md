---
title: hexo 帮助
date: 2018-08-13 9:30:58
tags: hexo
toc: true
---

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

4.
开启toc(目录)功能https://pengloo53.gitbooks.io/hexo/content/chapter2/7%20%E6%B7%BB%E5%8A%A0%E6%96%87%E7%AB%A0%E7%9B%AE%E5%BD%95.html
文件目录: 
1. myblog\themes\landscape\layout\_partial
2. myblog\themes\landscape\source\css\_partial

默认展开所有目录
https://github.com/iissnan/hexo-theme-next/issues/531
文件目录: 
1. myblog\themes\next\source\css\_custom
```
