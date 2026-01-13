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
serverAddr = "120.24.184.83"
serverPort = 7000
auth.token = "Abc12345"
transport.tcpMux = true

[[proxies]]
name = "test-tcp"
type = "tcp"
# localIP = "127.0.0.1"
localPort = 8000
remotePort = 7001
# customDomains = ['www']
```