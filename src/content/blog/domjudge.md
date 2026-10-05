---
title: "DomJudge比赛环境部署(未实践)"
date: "2026-10-02T00:00:00+08:00"
description: "部署DomJudge比赛的环境"
categories:
  - "竞赛"
tags:
  - "domjudge"
  - "ubuntu"
  - "oj"
---

# Ubuntu Server 24.04 LTS
这里简要说一下
- 安装时换源，填写清华源像，例如：
```
https://mirrors.tuna.tsinghua.edu.cn/ubuntu
```
- 记得勾选 `OpenSSH`

# DomJudge比赛环境部署
---
## 配置DomJudge & Judgehost

### 声明
  本文档参考[配置DomJudge & Judgehost](https://blog.techmczs.top/p/configuredj/)
### 环境要求
- domjudge/domserver: 9.0.0
- domjudge/judgehost: 9.0.0
- mariadb: 11.8.3
- System: Ubuntu Server 24.04 LTS
### 前置工作
1. GRUB 配置
    Domjudge 的 judgehost 依赖 cgroup 进行资源限制（内存、CPU）。Ubuntu 24.04 默认使用 cgroup v2，需要额外配置
    - 修改 GRUB
    ```bash
    sudo nano /etc/default/grub
    ```
    找到 `GRUB_CMDLINE_LINUX_DEFAULT`，修改为：
    ```
    GRUB_CMDLINE_LINUX_DEFAULT="quiet cgroup_enable=memory swapaccount=1 isolcpus=2"
    ```
    - 参数说明：
      - `cgroup_enable=memory`：启用 memory cgroup
      - `swapaccount=1`：启用 swap 统计
      - `isolcpus=2`：隔离 CPU 核心 2 给 judgehost 专用（根据核心数调整，多核可写 `isolcpus=2,3`）
  
    - 重启并验证
    ```bash
    sudo update-grub
    sudo reboot
    ```
    重启后执行：
    ```bash
    # 检查 GRUB 参数
    # 应包含 cgroup_enable=memory swapaccount=1 isolcpus=2
    cat /proc/cmdline

    # 应输出 cgroup2fs（确认是 cgroup v2）
    stat -fc %T /sys/fs/cgroup
    ```

### 系统配置
设置时区、软件源、更新系统、安装基础工具
```bash
# 设置时区
sudo timedatectl set-timezone Asia/Shanghai

# 更换软件源（可选，国内推荐清华源）
sudo nano /etc/apt/sources.list

# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装基础工具
sudo apt install -y vim curl wget git nginx
```


### Docker 安装与配置

```bash
curl -fsSL https://get.docker.com -o install-docker.sh
# 建议加上镜像参数
sudo sh install-docker.sh --mirror Aliyun
# sudo sh install-docker.sh

# 将当前用户加入 docker 组（免 sudo）
sudo usermod -aG docker $USER
# 重新登录生效
```

验证：
```bash
docker --version
docker compose version
# 查看服务运行状态
systemctl status docker
# 测试容器拉取与运行
docker run hello-world
```

配置 Docker daemon（关键）

创建/编辑 `/etc/docker/daemon.json`：

```json
{
    "registry-mirrors": [
        "https://docker.1panel.live",
        "https://docker.m.daocloud.io",
        "https://docker.1ms.run",
        "https://hub.rat.dev"
    ],
    "default-cgroupns-mode": "host"
}
```

配置说明：
- `registry-mirrors`：国内镜像加速（解决 Docker Hub 连不上的问题）
- `default-cgroupns-mode: host`：**关键配置**，让容器使用宿主机的 cgroup 命名空间，解决 judgehost cgroup v2 问题

> **注意**：docker-compose 不支持 `cgroupns: host` 参数（会报 `cgroupns false schema`），必须通过 daemon.json 全局配置。

重启 Docker

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```
验证：
```bash
docker info | grep -A 5 "Registry Mirrors"
docker info | grep "Cgroup"
```
---

### 安装目录结构
采用按应用分目录的方案：

```
/opt/
├── dpanel/                  # Docker 管理面板（可选）
│   ├── docker-compose.yaml
│   └── data/
└── domjudge/                # Domjudge 主目录
    ├── docker-compose.yaml
    ├── database.secret      # 数据库密码
    ├── judgehost.secret     # judgehost API 密码
    └── data/
        └── mysql/           # MariaDB 数据持久化,数据自动生成
```
---
### Dpanel Lite 安装（可选）

创建目录
```bash
sudo mkdir -p /opt/dpanel
# data 子目录由 Docker 自动创建，无需手动创建
```
在 `/opt/dpanel/` 目录下创建 `docker-compose.yaml` 中配置
```yaml
services:
  dpanel:
    image: dpanel/dpanel:lite
    container_name: dpanel # 更改此名称后，请同步修改下方 APP_NAME 环境变量
    restart: always
    ports:
      - 8807:8080 # 替换 8807 可更改面板访问端口
    environment:
      APP_NAME: dpanel # 请保持此名称与 container_name 一致
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /opt/dpanel/data:/dpanel # 将 /opt/dpanel/data 更改为你想要挂载的宿主机目录
```
保存后启动
```bash
cd /opt/dpanel
docker compose up -d
```
验证
```bash
docker ps
docker compose ps
```
看到 `dpanel` 状态 `Up`，访问 `8807` 这个端口即可登录面板

### 配置 Mariadb Domjudge Judgehost
```bash
sudo mkdir -p /opt/domjudge
# data 子目录由 Docker 自动创建，无需手动创建
```
切换到/opt/domjudge目录下，新建database.secret，填入
```bash
MYSQL_ROOT_PASSWORD=<YOUR PASSWORD>
MYSQL_PASSWORD=<YOUR PASSWORD>
```
这将设置你的 `Mariadb` 数据库密码。
新建`docker-compose.yaml`，配置文件，填入
```yaml
services:
  dj-mariadb:
    container_name: dj-mariadb
    image: mariadb:11.8.3 # 指定版本，最新版本填写mariadb:latest
    restart: unless-stopped
    ports:
      - "13306:3306"
    volumes:
      - ./data/mysql:/var/lib/mysql
    env_file: database.secret # 引入数据库密码文件
    environment:
      - MYSQL_USER=domjudge
      - MYSQL_DATABASE=domjudge
      - CONTAINER_TIMEZONE=Asia/Shanghai
    command: --max-connections=1024 --max-allowed-packet=1G --innodb-snapshot-isolation=OFF --innodb-log-file-size=512M
    healthcheck:
      test: mysqladmin ping -h localhost -u $$MYSQL_USER --password=$$MYSQL_PASSWORD
      start_period: 10s
      interval: 5s
      timeout: 1s
      retries: 5

  domserver:
    container_name: domserver
    image: domjudge/domserver:9.0.0 # 指定版本，最新版本填写domjudge/domserver:latest
    restart: unless-stopped
    ports:
      - "5881:80"
    links:
      - 'dj-mariadb:mariadb'
    depends_on:
      dj-mariadb:
        condition: service_healthy
    env_file: database.secret # 引入数据库密码文件
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      - MYSQL_HOST=mariadb
      - MYSQL_USER=domjudge
      - MYSQL_DATABASE=domjudge
      - CONTAINER_TIMEZONE=Asia/Shanghai
      - WEBAPP_BASEURL=/dj

  judgehost:
    image: 'domjudge/judgehost:9.0.0'
    links:
      - 'domserver:domserver'
    depends_on:
      domserver:
        condition: service_healthy
    privileged: true
    volumes:
      # 把宿主机的 cgroup 目录“共享”给判题容器，
      # 让 judgehost 能限制每份提交代码的 CPU、内存用量。
      # Ubuntu 24.04 是 cgroup v2，所以挂载整个 /sys/fs/cgroup。
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    env_file: judgehost.secret
    environment:
      - CONTAINER_TIMEZONE=Asia/Shanghai
      - DOMSERVER_BASEURL=http://domserver/dj/
    deploy:
      mode: replicated
      replicas: 2 # 评测机数量
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 5
```
保存后启动
```bash
cd /opt/domjudge
# 初始化 DOMjudge 和 数据库
sudo docker compose up -d dj-mariadb domserver
```
确认 mariadb 与 domserver 都 `healthy`：
> **如果遇到提示**
> ```
> dj-mariadb unhealthy[×]
> ```
> 进入docker终端（也可用dpanel进入）
> ```bash
> sudo docker exec -it dj-mariadb /bin/bash
> ```
> 进入后运行
> ```bash
> apt update
> apt install mysql-client
> ```
> 完成后`Ctrl+D`退出容器，再尝试执行
> ```bash
> sudo docker compose up -d dj-mariadb domserver
> ```
> 确认 mariadb 与 domserver 都 `healthy`


```bash
sudo docker compose ps
```


获取 judgehost API 密码并新建 `judgehost.secret`
```bash
sudo docker exec -it domserver cat /opt/domjudge/domserver/etc/restapi.secret
```
输出第三行default http://... 密码
填入 `judgehost.secret`
```bash
JUDGEDAEMON_PASSWORD=<YOUR PASSWORD>
```
获取管理员初始密码（登录面板用）：
```bash
sudo docker exec -it domserver cat /opt/domjudge/domserver/etc/initial_admin_password.secret
```

启动判题节点
```bash
sudo docker compose up -d
```
验证
```bash
sudo docker compose ps
```
应有 1 个 `dj-mariadb` + 1 个 `domserver` + 与replicas数量相同的 `judgehost` ，全部 `Up`

> **注意**：judgehost 必须依赖前置的 `default-cgroupns-mode: host` daemon 配置，否则会报 `missing cgroup hierarchy prefix under /proc/self/cgroup`。compose 不支持 `cgroupns: host` 参数，只能通过 daemon.json 全局配置。

验证 judgehost 注册成功：
```bash
docker inspect domjudge-judgehost-1 | grep CgroupnsMode
# 应显示 "CgroupnsMode": "host"
docker logs domjudge-judgehost-1 --tail 10
# 应显示 Judge started / Registering judgehost / No submissions in queue
```

## Nginx 反向代理

### 3.1 编写 Nginx 配置
Nginx 在上文已经安装，这里直接配置反向代理。
```bash
sudo nano /etc/nginx/sites-available/domjudge
```

```nginx
server {
    listen 80;
    server_name _;

    client_max_body_size 64M;

    location /dj/ {
        proxy_pass http://127.0.0.1:5881/dj/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
    }

    # 根路径重定向到 /dj/
    location = / {
        return 301 /dj/;
    }
}
```

### 3.2 启用配置

```bash
sudo ln -sf /etc/nginx/sites-available/domjudge /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```
---

## Domjudge 初始化与 Example 比赛
这里暂时写了，到时候写
- 比赛配置
- 队伍数据批量导入

---

### ICPC tools 配置
[下载 ICPC tools](https://tools.icpc.global/)
#### 1. CDS 配置

#### 2. Resolver 配置

#### 3. Presentation 滚榜
