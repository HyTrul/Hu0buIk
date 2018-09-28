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
