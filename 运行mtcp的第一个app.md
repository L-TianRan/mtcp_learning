 # 运行mtcp的第一个app

 ## 0.查看example
打开`mtcp/apps/example`文件夹，查看其中`epserver`的介绍。
``` epserver: a simple mtcp-epoll-based web server
    Single-Process, Multi-threaded Usage:
      ./epserver -p www_home -f epserver.conf [-N #cores] 
      ex) ./epserver -p /home/notav/www -f epserver.conf -N 8
```

## 1.创建www文件夹
我在`/home/username`下创建了一个`www`文件夹，新建了仨文件。
```
$ pwd
/home/username/www
$ ls
history  index.html  nonsense.txt
```

## 2.运行epserver
进入`mtcp/apps/example`文件夹，执行命令
```
sudo ./epserver -p /home/litianran/www/ -f epserver.conf -N 16
```
报错
```
...
EAL: Error - exiting with code: 1
  Cause: Cannot init mbuf pool, errno: 12
```
