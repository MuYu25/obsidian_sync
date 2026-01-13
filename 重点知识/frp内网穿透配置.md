# server
fprs.toml
```toml
bindPort = 7000 # 程序运行进程 
vhostHTTPPort = 7100    # http的穿透地址
vhostHTTPSPort = 7200   # https
transport.tcpMux = true  # tcp多路复用
# auth.method = "token" 
auth.token = "Abc12345"  # 连接mi'yao
webServer.port = 7300 
	webServer.addr = "xxx.xxx.xxx.xxx"  # 公网ip地址
webServer.user = "MuYu" 
webServer.password = "Abc12345"
```