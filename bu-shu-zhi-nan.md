# 部署指南

### 安装需求

完整安装 InfluxDB 需要  `root` 或者其他管理员账号。

#### InfluxDB 的 TCP 端口号 <a id="influxdb-oss-networking-ports"></a>

默认情况下，InfluxDB 使用以下端口号:

* TCP 端口 `8086` 用于客户端与服务器间使用 InfluxDB API 通信。
* TCP 端口 `8088` 用于RPC 服务执行备份和恢复操作。

以上这些端口都可以在配置文件 `/etc/influxdb/influxdb.conf` 中进行修改。

#### 时间同步 <a id="network-time-protocol-ntp"></a>

InfluxDB 使用的是当前主机的 UTC 的当前时区时间，如果不使用 NTP 进行时间同步，则插入的时间戳可能不正确。

### 安装 InfluxDB

推荐使用 Red Hat 或者 CentOS 进行安装。可以使用`yum`进行最新版本的安装，在 shell 中执行以下内容。

```text
cat <<EOF | sudo tee /etc/yum.repos.d/influxdb.repo
[influxdb]
name = InfluxDB Repository - RHEL \$releasever
baseurl = https://repos.influxdata.com/rhel/\$releasever/\$basearch/stable
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdb.key
EOF
```

yum 仓库配置完成后，可以直接使用以下命令进行安装及启动 InfluxDB 服务。

```text
# for CentOS 6
sudo yum install influxdb
sudo service influxdb start
# for CentOS 7+
sudo yum install influxdb
sudo systemctl start influxdb
```

### 配置 InfluxDB

默认安装的情况下，配置文件为`/etc/influxdb/influxdb.conf` 也可以使用以下两种方式指定配置文件路径：

* 使用 `-config` 参数指定：

  ```text
  influxd -config /etc/influxdb/influxdb.conf
  ```

* 设置 `INFLUXDB_CONFIG_PATH` 环境变量指定：

  ```text
  echo $INFLUXDB_CONFIG_PATH
  /etc/influxdb/influxdb.conf

  influxd
  ```

  在提供的默认配置文件中，大部分配置项是注释掉的，可以根据自己的需求取消注释并修改配置项的值。以下介绍几种常见需要修改的配置项。

#### 修改 RPC 服务监听端口

RPC 端口的修改直接配置文件最外层的 `bind-address`配置项

```text
# Bind address to use for the RPC service for backup and restore.
bind-address = ":<port>"
```

{% hint style="info" %}
配置中`":"`为默认格式，不要省略
{% endhint %}

#### 修改 API 监听端口

API 端口的修改，需要修改`http`配置块下的`bind-address`配置项

```text
[http]
  # The bind address used by the HTTP service.
   bind-address = ":<port>"
```

#### 修改数据目录位置

{% hint style="danger" %}
修改数据目录路径必须是已经存在的，并且目录的属主需要修改为 influxdb，权限为 drwxr-xr-x
{% endhint %}

```text
[meta]
  # Where the metadata/raft database is stored
  dir = "/opt/influxdata/meta"
  
[data]
  # The directory where the TSM storage engine stores TSM files.
  dir = "/opt/influxdata/data"

  # The directory where the TSM storage engine stores WAL files.
  wal-dir = "/opt/influxdata/wal"

```

