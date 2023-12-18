# Linux_Process_Command



> **`pidof`**

例如：

```shell
pidof chrome
```

结果：

```
14325 14258 13788 13689 12083 12026 4063 3480 3446 3444 3422 3418 3417 3401
```

根据进程名称显示了所有 PID（包括父进程和子进程）。因此我们需要找出父进程PID （PPID）



> **`pgrep`**

输入：

```
pgrep barrier
```

输出：

```
18193
18289
```

与上述功能类似，但是会按照结果从小到大排序，这说明父进程PID是最后一个（18289）



> **`pstree`**

将运行的进程显示未一棵树，树根是最上层的父进程，如果省略参数，树根就是 init 进程。

```shell
pstree -p | grep "chrome"
```

输出：

![](https://raw.githubusercontent.com/huibazdy/TyporaPicture/main/pstree.png)

该命令的特点是隔离了父进程与子进程，可视化显示某个应用程序的进程关系。



> **`ps`**

显示进程活动信息：

```shell
ps -aux | grep "chrome"
```

可以根据创建日期来看出谁是父进程。