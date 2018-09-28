# DNS 集群

## 主节点 （10.0.7.1）
> DNS主节点配置

### 0x00 安装bind

`yum -y install bind-libs bind-utils bind bind-chroot`

### 0x01 配置/etc/named.conf

```
options {
    listen-on port 53 { any; };
    directory     "/var/named";
    dump-file     "data/cache_dump.db";
    zone-statistics yes;
    statistics-file "data/named_stats.txt";
    memstatistics-file "data/named_mem_stats.txt";
    allow-query     { any; };
    // recursion:
    // 作为授权域名服务器 no，作为一个递归DNS服务器 yes;
    recursion yes;
    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
    empty-zones-enable no;
};

logging {
  // 错误日志
  channel error_log {
    file "data/error_log" versions 10 size 10m;
    severity warning;
    print-category yes;
    print-severity yes;
    print-time yes;
};
  // 查询日志
  channel query_log {
    file "data/query_log" versions 10 size 100m;
    severity info;
    print-category yes;
    print-severity yes;
    print-time yes;
  };
  // 错误日志
  category default {
    error_log;
  };
  // 查询日志
  category queries {
    query_log;
  };
};

// 引入rfc1912.zones 和 root.key
include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

### 0x02 配置/etc/named.rfc1912.zones

```
// 正向解析
zone "xiaohou.com" IN {
        type master;
        file "xiaohou.com.zone";
        allow-update { none; };
        };
// 反向解析
zone "7.0.10.in-addr.arpa" IN {
    type master;
    file "10.0.7.arpa.zone";
    allow-update { none; };
};
```

### 0x03 配置正向区域/var/named/xiaohou.com.zone

```
$TTL 86400
;资源记录简写时补全的后缀,如果不指定默认为zone中定义的
$ORIGIN xiaohou.com.
@   IN  SOA     ns1.xiaohou.com. root.xiaohou.com. (
        2017031901  ;Serial
        3600        ;Refresh
        1800        ;Retry
        604800      ;Expire
        86400       ;Minimum TTL
)
        IN  NS      ns1.xiaohou.com.
        IN  NS      ns2.xiaohou.com.
        IN  MX  10  mx1
        IN  MX  20  mx2
ns1     IN  A   10.0.7.1
ns2     IN  A   10.0.7.2
@       IN  A   10.0.7.1
www     IN  A   10.0.7.2
mx1     IN  A   10.0.7.3
mx2     IN  A   10.0.7.4
```

### 0x04 配置反向区域/var/named/10.0.7.arpa.zone

```
$TTL 1D
@       IN SOA  ns1.xiaohou.com.  root.xiaohou.com. (
                    11      ; serial
                    1H      ; refresh
                    5M      ; retry
                    3M      ; expire
                    10H )   ; minimum
          IN NS   ns1.xiaohou.com.
          IN NS   ns2.xiaohou.com.
1         IN PTR  ns1.xiaohou.com.
2         IN PTR  ns2.xiaohou.com.
3         IN PTR  mx1.xiaohou.com.
4         IN PTR  mx2.xiaohou.com.
````

### 0x05 启动

```
systemctl start named
systemctl enable named
```

**如果需要使用chroot启动**

/usr/libexec/setup-named-chroot.sh  /var/named/chroot/ on
systemctl stop named
systemctl disable named
systemctl start named-chroot
systemctl enable named-chroot

## 从节点 （10.0.7.2）

### 0x00 安装bind

`yum -y install bind-libs bind-utils bind bind-chroot`

### 0x01 配置/etc/named.conf

```
options {
    listen-on port 53 { any; };
    directory     "/var/named";
    dump-file     "data/cache_dump.db";
    statistics-file "data/named_stats.txt";
    memstatistics-file "data/named_mem_stats.txt";
    allow-query     { any; };
    //recursion:
    //作为授权域名服务器 no，作为一个递归DNS服务器 yes;
    recursion yes;
    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
    empty-zones-enable no;
};

logging {
  channel error_log {
    file "data/error_log" versions 10 size 10m;
    severity warning;
    print-category yes;
    print-severity yes;
    print-time yes;
  };
  channel query_log {
    file "data/query_log" versions 10 size 100m;
    severity info;
    print-category yes;
    print-severity yes;
    print-time yes;
  };
  category default {
    error_log;
  };
  category queries {
    query_log;
  };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

### 0x02 配置/etc/named.rfc1912.zones

```
zone "xiaohou.com" IN {
        type slave;
        masters { 10.0.7.1; };
        file "slaves/xiaohou.com.zone";
        };
zone "7.0.10.in-addr.arpa" IN {
        type slave;
        masters { 10.0.7.1; };
        file "slaves/10.0.7.arpa.zone";
};
```

### 0x03 启动

```
systemctl start named
systemctl enable named
```

**如果需要使用chroot启动**

/usr/libexec/setup-named-chroot.sh  /var/named/chroot/ on
systemctl stop named
systemctl disable named
systemctl start named-chroot
systemctl enable named-chroot

## 子域授权 + 配置子域服务器（10.0.7.3）

子域授权，是DNS的分布式的一种实现。**在原有的zone上划出一个子域**，并给新DNS服务器管理。如果有客户端请求解析在此区域中的域名，则只要找新的子DNS服务器，以便减轻主DNS的压力。

### 0x00 配置主服务器

首先在主域服务器添加子域：

`vim /var/named/xiaohou.com.zone`

```
$TTL 86400
$ORIGIN xiaohou.com.
@   IN  SOA     ns1.xiaohou.com. root.xiaohou.com. (
        2017031902  ;Serial
        3600        ;Refresh
        1800        ;Retry
        604800      ;Expire
        86400       ;Minimum TTL
)
        IN  NS      ns1.xiaohou.com.
        IN  NS      ns2.xiaohou.com.
        IN  MX  10  mx1
        IN  MX  20  mx2
ns1     IN  A   10.0.7.1
ns2     IN  A   10.0.7.2
@       IN  A   10.0.7.1
www     IN  A   10.0.7.2
mx1     IN  A   10.0.7.3
mx2     IN  A   10.0.7.4

blog     IN  NS      ns1.blog
ns1.blog IN  A       10.0.7.3
```

**记得修改版本号，保证从服务器获取更新；**

### 0x01 重新载入配置

`rndc reload`

## 继续配置子域服务器（10.0.7.3）

### 0x00 安装bind

`yum -y install bind-libs bind-utils bind bind-chroot`

### 0x01 配置/etc/named.conf

```
options {
    listen-on port 53 { any; };
    directory     "/var/named";
    dump-file     "data/cache_dump.db";
    statistics-file "data/named_stats.txt";
    memstatistics-file "data/named_mem_stats.txt";
    allow-query     { any; };
    //recursion:
    //作为授权域名服务器 no，作为一个递归DNS服务器 yes;
    recursion yes;
    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
    empty-zones-enable no;
};

logging {
  channel error_log {
    file "data/error_log" versions 10 size 10m;
    severity warning;
    print-category yes;
    print-severity yes;
    print-time yes;
  };
  channel query_log {
    file "data/query_log" versions 10 size 100m;
    severity info;
    print-category yes;
    print-severity yes;
    print-time yes;
  };
  category default {
    error_log;
  };
  category queries {
    query_log;
  };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

### 0x02 配置/etc/named.rfc1912.zones

```
zone "blog.xiaohou.com" IN {
        type master;
        file "blog.xiaohou.com.zone";
        allow-update { none; };
        };
```

### 0x03 配置/var/named/blog.xiaohou.com.zone

```
$TTL 1D
$ORIGIN blog.xiaohou.com.
@   IN  SOA     ns1.blog.xiaohou.com. root.blog.xiaohou.com. (
        2017031901  ;Serial
        1H          ;Refresh
        10M         ;Retry
        3D          ;Expire
        1D          ;Minimum TTL
)

          IN  NS    ns1
          IN  NS    ns2
ns1       IN  A   10.0.7.3
ns2       IN  A   10.0.7.4
@         IN  A   10.0.7.5
www       IN  A   10.0.7.6
mx1       IN  A   10.0.7.7
mx2       IN  A   10.0.7.8
```

### 0x04 启动

/usr/libexec/setup-named-chroot.sh  /var/named/chroot/ on
systemctl stop named
systemctl disable named
systemctl start named-chroot
systemctl enable named-chroot
