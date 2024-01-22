---
title: shell-cron-check-process
date: 2024-01-22 15:19:05
tags: shell
---

### Shell + cron 来解决服务挂掉自启动
`前言`: 有时候你必须要去检测某个进程是否还存在，当不存在的时候启动服务，来保障服务的正常运行。当然这个必须慎用，最主要的还是要保障服务的稳定性，但是这个也有应用场景。

#### Shell
简单介绍一下，就是用户与计算机进行交互的，常见的命令行 比如：cd、mkdir 等

下面是一个简单的检测java进程是否存在，存在则不作为，不存在则去启动的脚本。
```
#!/bin/bash

# 设置 Java 安装路径，如果未设置，则使用默认路径
if [ ! -n "$JAVA_HOME" ]; then
    JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.262.b10-1.el7.x86_64/jre
fi

# 设置 Java 进程名称
java_a_process_name="$JAVA_HOME/bin/java -jar /home/tomcat/apps_test/a-SNAPSHOT.jar" #这里是进程名称 可以通过 ps -ef | grep 来查看
java_b_process_name="$JAVA_HOME/bin/java -jar /home/tomcat/apps_test/b-SNAPSHOT.jar" 
java_c_process_name="$JAVA_HOME/bin/java -jar /home/tomcat/apps_test/c-SNAPSHOT.jar" 
java_d_process_name="$JAVA_HOME/bin/java -jar /home/tomcat/apps_test/d-SNAPSHOT.jar" 


# 检查Java进程是否存在
if ! pgrep -f "$java_a_process_name" > /dev/null; then
    # 如果Java进程不存在，启动它
    echo ">>>>> Not found service SMS Java process, start it ...."
    
    # 在这里添加启动Java进程的命令，例如：
    /home/tomcat/apps_test/start.sh a-SNAPSHOT.jar
    
    # 请根据你的实际情况修改上面的启动命令
    echo ">>>>> Java process service SMS start end"
else
    # 如果Java进程已经在运行，输出提示信息
    echo ">>>>> Java process service SMS is running"
fi

......

```

#### Cron
简单介绍一下，cron是用于定期执行任务的时间管理工具，大部分linux服务器都默认已经安装了

用法简介：
```
# 查看当前用户的crontab

crontab -l

#编辑当前用户的 Crontab (也就是新增啊、修改定时任务使用的)

crontab -e

#Cron 表达式格式

* * * * * command-to-be-executed    

这里不得不提到 5个 *是如何使用的
5个* 分别代表：分（0-59）、小时（0-23）、日期 (1 - 31)、月份 (1 - 12)、星期几 (0 - 6，其中 0 是星期天)


举几个例子：
0 3 * * * /path/to/your/script.sh     （每天早上3点执行script.sh 脚本）
0 4 * * 1 /path/to/your/script.sh      (每周一早上4点执行)
*/30 * * * * /path/to/your/script.sh   (每隔30分钟执行)


```
