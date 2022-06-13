---

title: homelab系列-内网访问
date: 2022-6-13 15:51:16
permalink: access_to_private_network/
tags:
- nas
- homelab
---

这篇说一下homelab相关的内网访问配置，本来想写一些资源管理的内容，提纲列到一半发现存在一些依赖，只好先把网络配置的部分说了。

## 基本需求

在硬件部署完成以后，剩下的步骤其实都可以通过web端进行，而我大部分捣腾这台机器的时间也不是在家里，这样一来就需要能通过外部网络环境访问到家里的局域网。

出于安全考虑，需要尽可能少的暴露端口到公网上。此外，国内的家用宽带常规是不允许提供互联网服务的，暴露homelab上的web服务端口也可能给自己带来不必要的麻烦。

## 方案设计

### 动态域名解析

远程访问需要解决的第一个问题是获取家里宽带的ip。家用宽带的IP每次拨号都会发生变化，且存在固定时间间隔强制断开重新拨号的情况，从便利性上来说这一步也是必要的。不过方案也是现成的，很多路由器或者nas系统都提供了动态域名解析功能，一般是使用品牌商提供的子域名用于解析，直接使用即可。

不过因为我有几个托管在aws上的域名，就索性定义了一个二级域名，用一个定时脚本结合`aws-cli`工具来做更新，脚本是网上找的，因为aws查询IP的服务器地址是在国外，调用时走代理路线导致拿到代理的IP，于是换用了国内的ip查询服务。

```bash
#!/bin/bash

#Variable Declaration - Change These
HOSTED_ZONE_ID=<HOST_ZONE_ID>
NAME=<DOMAIN_NAME>
TYPE="A"
TTL=60

#get current IP address
#IP=$(curl http://checkip.amazonaws.com/)
res=$(curl myip.ipip.net)
res_sub=${res: 6}
IP=${res_sub%% *}
#validate IP address (makes sure Route 53 doesn't get updated with a malformed payload)
if [[ ! $IP =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]]; then
        exit 1
fi

#get current

/usr/local/bin/aws route53 list-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID | \
jq -r '.ResourceRecordSets[] | select (.Name == "'"$NAME"'") | select (.Type == "'"$TYPE"'") | .ResourceRecords[0].Value' > /tmp/current_route53_value

cat /tmp/current_route53_value

#check if IP is different from Route 53
if grep -Fxq "$IP" /tmp/current_route53_value; then
        echo "IP Has Not Changed, Exiting"
        exit 1
fi


echo "IP Changed, Updating Records"

#prepare route 53 payload
cat > /tmp/route53_changes.json << EOF
    {
      "Comment":"Updated From DDNS Shell Script",
      "Changes":[
        {
          "Action":"UPSERT",
          "ResourceRecordSet":{
            "ResourceRecords":[
              {
                "Value":"$IP"
              }
            ],
            "Name":"$NAME",
            "Type":"$TYPE",
            "TTL":$TTL
          }
        }
      ]
    }
EOF

#update records
/usr/local/bin/aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch file:///tmp/route53_changes.json
```

### 代理服务

这里的实现方式和科学上网相似，只要把服务端部署在家里的homelab上即可。部署VPN也是可以的，不过我个人感觉这种方式有点重，安全性上来说我觉得应该相差不大，都算的上是经过考验的。

最终暴露在公网上的端口只有两个，一个是http代理服务端口，另一个是方便登录维护的ssh端口。当然这两者都建议映射到五位数以上的端口上，不是说这样就能确保安全，至少能减少一些无聊的嗅探也算节约一些资源吧……

安全性上，ssh服务建议关闭密码登录，或者是使用`Fail2Ban`这类软件防止恶意爆破。

以上方案适用于家庭网络至少具有公网IP的情况。如果没有的话，则需要使用一些内网穿透的方案，像是frp、ngrok之类。不过这类方案需要一台vps转发，有公网IP的情况下我就不折腾这个浪费流量了。

另外我也准备了zerotier这条备用路线以备不时之需。据说因为是基于udp打洞的方式，在网络质量上由于运营商qos的缘故要比tcp差一些，我实测下来在跨网络运营商的情况下确实差一些，暂时只做备用。

### 反向代理

这一步不是必要的，不过个人觉得做了以后可以一定程度上优化访问体验。

由于在同一台服务器上部署了大量web服务，访问时通过端口区分，所以浏览器上会有很多类似`192.168.x.x:xxxx`这样的地址。一方面不方便自己记忆，另外似乎浏览器的密码管理对这种情况处理的也不太好。因此我加了一步用nginx作为反向的代理，根据不同的域名代理不同的后端服务。因为走的是内网代理路线，这个nginx是不需要暴露端口到公网的，也不存在安全性的问题。

nginx的配置比较简单，为了便于维护，核心的nginx配置我只做了基本的日志配置和引入配置目录

```nginx
http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    server_tokens off;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;
    gzip                on;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;
}
~                            
```

每个服务的反向代理配置分拆为单独的配置文件，内容也很简单，以qbittorrent为例

```nginx
server {
listen       80;
server_name  <qbit_domain>;
location / {
proxy_pass http://<qbit_domain>:8080;
proxy_set_header   Host    $host;
proxy_set_header   X-Real-IP   $remote_addr;
proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

这里需要解决一个小问题就是将各个服务配置的域名都指向homelab使用的局域网ip。域名配置有两种做法，可以在hosts文件或者路由器的dns配置里按需配置，将域名指向homelab使用的局域网ip。这么做比较灵活，域名想配成什么都行。

另一个做法是使用公开域名做泛域名解析。比如配置一条A记录为`*.home.xxxx.com`指向`192.168.1.100`。这个方案的好处是只要配一次就好了，不需要逐个设备配置，也不需要每加一个服务就逐个设备改一遍hosts。缺点就是域名设置上没那么灵活了。

*需要注意一点，我在用openwrt做软路由的时候使用了dnsmasq服务，结果发现域名解析无法指向私网地址，解决办法是在防火墙设置中打开IP动态伪装。*

如果代理路线不能保证通信安全的话，还可以在nginx上配置ssl证书，通过https线路访问，来起到加密通信的作用。

