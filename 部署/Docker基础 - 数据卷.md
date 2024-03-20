# Docker-数据卷

数据做持久化和容器之间数据能够共享。

## 数据卷

**Data Volume**，主机和容器之间共享数据。所有的docker容器内的卷，没有指定目录的时候，默认`/var/lib/docker/volumes/xxx/_data`

```
docker run -v <host path>:<container path> 镜像名
```

具名挂载和匿名挂载

```
-v 容器内路径              # 匿名挂载
-v 卷名:容器内路径         # 具名挂载
-v /宿主机绝对路径目录:/容器内目录   # 指定路径挂载
```

### Dockerfile文件中

```dockerfile
VOLUME ["/dataVolumeContainer1","/dataVolumeContainer2"]
```



## 数据卷容器

**Data Volume Container**

容器之间共享数据。

命名的容器挂载数据卷，其他容器通过挂载这个容器实现数据共享，挂载数据卷的容器，称之为数据卷容器。

数据共享具有传递性。

> 数据卷容器，其实就是一个正常的容器，专门用来提供数据卷供其它容器挂载的。

`--volumes-from  container1 `，以`container1`容器为父容器，完成容器之间数据共享、备份

