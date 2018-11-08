## 安装Docker
- Mac安装:[下载](https://docs.docker.com/docker-for-mac/install/)
- Windows安装:[下载](https://docs.docker.com/docker-for-windows/install/)

### 安装 WPT Agent 和 WPT Server

Docker基本环境配置好后，接下来需要安装 WPT 的包了。WPT 的软件包分为 Agent 和 Server 两个部分，对应：

- [https://hub.docker.com/r/webpagetest/agent/](https://hub.docker.com/r/webpagetest/agent/)
- [https://hub.docker.com/r/webpagetest/server/](https://hub.docker.com/r/webpagetest/server/)

```
$ docker pull webpagetest/server
$ docker pull webpagetest/agent
```

## 启动服务

#### 运行 WPT Server

```
$ docker run -d -p 4000:80 --rm webpagetest/server
```

#### 运行 WPT Agent

```
$ docker run -d -p 4001:80 \
--network="host" \
-e "SERVER_URL=http://localhost:4000/work/" \
-e "LOCATION=Test" \
webpagetest/agent
```

运行上述步骤后，直接访问 http://localhost:4000 即可看到 

#### 依赖检查
WPT 已经安装好了，WPT的对应配置检查可以通过：http://localhost:4000/install 来看它的依赖是否都安装。
![](https://qpic.url.cn/feeds_pic/Q3auHgzwzM4vdyzE3wGKlRgkjoibP8l42fB3rldwvBeAvGamY2icnUJA/)

#### Mac 下 Traffic Shaping问题

OSX 下会遇到`Error configuring traffic-shaping`报错，这是因为OSX下还没有实现 [traffic-shaping](https://github.com/WPO-Foundation/wptagent/issues/23)
![](https://qpic.url.cn/feeds_pic/ajNVdqHZLLArWzStD2HI0v9M7J7vqqtsW98tgXveyC5gxCqqMicB8icw/)

可以去掉traffic shaping特性通过在settings/locations.ini设置一个假的connectivity值，并且在agent运行的时候增加`--shaper`参数。

更好的办法是基于原有的WPT agent/server镜像制作新的Docker镜像，方便后续搭建Docker集群。

## 制作镜像

### Server

创建一个server文件夹，包含Dockerfile和locations.ini文件。

Dockerfile:
```
FROM webpagetest/server
ADD locations.ini /var/www/html/settings/
```

locations.ini:

```
[locations]
1=Test_loc
[Test_loc]
1=Test
label=Test Location
group=Desktop
[Test]
browser=Chrome,Firefox
label="Test Location"
connectivity=LAN
```

本地build镜像
```
$ docker build -t local-wptserver .
```

### Agent

创建一个agent文件夹，包含Dockerfile和script.sh文件。

Dockerfile

```
FROM webpagetest/agent
ADD script.sh /
ENTRYPOINT /script.sh
```

script.sh

```
#!/bin/bash
set -e
if [ -z "$SERVER_URL" ]; then
echo >&2 'SERVER_URL not set'
exit 1
fi
if [ -z "$LOCATION" ]; then
echo >&2 'LOCATION not set'
exit 1
fi
EXTRA_ARGS=""
if [ -n "$NAME" ]; then
EXTRA_ARGS="$EXTRA_ARGS --name $NAME"
fi
python /wptagent/wptagent.py --server $SERVER_URL --location $LOCATION $EXTRA_ARGS --xvfb --dockerized -vvvvv --shaper none
```

让 `script.sh` 可执行

```sh
chmod u+x script.sh
```

制作Agent镜像

```sh
$ docker build -t local-wptagent .
```

## 开始运行一个Webpagetest Docker实例

```sh
$ docker run -d -p 4000:80 local-wptserver
$ docker run -d -p 4001:80 \
--network="host" \
-e "SERVER_URL=http://localhost:4000/work/" \
-e "LOCATION=Test" \
local-wptagent

```

最后访问 http://127.0.0.1:4000 即可查看到效果。
