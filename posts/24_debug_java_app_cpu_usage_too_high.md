---
title: 记一次java应用cpu占用过高debug
date: 2023-09-25 12:58:59
permalink: debug_java_app_cpu_usage_too_high/
---

上周发生的一件小事，因为过程还比较顺利，感觉也还比较有参考性，就顺便记录下来。

起因是偶然发现运行`hentai@home` 的机器出现了cpu吃满的情况。这个情况算是有一点异样，因为根据以往经验，这个类pcdn的应用属于IO密集型，不太可能会占用这么高的cpu。于是我试着杀掉进程后重启。重启后新进程的cpu使用率基本维持在10%上下，从实际运行状况看也没有什么异常。

结果过了几个小时后，发现又飙到了100%。这下看起来可能是触发了什么异常状况了。第一时间尝试升级应用到最新版本，问题依旧。清理掉应用缓存重新配置？这个又代价太大了，怕是要个把月才能完全恢复过来。

于是开始考虑跟进去查一下问题，`hentai@home` 是一个单体的java应用，因此先考虑用常规的java应用调试方法追查。

1. 首先用top命令查到cpu占用异常的进程(典型的输出如下)

    ```console
    red@red2:/cantos/app/hath$ top
    top - 05:02:02 up 36 days, 20:26,  2 users,  load average: 6.08, 6.54, 6.37
    Tasks: 123 total,   1 running, 122 sleeping,   0 stopped,   0 zombie
    %Cpu(s): 61.3 us, 38.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.7 si,  0.0 st
    MiB Mem :    964.6 total,     88.7 free,    291.5 used,    584.3 buff/cache
    MiB Swap:   1962.0 total,   1909.7 free,     52.2 used.    488.0 avail Mem
    
        PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    
    3419086 red       20   0 2776900 148548  15160 S 199.0  15.0   1884:33 java
    ```

2. 拿到这个PID后进一步查一下该进程下的线程信息，命令为`ps -mp <PID> -o THREAD,tid,time`

    ```console
    red@red2:/cantos/app/hath$ ps -mp 3419086 -o THREAD,tid,time
    
    USER     %CPU PRI SCNT WCHAN  USER SYSTEM     TID     TIME
    red       183   -    - -         -      -       - 1-07:21:27
    red       0.0  19    - futex_    -      - 3419086 00:00:00
    red       0.0  19    - futex_    -      - 3419087 00:00:00
    red       0.3  19    - futex_    -      - 3419088 00:03:22
    red       0.0  19    - futex_    -      - 3419089 00:00:02
    red       0.0  19    - futex_    -      - 3419090 00:00:00
    red       0.0  19    - futex_    -      - 3419091 00:00:00
    red       0.0  19    - futex_    -      - 3419092 00:00:00
    red       0.0  19    - futex_    -      - 3419093 00:00:18
    red       0.0  19    - futex_    -      - 3419094 00:00:01
    red       0.0  19    - futex_    -      - 3419095 00:00:00
    red       0.0  19    - futex_    -      - 3419097 00:00:04
    red       0.0  19    - futex_    -      - 3419098 00:00:14
    red       0.0  19    - futex_    -      - 3419099 00:00:00
    red       0.0  19    - futex_    -      - 3419100 00:00:01
    red       0.1  19    - poll_s    -      - 3419101 00:01:50
    red      50.5  19    - -         -      - 3448406 08:25:33
    red      43.8  19    - -         -      - 3599194 06:23:30
    red      41.2  19    - -         -      - 3661814 05:42:25
    red      39.7  19    - -         -      - 3748094 05:08:05
    red      37.6  19    - -         -      - 3989393 03:49:56
    red       0.0  19    - skb_wa    -      -  100601 00:00:00
    red      31.8  19    - -         -      -  300264 00:14:09
    red       0.0  19    - inet_c    -      -  347998 00:00:00
    red       0.0  19    - futex_    -      -  354844 00:00:00
    red       0.0  19    - poll_s    -      -  354860 00:00:00
    red       0.0  19    - futex_    -      -  354886 00:00:00
    red       0.0  19    - futex_    -      -  354940 00:00:00
    red       0.0  19    - futex_    -      -  354973 00:00:00
    red       0.0  19    - poll_s    -      -  354991 00:00:00
    red       0.0  19    - poll_s    -      -  355005 00:00:00
    red       0.0  19    - poll_s    -      -  355006 00:00:00
    ```

    到这一步已经能查到具体线程的情况了，可以看到有6个线程的执行时间特别长，且占用很高，可以选一个具体的TID继续追查。

    不过这里的TID是10进制格式，与jvm输出的还有一些区别，因此选择再多一步将格式转成十六进制。

3. TID转十进制方法有很多, 命令行转换如下

    ```console
    red@red2:~$ printf '%x' 3448406
    349e56
    ```

4. 拿到TID后，下一步使用的工具是JDK中的jstack，用于查看堆栈信息

    ```console
    red@red2:~$ jstack 3419086 | grep 349e56
    "Thread-29288" #29309 prio=5 os_prio=0 cpu=24966847.50ms elapsed=45068.84s tid=0x00007fabbc017000 nid=0x349e56 runnable  [0x00007fabf96e0000]
    ```

    如果对这个java应用的源码比较熟悉的话，打印完整的堆栈信息会更利于排查，类似如下

    ```console
    "Thread-5858353" #5859010 prio=5 os_prio=0 cpu=26.86ms elapsed=16.80s tid=0x00007f813404f000 nid=0x1efd22 waiting on condition  [0x00007f813a8e9000]
    java.lang.Thread.State: TIMED_WAITING (sleeping)
            at java.lang.Thread.sleep(java.base@11.0.20.1/Native Method)
            at hath.base.HTTPResponseProcessorProxy.getPreparedTCPBuffer(HTTPResponseProcessorProxy.java:64)
            at hath.base.HTTPSession.run(HTTPSession.java:198)
            at java.lang.Thread.run(java.base@11.0.20.1/Thread.java:829)
    ```

    打印几段，知道代码都在哪儿打转，基本也就知道情况了。不过我对`hentai@home`这个项目并不熟，所以多做一步，打开debug模式连上去再查一下具体信息。

5. 以debug模式启动应用

    `java -jar -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=0.0.0.0:5005 HentaiAtHome.jar`

    这个命令会开启5005端口，这样就可以用集成开发工具连接来做debug。

6. 这里以vscode为例，添加如下debug配置

    ```json
    {
        "type": "java",
        "name": "Remote Debug h@h",
        "request": "attach",
        "hostName": "192.168.2.223", // Java 进程所在的主机名或 IP 地址
        "port": 5005 // 远程调试端口，与你的 Java 进程配置一致
    }
    ```

    然后在vscode里启动调试就能连到目标进程了，找到上面jstack打印的Thread-5858353，简单暂停一下然后单步调试，恩，问题一目了然，陷入死循环了……

    简单查了几个变量值，查到出现问题的文件，发现是文件大小为0时的临界处理有问题。

    猜测空文件大概是非正常退出时导致的未写入，最后简单把这些大小为0的文件删掉，重新启动，问题解决。有空的话，再顺便去e-hentai论坛上反馈一下这个问题吧。
