# blog
个人学习笔记


# 复制到本地编辑
## 下载到本地
```sh
C02YF15AJG5J:~ yuandb$ mkdir blog
C02YF15AJG5J:~ yuandb$ cd blog
C02YF15AJG5J:blog yuandb$ ls
C02YF15AJG5J:blog yuandb$ git clone git@github.com:yuanduobao/blog.git
Cloning into 'blog'...
The authenticity of host 'github.com (20.205.243.166)' can't be established.
RSA key fingerprint is SHA256:nThbg6kXUpJWGl7E1IGOCspRomTxdCARLviKw6E5SY8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'github.com,20.205.243.166' (RSA) to the list of known hosts.
remote: Enumerating objects: 14, done.
remote: Counting objects: 100% (14/14), done.
remote: Compressing objects: 100% (13/13), done.
remote: Total 14 (delta 5), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (14/14), 6.82 KiB | 1.71 MiB/s, done.
Resolving deltas: 100% (5/5), done.
C02YF15AJG5J:blog yuandb$ ls
blog
C02YF15AJG5J:blog yuandb$ pwd
/Users/yuandb/blog
C02YF15AJG5J:blog yuandb$
```
## 本地创建新子目录
```sh
C02YF15AJG5J:blog yuandb$ cd blog
C02YF15AJG5J:blog yuandb$ ls
LICENSE		README.md
C02YF15AJG5J:blog yuandb$ mkdir pglogical
C02YF15AJG5J:blog yuandb$ cd pglogical/
C02YF15AJG5J:pglogical yuandb$ cp -R /System/Volumes/Data/yuandb/01-work/10-笔记/01-gitlab-workspace/documents/04-postgresql/14-源码阅读/07-逻辑复制/* .
C02YF15AJG5J:pglogical yuandb$
C02YF15AJG5J:pglogical yuandb$ ls
01-WAL2Mongo源码解析.md		01-WAL2Mongo源码解析.pdf	images
C02YF15AJG5J:pglogical yuandb$ rm 01-WAL2Mongo源码解析.pdf
C02YF15AJG5J:pglogical yuandb$ 
```

## 新增文件
```sh

git add -A
```

## 提交到git
```sh

C02YF15AJG5J:pglogical yuandb$ git commit -m 'add files'
[main 3fc4bb7] add files
 5 files changed, 3369 insertions(+)
 create mode 100644 "pglogical/01-WAL2Mongo\346\272\220\347\240\201\350\247\243\346\236\220.md"
 create mode 100644 pglogical/images/image-20210823093730110.png
 create mode 100644 pglogical/images/image-20210823094044960.png
 create mode 100644 pglogical/images/image-20210823095400091.png
 create mode 100644 pglogical/images/image-20210823095813101.png
C02YF15AJG5J:pglogical yuandb$ 
```


## 推送本地文件到git服务器
```sh
C02YF15AJG5J:pglogical yuandb$ git push
Enumerating objects: 10, done.
Counting objects: 100% (10/10), done.
Delta compression using up to 12 threads
Compressing objects: 100% (9/9), done.
Writing objects: 100% (9/9), 2.29 MiB | 1.33 MiB/s, done.
Total 9 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To github.com:yuanduobao/blog.git
   98f2e31..3fc4bb7  main -> main
C02YF15AJG5J:pglogical yuandb$
```

