# Docker compose demo with spring boot mysql


## Docker Compose 预览

docker 命令可以编译，打包，部署image，docker compose可以同时部署运行多个容器，docker compse使用YAML文件进行相关service的配置。你可以使用一行简单的命令启动配置文件中的所有服务。

使用docker-compose一般有以下步骤：

1. 编写Dockerfile，定义将要部署的应用

2. 编写docker-compose.yml文件，在配置文件中定义服务，他们可以运行在独立的容器之中

3. 执行docker-compose up,启动配置文件中的所有服务

## 创建项目

创建项目，这里使用spring-boot创建一个mysql的读写应用，该应用依赖mysql服务，所以我们的应用包括两个部分

一个是后台服务由spring boot 编写，另外一个是mysql提供数据库服务。具体[spring boot mysql](git@github.com:JetQin/spring-boot-mysql.git)

## 编写Dockerfile

Dockerfile  用来将我们的应用编译成image， 然后通过把image部署到容器中向外提供服务

```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ARG JAVA_OPTS
ENV JAVA_OPTS=$JAVA_OPTS
ADD target/gs-mysql-data-0.1.0.jar gs-data.jar
EXPOSE 5001
ENTRYPOINT exec java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar gs-data.jar
```

## 编写docker-compose

docker-compose.yml文件中定义了两个服务spring-boot-gs-data， 以及db.  这里有两种方式, 一是在容器内，二是通过docker-machin。

**docker 容器部署**，mysql的服务名定义为***db***，不对外提供服务，宿主机无法访问db服务，但是其他容器可以通过服务名db调用，如果你用的是mysql。可以这样配置jdbc url  ***jdbc:mysql://db:3306/db_example***

```
version: '3.3'
services:
  db:
    image: mysql
    container_name: mysql-container
    environment:
      - MYSQL_USER=springuser
      - MYSQL_PASSWORD=ThePassword
      - MYSQL_DATABASE=db_example
      - MYSQL_ROOT_PASSWORD=root

  gs_data:
    build: .
    image: spring-boot-gs-data
    container_name: gs-data-container
    ports:
      - "8080:8080"
    depends_on:
      - db
    environment:
      - DB_HOST=db
      - DB_PORT=3306
      - DB_USER=springuser
      - DB_PASSWORD=ThePassword
      - DB_NAME=db_example
```

通过docker-machine 访问，mysql 向宿主机暴露了端口，此时宿主机可以访问到mysql提供的数据库服务，此时访问此数据库服务需要使用docker-machine 的ip地址，可以通过***docke-machine ip （machine 名字）*** 获取ip地址

```
version: '3.3'                               # 定义compose 版本号
services:                                    # Service 定义
 db:                                         # 定义第一个服务名称
   image: mysql                              # 定义拉取的image名字
   container_name: mysql-container           # 定义container名称
 ports:
   - "3000:3306"                             # 对外提端口，宿主机端口为3000，容器端口3306
 environment:
   - MYSQL_USER=springuser                   # mysql默认用户，启动时创建
   - MYSQL_PASSWORD=ThePassword              # mysql默认密码，启动时创建
   - MYSQL_DATABASE=db_example               # mysql默认数据库，启动时创建
   - MYSQL_ROOT_PASSWORD=root                # mysql默认root密码，启动时创建
 gs_data:                                    # 定义第二个服务名称
   build: .                                  # 使用Dockefile进行build
   image: spring-boot-gs-data                # 编译好的image name
   container_name: gs-data-container         # 部署时容器名称
   ports:                                    # 对外提供服务端口
     - "8080:8080"
   depends_on:                               # 依赖于第一个服务
     - db
   environment:                              # 定义服务需要的一些环境变量
     - DB_HOST=192.168.99.100
     - DB_PORT=3000
     - DB_USER=springuser
     - DB_PASSWORD=ThePassword
     - DB_NAME=db_example
```

application.properties, 这里使用占位符动态获取数据库配置信息，配置信息由docker-compose启动时动态传入

```
server.port=8080
spring.jpa.hibernate.ddl-auto=create
spring.datasource.url=jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name = com.mysql.jdbc.Driver
spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.MySQL5Dialect
```

|       | 对宿主机开放端口 | 访问数据库方式  |
|:----- | -------- | -------- |
| 第一种方式 | 否        | 通过服务名字调用 |
| 第二种方式 | 是        | 通过ip地址调用 |

***如果服务少的化，可以使用第一种方式部署，如果docker是一个集群，最好使用第二种方式***

## 运行

启动时只需要执行如下命令即可一次性部署db，和gs_data两个服务，具体是否还需要其他服务，可以依赖你的需求定制

docker-compose.yml

```
docker-compose up          //前台启动，启动时会在命令行打印相关的日志信息
docker-compose up -d       //后台启动
```

输出

```
docker-compose up
Building gs_data
Step 1/7 : FROM openjdk:8-jdk-alpine
 ---> 97bc1352afde
Step 2/7 : VOLUME /tmp
 ---> Running in 6e580cf16d5b
Removing intermediate container 6e580cf16d5b
 ---> 0ce3e37f3d5f
Step 3/7 : ARG JAVA_OPTS
 ---> Running in efebcd3176fc
Removing intermediate container efebcd3176fc
 ---> 547b511e91e2
Step 4/7 : ENV JAVA_OPTS=$JAVA_OPTS
 ---> Running in ea93bcb1b04f
Removing intermediate container ea93bcb1b04f
 ---> 1bf53f6ed368
Step 5/7 : ADD target/gs-mysql-data-0.1.0.jar gs-data.jar
 ---> 975ebe826c4a
Step 6/7 : EXPOSE 5001
 ---> Running in 127bf2247342
Removing intermediate container 127bf2247342
 ---> 7d7006130982
Step 7/7 : ENTRYPOINT exec java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar gs-data.jar
 ---> Running in f1eab4b0251d
Removing intermediate container f1eab4b0251d
 ---> 08fb51304c50

Successfully built 08fb51304c50
Successfully tagged spring-boot-gs-data:latest
WARNING: Image for service gs_data was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Starting mysql-container ... done
Creating gs-data-container ... done
```

验证：

```
curl -s "http://192.168.99.100:8080/demo/add?name=Jet&email=test@test.com"  //添加用户
Saved

curl -s "http://192.168.99.100:8080/demo/all"          //查询
[{"id":1,"name":"Jet","email":"test@test.com"}]
```
