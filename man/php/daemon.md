## php 实现守护进程


#### 僵尸进程
```
僵尸进程是指父进程在fork出子进程，而后子进程在结束后，父进程并没有调用wait或者waitpid等完成对其清理善后工作，
导致改子进程进程ID、文件描述符等依然保留在系统中，极大浪费了系统资源。所以，僵尸进程是对系统有危害的，而孤儿进程则相对来说没那么严重。在Linux系统中，我们可以通过ps -aux来查看进程，如果有[Z+]标记就是僵尸进程。
```

 - PHP实现守护进程可以通过 pcntl 与 posix 扩展实现。

 - 编程中需要注意的地方有：

    - 通过二次 pcntl_fork() 以及 posix_setsid 让主进程脱离终端
    - 通过 pcntl_signal() 忽略或者处理 SIGHUP 信号
    - 多进程程序需要通过二次 pcntl_fork() 或者 pcntl_signal() 忽略 SIGCHLD 信号防止子进程变成 Zombie 进程
    - 通过 umask() 设定文件权限掩码，防止继承文件权限而来的权限影响功能
    - 将运行进程的 STDIN/STDOUT/STDERR 重定向到 /dev/null 或者其他流上
    - 如果要做的更好，还需要注意：

    - 如果通过 root 启动，运行时更换到低权限用户身份
    - 及时 chdir() 防止操作错误路径
    - 多进程程序考虑定时重启，防止内存泄露


 - 这里注意到，daemon有如下特征：

    - 没有终端
    - 后台运行
    - 父进程 pid 为1

 - 想要查看运行中的守护进程可以通过 ps -ax 或者 ps -ef 查看，其中 -x 表示会列出没有控制终端的进程。