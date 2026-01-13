# server
fprs.toml
```toml
bindPort = 7000 # 程序运行进程 
vhostHTTPPort = 7100    # http的穿透地址
vhostHTTPSPort = 7200   # https
transport.tcpMux = true  # tcp多路复用
# auth.method = "token" 
auth.token = "xxxxxx"  # 连接token
webServer.port = 7300 
webServer.addr = "xxx.xxx.xxx.xxx"  # 公网ip地址
webServer.user = "xxxx" 
webServer.password = "xxxx"
```

# client
frpc.toml
```toml
serverAddr = "xxxxxx"   # 远程ip地址
serverPort = 7000
auth.token = "xxxxx"    # 连接的token
transport.tcpMux = true

[[proxies]]
name = "test-tcp"
type = "tcp"        # 类型，tcp，htttp，https
# localIP = "127.0.0.1"
localPort = 8000    # 本地服务的实际地址
remotePort = 7001   # 远程访问的地址远程ip:{{remotePort}}+path访问本地服务
# customDomains = ['www.xxx.com'] # http穿透需要填写域名
```