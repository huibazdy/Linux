## 解压缩



### .tar.xz



***命令参数解释***

* `-x`：解压
* `-f`：解压指定压缩文件
* `-v`：显示正在解压的文件名称
* `-C`：解压到指定目录

***解压到当前目录***

```shell
tar -xf archive.tar.xz
```

***解压到指定目录***

```shell
tar -xf archive.tar.xz -C [directory]
```



### .tar.gz 或 .tgz

* 解压

    ```shell
    tar -zxvf <filename>   #解压到当前目录
    tar -C <Des_path> -zxvf <filename>  #解压到指定目录
    ```

* 压缩

    ```shell
    tar -zcvf <filename> <direcory_name>  #将Directory压缩为.tar.gz文件
    ```





## 查找文件

* ***locate***：`locate <filename>`（一周更新一次文件库，但效率比find命令高）
* ***find***：`find [path] -name <filename>`



## PS命令

监测运行在系统上的程序（进程），ps 命令工具可以输出很多和进程相关的信息，可以称的上的是管理工具中的瑞士军刀。但是随之而来的是巨量参数，这使得 ps 成为了最难掌握的命令。

大多数系统管理员掌握了能提供他们需要信息的一组参数后，就一直会使用这组参数。
