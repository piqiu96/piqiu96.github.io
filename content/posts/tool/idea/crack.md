---
title: "IDEA破解"
date: 2021-12-05T08:17:29+08:00
draft: false
categories: ["效率工具"]
---


## 1、安装软件
官网安装pycharm，IDE版本选择2019.3  
官网地址：https://www.jetbrains.com/

## 2、修改host文件
Mac：/etc/hosts 在hosts文件追加
```
0.0.0.0 account.jetbrains.com  
0.0.0.0 www.jetbrains.com
```

## 3、破解代码包
链接: https://pan.baidu.com/s/1iKGa5WG0Ezf3EFhl66mfSw?pwd=irfn 提取码: irfn

1. 下载破解代码包，放本地路径，例如：/Users/Develop/jetbrains/tool/jetbrains-agent.jar
2. 填写代理路径：Help =〉Edit Custom VM Options =〉内容末尾追加一行代理路径
```

-javaagent:/绝对路径/jetbrains-agent.jar

# 举例
Mac: -javaagent:/Users/Develop/jetbrains/tool/jetbrains-agent.jar  
Linux: -javaagent:/Users/Develop/jetbrains/tool/jetbrains-agent.jar
windows: -javaagent:C:\Users\Develop\jetbrains\tool\jetbrains-agent.jar
```

## 4、代理破解器
注意：激活不了的话，记得重启服务
操作路径：Help =〉register =〉License Server

HttpServer：http://fls.jetbrains-agent.com

## 参考资料
https://zhile.io/2018/08/25/jetbrains-license-server-crack.html




