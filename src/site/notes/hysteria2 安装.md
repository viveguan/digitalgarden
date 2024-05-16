---
{"dg-publish":true,"permalink":"/hysteria2 安装/"}
---

#### 安装或升级 
> bash <(curl -fsSL https://get.hy2.sh/) 
#### 移除 
> bash <(curl -fsSL https://get.hy2.sh/) --remove
#### 常用命令
> systemctl enable hysteria-server # 开机自启 
> systemctl status hysteria-server # 查看状态
> systemctl restart hysteria-server # 重启服务

---
#### 配置 Hyteria2

``` yaml
# 配置文件地址:vi /etc/hysteria/config.yaml    例子如下： 端口/密码/伪装 建议自己按需求修改

listen: :30000 # 监听端口，注意30000前面的:不能少
auth:
  type: password
  password: 7o4Xe0hh0eS2UniGyre
masquerade: # 伪装
  type: proxy
  proxy:
    url: https://www.bing.com/
    rewriteHost: true
# 证书 (acme 和 tls 二选一), tls自定义证书 应该是最简单的方案
tls: {cert: /etc/hysteria/cert.pem, key: /etc/hysteria/private.key} # 自定义证书
# acme: {domains: [yourDdns.com], email: yourEmail@gmail.com} # 使用acme.sh自动申请证书
```
这里给一个我真实配置的：
``` yaml
listen: :28443

#acme:
#  domains:
#    - your.domain.net
#  email: your@email.com

auth:
  type: password
  password: ihgYmbFFnznIMq+mMSZ

masquerade:
  type: proxy
  proxy:
    url: https://www.bing.com/
    rewriteHost: true
outbounds:
  - name: v4_only
    type: direct
    direct:
      mode: 4
  - name: v6_only
    type: direct
    direct:
      mode: 6
acl:
  inline:
    - reject(suffix:testcn.com)
    - v4_only(all)
tls: {cert: /etc/hysteria/cert.pem, key: /etc/hysteria/private.key}
```

生成自定义证书并重启, 这里拿 bing.com 举例：
``` shell
openssl req -x509 -nodes -newkey ec:<(openssl ecparam -name prime256v1) -keyout /etc/hysteria/private.key -out /etc/hysteria/cert.pem -subj "/CN=bing.com" -days 36500
chmod 777 /etc/hysteria/private.key
systemctl restart hysteria-server
```

客户端配置：clash为例：
``` yaml
- {name: VPS1, server: x.x.x.x, type: hysteria2, port: 30000, password: 7o4Xe0hh0eS2UniGyre, sni: bing.com, skip-cert-verify: true, down: "150", up: "30"}
```

参考文档：
[hysteria2 使用介绍 - VPS.Dance](https://vps.dance/hysteria2.html)

