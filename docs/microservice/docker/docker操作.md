## docker操作
### 1. docker基本操作

操作 | 命令
----------|---------
搜索镜像 | `docker search 关键字`
列举镜像 | `docker images`
查看正在运行的容器 | `docker ps`
查看所有容器 | `docker ps -a`
创建容器 | `docker create 容器名或者容器ID`
启动容器 | `docker start [-i] 容器名`
运行容器，相当于docker create + docker start | `docker run 容器名或者容器ID`
进入容器的命令行（退出容器后容器会停止） | `docker attach 容器名或者容器ID bash`
进入容器的命令行 | `docker exec -it 容器名或者容器ID bash`
查看Docker的底层信息 | `docker inspect 容器名`
停止container | `docker stop 容器ID`
停止所有container | `docker stop $(docker ps -a -q)`
删除container | `docker rm 容器ID`
删除所有container | `docker rm $(docker ps -a -q)`
删除image | `docker rmi imageID`
删除untagged image | `docker rmi $(docker images | grep "^<none>" | awk "{print $3}")`
删除所有image | `docker rmi $(docker images -q)`


### 2. docker登录Ubuntu
运行进入Ubuntu系统：
> `docker run -ti --name ubuntu docker.io/ubuntu bash `

共享宿主机目录到Ubuntu系统中：
> `docker run -it -v /AAA:/BBB ubuntu bash`
这样宿主机根目录中的AAA文件夹就映射到了容器Ubuntu中去了，两者之间能够共享

例子：
```shell
docker run -it \
-v /usr/docker/softwares:/softwares \
--name ubuntu \
docker.io/ubuntu \
bash
```

正确退出系统方式：
> 先按，ctrl+p
> 再按，ctrl+q

退出后 再进入ubuntu：
> 首先用docker ps -a 查找到该CONTAINER ID对应编号（比如：0a3309a3b29e）
> 进入该系统docker attach 0a3309a3b29e （此时没反应，ctrl+c就进入到ubuntu系统中去了）

### 3. 运行redis
```shell
 docker run -d \
 -p 6379:6379 \
 --name myredis \
 docker.io/redis
 ```

 进入redis容器：
 `docker exec -it myredis redis-cli`

 window访问该redis服务：
 `redis-cli -h 10.0.209.123 -p 6379`


### 4. 运行nginx
 ```shell
 docker run -d \
 -p 8080:80 \
 docker.io/nginx
 ```
 说明：8080是宿主机的端口，80是docker中nginx的端口，80端口映射到80端口中

判断8080端口是否开启
 `netstat -anp | grep 8080`

浏览器访问nginx：
`http://10.0.209.123:8080/`

进入docker的nginx中：
`docker exec -it c51972327907 bash`

