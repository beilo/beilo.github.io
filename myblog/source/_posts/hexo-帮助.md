---
title: hexo 帮助
date: 2018-08-13 9:30:58
tags: hexo
---

## 指令

### init 新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站。
`$ hexo init [folder]`

### new 新建一篇文章。
`$ hexo new [layout] <title>`

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
```
