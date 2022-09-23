# 收集个人在生产环境常遇到的问题排查技巧（包括：redis,mysql,tomcat,nginx,多线程，jvm日志等）

## Redis
 - 内存报警分析案例
 - 阻塞式命令引发的问题

## mysql
- explain进行sql分析关注点
- 数据库dble中间件配置读写分离策略后，但读请求被强制分配到主节点问题

## 数据库中间件
- mycat
- dble

## tomcat


## nginx

## 多线程
 - 多线程实现不同方式
 - 核心线程数设定标准
 - 线程池参数说明
 - 钩子方法使用
 - 多线程导致服务器灰度重启不生效问题分析


## jvm
   - 排查类加载器加载jar文件后，出现loadClass抛出ClassNotFound异常
   
## 全链路监控之pinpoint
- 服务器负载数据分析

- 调用链路分析
