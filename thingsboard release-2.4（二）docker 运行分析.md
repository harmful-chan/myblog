## 启动过程详解
Github：https://github.com/thingsboard/thingsboard/tree/release-2.4/docker

安装：执行`./docker-install-tb.sh --loadDemo`，运行`docker-compose -f docker-compose.yml -f docker-compose.postgres.yml up -d redis `和`docker-compose -f docker-compose.yml -f docker-compose.postgres.yml run --no-deps --rm -e INSTALL_TB=true -e LOAD_DEMO=true tb1`，分别启动redis缓存，postgres数据库写入测试数据，tb1应用。

启动服务：执行`./docker-start-services.sh`，运行`docker-compose -f docker-compose.yml -f docker-compose.postgres.yml up -d`后台运行所有容器。    

**ps：官方给的docker文件都是从官方docker hub拉取镜像的，运行自己的程序要自己本机编译改docker-compose.yml镜像只想自己的项目**
<br>

## 启动文件分析
### 1. .env
总结：设置引用到的项目名称，数据库类型
```shell
DOCKER_REPO=thingsboard
JS_EXECUTOR_DOCKER_NAME=tb-js-executor
TB_NODE_DOCKER_NAME=tb-node
WEB_UI_DOCKER_NAME=tb-web-ui
MQTT_TRANSPORT_DOCKER_NAME=tb-mqtt-transport
HTTP_TRANSPORT_DOCKER_NAME=tb-http-transport
COAP_TRANSPORT_DOCKER_NAME=tb-coap-transport
TB_VERSION=latest
DATABASE=postgres    #cassandra
LOAD_BALANCER_NAME=haproxy-certbot
```
### 2. compose-utils.sh
总结：根据`DATABASE=postgres`，设置
`ADDITIONAL_COMPOSE_ARGS="-f docker-compose.postgres.yml"`
```shell
#!/bin/bash
function additionalComposeArgs() {
    source .env    #添加项目
    ADDITIONAL_COMPOSE_ARGS=""
    case $DATABASE in    #相当于switch语句
        postgres)
        ADDITIONAL_COMPOSE_ARGS="-f docker-compose.postgres.yml"
        ;;
        cassandra)
        ADDITIONAL_COMPOSE_ARGS="-f docker-compose.cassandra.yml"
        ;;
        *)
        echo "Unknown DATABASE value specified: '${DATABASE}'. Should be either postgres or cassandra." >&2
        exit 1
    esac
    echo $ADDITIONAL_COMPOSE_ARGS
}

function additionalStartupServices() {
    source .env
    ADDITIONAL_STARTUP_SERVICES=""
    case $DATABASE in
        postgres)
        ADDITIONAL_STARTUP_SERVICES=postgres
        ;;
        cassandra)
        ADDITIONAL_STARTUP_SERVICES=cassandra
        ;;
        *)
        echo "Unknown DATABASE value specified: '${DATABASE}'. Should be either postgres or cassandra." >&2
        exit 1
    esac
    echo $ADDITIONAL_STARTUP_SERVICES
}

```

### 3. docker-install-tb.sh
总结：后台启动`docker-compose.yml`里的`redis`，`docker-compose.postgres.yml`里的`postgres`。`docker-compose.postgres.yml`里的`tb1`。
```shell
$ docker-compose -f docker-compose.yml -f docker-compose.postgres.yml up -d redis 
$ docker-compose -f docker-compose.yml -f docker-compose.postgres.yml run --no-deps --rm -e INSTALL_TB=true -e LOAD_DEMO=true tb1
```

```shell
#!/bin/bash
while [[ $# -gt 0 ]]
...    #如果有传入参数--loadDemo，则设置LOAD_DEMO=true

if [ "$LOAD_DEMO" == "true" ]; then
...    #如果LOAD_DEMO=true，设置loadDemo=true，否则为false

set -e    #遇到错误立即退出

source compose-utils.sh    
ADDITIONAL_COMPOSE_ARGS=$(additionalComposeArgs) || exit $?
ADDITIONAL_STARTUP_SERVICES=$(additionalStartupServices) || exit $?
#compose-utils.sh提供additionalComposeArgs，additionalStartupServices两个方法，设置获取数据库类型和名称


if [ ! -z "${ADDITIONAL_STARTUP_SERVICES// }" ]; then
    docker-compose -f docker-compose.yml $ADDITIONAL_COMPOSE_ARGS up -d redis $ADDITIONAL_STARTUP_SERVICES
fi
#把ADDITIONAL_STARTUP_SERVICES替换为“ ”，-z判断长度为0则为真，！取反
#添加ADDITIONAL_COMPOSE_ARGS参数，后台运行redis


docker-compose -f docker-compose.yml $ADDITIONAL_COMPOSE_ARGS run --no-deps --rm -e INSTALL_TB=true -e LOAD_DEMO=${loadDemo} tb1
#运行tb，添加环境变量INSTALL_TB=true，LOAD_DEMO=true
```

### 3. docker-start-services.sh
总结：`docker-compose -f docker-compose.yml -f docker-compose.postgres.yml up -d`，后台运行所有容器。
```shell
#!/bin/bash
set -e

source compose-utils.sh

ADDITIONAL_COMPOSE_ARGS=$(additionalComposeArgs) || exit $?

docker-compose -f docker-compose.yml $ADDITIONAL_COMPOSE_ARGS up -d
```