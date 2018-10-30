# 从零搭建FastDFS

### FastDFS介绍

#### 简介

FastDFS是开源的高性能分布式文件系统（DFS），它的主要功能包括：文件存储、文件同步、文件访问，以及高容量和负载平衡。主要解决了海量数据存储问题，特别是以中小文件（建议范围：4KB < file_size < 500MB）为载体的在线服务。

FastDFS 有三个角色：**跟踪服务器Tracker Server**、**存储服务器 StorageServer**、**客服端Client**

* **Tracker Serve**r： 跟踪服务器，主要做调度工作，起到均衡的作用；负责管理所有的storage server和group，每个storage启动后会连接tracker，告知自己的group信息，并保持周期性心跳。
* **Storage Server**：存储服务器，主要提供容量和备份服务；以group为单位，每个group内可以有多台storage server，数据互为备份（组内冗余）。
* **Client**：客户端，上传和下载数据的服务器，也就是我们项目部署的服务器。

![](images\FastDFS结构图.png)

#### FastDFS的存储策略

为了支持大容量，存储节点（服务器）采用了分组（卷)group的方式。存储系统由一个或多个组构成，组与组之间是相互独立的，所有组的文件累加就是整个存储系统中的文件容量。**一个组由一台或多台服务器组成，一个组下面的存储服务器中的文件都是相同的**，组中的多台存储服务器起到了冗余备份（组内冗余）和负载均衡的作用。

在组中添加服务器时，文件同步由系统自动完成，同步完成后，自动将新服务器切换到线上提供服务。当存储空间耗尽时，可以动态添加组，这样就增加的存储系统的容量。

**FastDFS的上传过程**

FastDFS提供了基本的文件访问接口，如upload、download、append、delete等，以客户端库的方式提供给用户使用。

Storage Server会定期的向Tracker Server发送自己的存储信息。当Tracker Server Cluster中的Tracker Server不止一个时，各个Tracker之间的关系是对等的，所以客户端上传时可以选择任意一个Tracker。

当Tracker收到客户端上传文件的请求时，会为该文件分配一个可以存储文件的group，当选定了group后就要决定给客户端分配group中的哪一个storage server。当分配好storage server后，客户端向storage发送写文件请求，storage将会为文件分配一个数据存储目录。然后为文件分配一个fileid，最后根据以上的信息生成文件名存储文件。

![](images\FastDFS上传时序图.png)

**FastDFS的文件同步**

写文件时，客户端将文件写入group内一个storage server即认为是写入成功。随后，会由后台线程将文件同步至同group其他storage server。

**FastDFS的文件下载**

客户端uploadfile成功后，会拿到一个storage生成的文件名，接下来客户端根据这个文件名即可访问到该文件。

![](images\FastDFS文件下载时序图.png)

跟upload file一样，在downloadfile时客户端可以选择任意tracker server。tracker发送download请求给某个tracker，必须带上文件名信息，tracke从文件名中解析出文件的group、大小、创建时间等信息，然后为该请求选择一个storage用来服务读请求。



### 安装FastDFS环境

选择安装[官网](https://github.com/happyfish100/fastdfs)最新版本，因为相关文件都在github上，所以都通过git方式下载

首先创建相关文件夹

```shell
mkdir /home/dfs
mkdir /home/dfs/storage
mkdir /home/dfs/tracker
mkdir /home/dfs/client
mkdir /home/dfs/file
```



#### 安装libfatscommon

libfatscommon是fastDFS和fastDHT抽取出的公共函数库，需先安装。安装文件都放在 /usr/local/src下

```shell
cd /usr/local/src 
git clone https://github.com/happyfish100/libfastcommon.git --depth 1
cd libfastcommon
./make.sh && ./make.sh install #编译安装
```

#### 安装FastDFS

```shell
cd ../  #返回到 /usr/local/src
git clone https://github.com/happyfish100/fastdfs.git --depth 1
./make.sh && ./make.sh install #编译安装
cp /etc/fdfs/tracker.conf.sample /etc/fdfs/tracker.conf
cp /etc/fdfs/storage.conf.sample /etc/fdfs/storage.conf
cp /etc/fdfs/client.conf.sample /etc/fdfs/client.conf #客户端文件，测试用
cp /usr/local/src/fastdfs/conf/http.conf /etc/fdfs/ #供nginx访问使用
cp /usr/local/src/fastdfs/conf/mime.types /etc/fdfs/ #供nginx访问使用
```

#### 安装fastdfs-nginx-module

模块说明：

​	FastDFS 通过 Tracker 服务器，将文件放在 Storage 服务器存储， 但是同组存储服务器之间需要进行文件复制， 有同步延迟的问题。假设 Tracker 服务器将文件上传到了 storage A ，上传成功后文件 ID已经返回给客户端。此时 FastDFS 存储集群机制会将这个文件同步到同组存储  storage B，在文件还没有复制完成的情况下，客户端如果用这个文件 ID 在 storage B  上取文件,就会出现文件无法访问的错误。而 fastdfs-nginx-module 可以重定向文件链接到源服务器取文件，避免客户端由于复制延迟导致的文件无法访问错误。

```shell
cd ../ #返回上一级目录
git clone https://github.com/happyfish100/fastdfs-nginx-module.git --depth 1
cp /usr/local/src/fastdfs-nginx-module/src/mod_fastdfs.conf /etc/fdfs
```

#### 安装nginx

```shell
wget http://nginx.org/download/nginx-1.15.4.tar.gz #下载nginx压缩包
tar -zxvf nginx-1.15.4.tar.gz #解压
cd nginx-1.15.4/
#添加fastdfs-nginx-module模块
./configure --add-module=/usr/local/src/fastdfs-nginx-module/src/ 
make && make install #编译安装
```

### 单机部署

#### tracker配置

```shell
vim /etc/fdfs/tracker.conf
#以下为修改项目
port=22122 #一般不作修改
base_path=/home/dfs/tracker #存储日志和storage相关信息
```

当tracker server启动成功后，会在{base_path}下，即/home/dfs/tracker创建data、logs两个目录，结构如下

```
${base_path}
  |__data
  |   |__storage_groups.dat：存储分组信息
  |   |__storage_servers.dat：存储服务器列表
  |__logs
  |   |__trackerd.log： tracker server 日志文件 
```

#### storage配置

```shell
vim /etc/fdfs/storage.conf
#需要修改的内容如下
port=23000  # storage服务端口（默认23000,一般不修改）
base_path=/home/dfs/storage   # 数据和日志文件存储根目录
store_path0=/home/dfs/file  # 第一个存储目录
tracker_server=127.0.0.1:22122  # tracker服务器IP和端口
http.server_port=8888  # http访问文件的端口(默认8888,看情况修改,和nginx中保持一致)
```

#### client测试配置

```shell
vim /etc/fdfs/client.conf
#需要修改的内容如下
base_path=/home/dfs/client
tracker_server=127.0.0.1:22122    #tracker服务器IP和端口
#保存后测试,返回ID表示成功 如：group1/M00/00/00/xx.tar.gz
fdfs_upload_file /etc/fdfs/client.conf /usr/local/src/nginx-1.15.4.tar.gz
```

