## Docker容器化技术

![img](http://www.ruanyifeng.com/blogimg/asset/2018/bg2018020901.png)

### 一、Docker介绍

#### 1.1 引言

> 1. 我本地运行没问题、环境不一致
> 2. 在多用户的炒作系统下会相互影响
> 3. 用户量爆增的情况下运维成本过高
> 4. 安装软件成本过高

#### 1.2 Docker的由来



#### 1.3 Docker的思想

> 1. 集装箱
>
>    会将所有的内容放进不同集装箱里，谁需要这些环境就直接拿这个集装箱就可以了
>
> 2. 标准化
>
>    1. 运输的标准化：Docker有一个码头，所有上传的集装箱都放在了这个码头上，当谁需要这个环境就直接指派大海疼去搬运这个集装箱就可以了
>    2. 命令的标准化：Docker提供了一系列命令，帮助我们去获取或上传集装箱等等
>    3. 提供了REST的api:衍生出很多的图形化界面，Rancher
>
> 3. 隔离性
>
>    Docker在运行集装箱的内容时，会在Linux内核中单独开辟一些空间，这片空间不会影响到其他程序

> - 注册中心上面放的就是集装箱
> - 镜像（集装箱）
> - 容器（运行起来的镜像）

### 二、Docker的基本操作

#### 2.1 安装Docker

##### 2.1.1 Centos安装步骤

> 1. 下载关于Docker的依赖环境  

```bash
yum -y install yum-utils device-mapper-persistent-data lvm2
```

##### 2.1.2 Ubuntu安装步骤

> - 由于apt官方库里的docker版本可能比较旧，所以先卸载可能存在的旧版本：

```bash
$ sudo apt-get remove docker docker-engine docker-ce docker.io
```

> - 更新apt包索引：

```bash
$ sudo apt-get update
```

> - 安装以下包以使apt可以通过HTTPS使用存储库（repository）：

```bash
$ sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```

> - 添加Docker官方的GPG密钥：

```bash
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

> - 使用下面的命令来设置**stable**存储库：

```bash
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

> - 再更新一下apt包索引

```bash
	$ sudo apt-get update
```

> - 安装最新版本的Docker CE：

```bash
$ sudo apt-get install -y docker-ce
```

#### 2.2 Docker中央仓库

> 1. Docker官方中央仓库，镜像最全，下载最慢https://hub.docker.com/
> 2. 国内镜像网站，网易蜂巢，daoCloudhttp://hub.daocloud.io/（推荐使用）等
> 3.  在公司内会使用私服的方式拉取镜像（添加配置）

#### 2.3 镜像的操作

```bash
# 拉取镜像到本地
$ docker pull 镜像名称[:标签（版本号等）]

# 例如
$ docker pull daocloud.io/library/tomcat:8.5.15-jre8
```

```bash
# 查看本地所有镜像
$ docker images

# 删除本地镜像
$ docker rmi 镜像的标识
```

```bash
# 镜像的导入导出
# 将本地镜像导出
$ docker save -o 导出路径 镜像id
# 加载本地的镜像文件
$ docker load -i 镜像文件
```

```bash
# 修改镜像名称
$ docker tag b8 新镜像名称:版本
```

#### 2.4 容器的操作

```bash
# 1.运行容器
$ docker run -d -p 宿主机端口:容器端口 --name 容器名称 镜像的标识|镜像名称[:tag]
# -d 代表后台运行容器
# -p 映射当前Linux端口和容器端口
# --name 指定容器的名称
```

```bash
# 查看正在运行的容器
docker ps -[aq]

# -a 查看所有容器， 包括没运行的容器
# -q 只能看到容器的id
```

```bash
# 查看容器的日志
docker logs -f 容器id
```

```bash
# 进入容器内部
$ docker exec -it 容器id bash 
```

```bash
# 删除容器
# 停止指定容器
$ docker stop 容器id

# 停止全部容器
$ docker stop $(docker ps -qa)

# 删除指定容器
$ docker rm 容器id

# 删除全部容器
$ docker rm $(docker ps -qa)
```

```bash
# 启动容器
docker start 容器id
```



### 三、Docker应用

#### 3.1 准备一个SSM项目

#### 3.2 准备Mysql容器

```bash
docker run -d -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=231509lkm daocloud.io/library/mysql:5.7.4
```

#### 3.3 启动项目容器

> 只需将SSM项目的war/jar包部署到Tomcat容器内部即可，可以通过命令将宿主机的内容复制到容器中

```bash
$ docker cp 文件名称 容器id:容器内部路径
```

#### 3.4 数据卷

> 数据卷:将宿主机的一个目录映射到容器的一个目录

```bash
# 1.创建数据卷
docker volumes creat 数据卷名称
# 创建的数据卷默认存放在/var/lib/docker/volumns/下	

# 查看数据卷的详细信息
docker volumes inspact 数据卷名称

# 3.查看全部数据卷
docker volumes ls

# 4. 删除数据卷
docker volumes rm 数据卷名称

# 5.应用数据卷
# 当映射数据卷时，如果数据卷不存在，docker会自动创建，会将容器内部自带的文件，存放在默认的存放路径中
docker run -v 数据卷名称:容器内部路径 镜像id
# 直接指定一个路径作为数据卷的存放位置，这个路径下是空的
docker run -v 路径:容器内部路径 镜像id
```



### 四、Docker自定义镜像

#### 4.1 常规操作

> 中央仓库上的镜像也是Docker用户自己上传上去的

```bash
# 创建一个Dockerfile 文件
# Dockerfile文件中常用的内容
from: 指定当前自定义镜像依赖的环境
copy: 将相对路径下的内容复制到自定义镜像中
workdir:声明镜像的默认工作目录
cmd:需要执行的命令(在workdir下执行的，cmd可以写多个，但只以最后一个为准)
```

------

```bash
# 将准备好的Dockerfile和相应文件拖拽到Linux操作系统中，通过Docker命令制作镜像
docker buile -t 镜像名称[:tag] Dockerfile文件所在目录(如果是当前目录可用.表示)
```

#### 4.2 本地服务的打包部署(idea整合Docker)

1. idea下载Docker插件

2. 配置Docker服务器地址

   <img src="C:\Users\51665\AppData\Roaming\Typora\typora-user-images\image-20201021153954907.png" alt="image-20201021153954907" style="zoom:50%;" />

3. 编写pom文件，添加插件

   ```xml
   <plugin>
       <groupId>com.spotify</groupId>
       <artifactId>docker-maven-plugin</artifactId>
       <version>0.4.14</version>
       <configuration>
           <dockerHost>http://47.108.24.142:2375</dockerHost>
           <imageName>javaboy1/${project.artifactId}</imageName>
           <imageTags>
               <imageTag>${project.version}</imageTag>
           </imageTags>
           <forceTags>true</forceTags>
           <dockerDirectory>${project.basedir}</dockerDirectory>
           <resources>
               <resource>
                   <targetPath>/</targetPath>
                   <directory>${project.build.directory}</directory>
                   <include>${project.build.finalName}.jar</include>
               </resource>
           </resources>
       </configuration>
   </plugin>
   ```

4. 创建一个Dockerfile文件

   ```bash
   FROM hub.c.163.com/library/java:latest
   VOLUME /tmp #定义数据卷
   ADD target/chaptermybatis-0.0.1-SNAPSHOT.war app.war  #文件的拷贝
   ENTRYPOINT ["java","-jar","/app.war"]   # 需要执行的命令
   ```

5. 执行指令

```bash
mvn clean package docker:build
```

### 五、Docker-Compose

> 之前运行一个镜像，需要添加大量的参数
>
> 可以通过Docker-Compose编写这些参数
>
> Docker-Compose可以帮助我们管理容器
>
> 只需要添加一个docker-compose.yml文件即可

#### 5.1 下载Docker-Compose

> 1. 下载1.24.1版本Docker-Composehttps://github.com/docker/compose/releases/download/1.24.1/docker-compose-Linux-x86_64
> 2. 将下载好的文件拖拽到linux操作系统中
> 3. 赋予docker-compose可执行权限
> 4. 配置环境变量（将docker-compose文件移动到了/usr/local/bin目录下，修改/etc/profile文件，将/usr/local/bin配置到了PATH中）

#### 5.2 使用docker-compose管理容器

> 在yml文件中不要使用制表符

```yml
version: '3.1'
service: 
  mysql:  # 服务名称
      restart: always #只要docker启动，那么这个文件就一起启动
      image: daocloud.io/library/mysql:5.7.6
      container_name: mysql #容器名称
      ports:
        - 3306:3306 #指定端口号的映射
      environment:
        MYSQL_ROOT_PASSWORD: 231509lkm # 指定mysql
        AZ: Asia/Shanghai #指定市区
      volumes: 
        - /opt/docker_mysql_tomcat/mysql_data:/var/lib/mysql #映射数据卷
   tomcats:
     restart: always
     images: docker pull daocloud.io/library/tomcat:8.5.15-jre8
     comtainer_name: tomcat
     ports:
       - 8080:8080
     environment: 
       TZ: Asia/Shanghai
     volumes:
       - /opt/docker_mysql_tomcat/tomcat_webapps:/usr/local/tomcat/webapps
       - /opt/docker_mysql_tomcat/tomcat_logs:/usr/local/tomcat/logs
```

#### 5.3 使用docker-compose命令管理容器

> 使用docker-compose命令时，会默认在当前目录下寻找docker-compose文件

```
# 基于docker-compose启动管理的容器
docker-compose up -d

# 关闭并删除容器
docker-compose down

# 开启|关闭|重启 由docker-compose管理的容器
docker-compose start|stop|restart

# 查看由docker-compose管理的容器
docker-compose ps

# 查看日志
docker-compose logs -f
```

#### 5.4 docker-compose配置Dockerfile使用

> 使用docker-compose.yml文件以及Docfile文件在生成自定义镜像的同时启动当前镜像，并且由docker-compose去管理容器

[docker-compose.yml]()

```yml
# yml文件
version: '3.1'
  services:
    ssm:
      restart: alweys
      build:  # 构建自定义镜像
        compile: ../ # 指定Dockerfile文件所在路径、
        dockerfile: Dockerfile # 指定dockerfile 文件名称
      image: ssm:1.0.1
      container_name: ssm
      ports:
        - 8081:8080
      environment:
        TZ: Asia/Shanghai
```

[Dockerfile文件]()

```
from docker pull daocloud.io/library/tomcat:8.5.15-jre8
copy ssm /usr/local/tomcat/logs
```

----

```
# 可以直接启动docker-compose.yml和Docfile构建的自定义镜像
docker-compose up -d
# 如果自定义镜像已经存在，会直接运行已存在的自定义镜像
# 重新构建自定义镜像
docker-compose build
# 运行前，重新构建
docker-compose up -d --build
```



### 六、Docker CI、CD

#### 6.1 引言

> 如果项目更新了，如何使项目部署的流程化操作自动进行

#### 6.2 CI介绍

> CI（continuous intergration)持续集成
>
> 持续集成：编写代码时，完成一个功能后，立即提交代码到git仓库中，将项目重新构建并且测试
>
> - 快速发现错误
> - 防止代码偏离主分支