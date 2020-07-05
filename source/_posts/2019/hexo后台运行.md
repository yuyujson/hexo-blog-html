---
title: hexo后台运行
permalink: hexo-hou-tai-yun-xing
date: 2019-12-03 14:16:17
tags: hexo
categories: util
---
> 前言: hexo部署在服务器后, 一旦断开ssh链接就会自动关闭. 故在网上找了个接管hexo进程的工具

## 安装pm2
```
npm  install -g pm2
// 下方为打印出的日志信息
/self/app/node-v12.13.1-linux-x64/bin/pm2 -> /self/app/node-v12.13.1-linux-x64/lib/node_modules/pm2/bin/pm2
/self/app/node-v12.13.1-linux-x64/bin/pm2-dev -> /self/app/node-v12.13.1-linux-x64/lib/node_modules/pm2/bin/pm2-dev
/self/app/node-v12.13.1-linux-x64/bin/pm2-docker -> /self/app/node-v12.13.1-linux-x64/lib/node_modules/pm2/bin/pm2-docker
/self/app/node-v12.13.1-linux-x64/bin/pm2-runtime -> /self/app/node-v12.13.1-linux-x64/lib/node_modules/pm2/bin/pm2-runtime
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@2.1.2 (node_modules/pm2/node_modules/fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@2.1.2: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})

+ pm2@4.2.0
added 206 packages from 202 contributors in 12.702s
```

<!--more-->

完成后使用```pm2 -v```进行验证, 如果提示```pm2: 未找到命令```则进行下面操作
```
ln -s /self/app/node-v12.13.1-linux-x64/bin/pm2 /usr/bin/pm2
说明:
ln -s pm2安装路径(上方安装时打印的日志信息中有) /usr/bin/pm2
```
完成后再次使用```pm2 -v```进行验证
## 项目文件
项目根目录增加文件```hexo_run.js``` 内容如下
```
//run
const { exec } = require('child_process')
exec('hexo server',(error, stdout, stderr) => {
    if(error){
        console.log('exec error: ${error}')
        return
    }
    console.log('stdout: ${stdout}');
    console.log('stderr: ${stderr}');
})
```
## 启动
```
pm2 start hexo_run.js
```
## 停止
```
pm2 stop hexo_run.js
```
> 如果怕忘记命令可以创建一个```run.sh```文件吧这串命令放进去,增加权限. 下次直接执行这个文件就行