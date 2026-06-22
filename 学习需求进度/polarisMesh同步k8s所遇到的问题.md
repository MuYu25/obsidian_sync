- [x] polarisMesh中没有修改的`polaris-server`地址是有问题的，需要自己手动设置ip 才行
- [x] 已经排查`polaris-controller`的日志，修改polaris-controller的配置，现在没有请求地址错误的信息
- [x] 自动同步`controller`信息至`polarid-server`出现错误,26-06-21日23：17记，明日解决
```bash
      2026-06-21T23:12:25.957497Z	error	app/polaris-controller-manager.go:573	Sync: Failed to get controller ip address, dial ip4:1: lookup 192.168.1.4:8091: no such host
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x1946a52]

goroutine 141 [running]:
github.com/polarismesh/polaris-controller/cmd/polaris-controller/app.registerController(0xc0007aa1b0, {0x2186c28, 0xc0007b70c0})
	/home/runner/work/polaris-controller/polaris-controller/cmd/polaris-controller/app/polaris-controller-manager.go:575 +0x172
github.com/polarismesh/polaris-controller/cmd/polaris-controller/app.startPolarisController({{0x2182cd8, 0xc00042e6c0}, {0x2199178, 0xc0003825b0}, {0x2168a40, 0xc0003aef00}, {{0x50, {0x0, 0x0}, {0xdf8475800}, ...}, ...}, ...})
	/home/runner/work/polaris-controller/polaris-controller/cmd/polaris-controller/app/polaris-controller-manager.go:543 +0x3d2
github.com/polarismesh/polaris-controller/cmd/polaris-controller/app.StartControllers({{0x2182cd8, 0xc00042e6c0}, {0x2199178, 0xc0003825b0}, {0x2168a40, 0xc0003aef00}, {{0x50, {0x0, 0x0}, {0xdf8475800}, ...}, ...}, ...}, ...)
	/home/runner/work/polaris-controller/polaris-controller/cmd/polaris-controller/app/polaris-controller-manager.go:455 +0x1f5
github.com/polarismesh/polaris-controller/cmd/polaris-controller/app.Run.func1({0x2182f40?, 0xc0003a4af0?})
	/home/runner/work/polaris-controller/polaris-controller/cmd/polaris-controller/app/polaris-controller-manager.go:318 +0x249
created by k8s.io/client-go/tools/leaderelection.(*LeaderElector).Run in goroutine 1
	/home/runner/go/pkg/mod/k8s.io/client-go@v0.27.3/tools/leaderelection/leaderelection.go:208 +0xf6
```
上诉问题解决一半，polaris-controller状态已经同步到polaris-console中
- [ ]  `controller`同步pod状态到`polaris-console`,出现错误，26-06-22 23：08记，明日解决
    ```bash
    W0622 22:57:29.161545       1 warnings.go:70] v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
W0622 22:57:29.165533       1 warnings.go:70] v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
2026-06-22T22:57:30.113016Z	error	polarisapi/service_api.go:754	Failed to lookup access token {
 "code": 401001,
 "info": "access is not approved",
 "client": null,
 "namespace": null,
 "service": null,
 "instance": null,
 "routing": null,
 "alias": null,
 "rateLimit": null,
 "circuitBreaker": null,
 "configRelease": null,
 "user": {
  "id": "65e4789a6d5b49669adf1e9e8387549c",
  "name": null,
  "password": null,
  "owner": null,
  "source": null,
  "auth_token": null,
  "token_enable": null,
  "comment": null,
  "ctime": null,
  "mtime": null,
  "user_type": null,
  "mobile": null,
  "email": null
 },
 "userGroup": null,
 "authStrategy": null,
 "relation": null,
 "loginResponse": null,
 "modifyAuthStrategy": null,
 "modifyUserGroup": null,
 "resources": null,
 "optionSwitch": null,
 "instanceLabels": null,
 "data": null,
 "serviceContract": null
}
    ```