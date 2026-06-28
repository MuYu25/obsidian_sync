# 异常情况
`top P`中的表现, `cpu`温度常驻`90`度
![](assets/6-28日polaris-controller%20cpu占用异常高问题/file-20260628222702675.png)
`htop`中表现为CPU单核占用非常高，其他摸鱼

# 排查情况
##  查看Pod的CPU配额限制
### 输入
```bash
kubectl exec -it polaris-controller-polaris-controller-0 -n polaris-system -- cat /sys/fs/cgroup/cpu.max
```
### 输出
```bash
200000 100000
```
### 数据解读
CPU配额：`200000 100000`
- 第一个数字 `200000` 表示配额（quota），单位是微秒
- 第二个数字 `100000` 表示周期（period） 
- **计算CPU核数**：`200000 / 100000 = 2` 核
- **计算**：200000 / 100000 = **2核**
- 结论：Pod被限制最多使用2个CPU核心

## 查看Pod当前实际使用的CPU时间
### 输入
```bash
kubectl exec -it polaris-controller-polaris-controller-0 -n polaris-system -- cat /sys/fs/cgroup/cpu.stat
```

### 输出
```bash
usage_usec 1256304432
user_usec 1252596148
system_usec 3708283
nice_usec 0
core_sched.force_idle_usec 0
nr_periods 27500
nr_throttled 0
throttled_usec 0
nr_bursts 0
burst_usec 0
```

### 数据解读
关注这两行：
- `usage_usec`：累计使用的CPU微秒数
- `nr_periods`：过去的调度周期数
CPU实际使用：`usage_usec 1256304432`
- 这个数字是**累计使用时间**（微秒），需要结合 `nr_periods` 计算平均使用率
- **当前平均CPU使用率** = (usage_usec / 1000000) / (nr_periods * 0.1)  
    = (1256秒) / (27500 * 0.1秒)  
    = 1256 / 2750  
    = **约45.7%**（总CPU使用率，2核满载为200%）
**问题来了**：虽然总使用率只有45.7%（2核中的0.9核），但你观察到单核100%，说明**负载不均匀**。

## 查看Pod被允许使用的CPU核心列表（cgroup v2版）

### 输入
```bash
kubectl exec -it polaris-controller-polaris-controller-0 -n polaris-system -- cat /sys/fs/cgroup/cpuset.cpus.effective
```
### 输出
```bash
0-31
```

### 数据解读
CPU亲和性：`0-31`
- **致命发现**：Pod被允许使用所有32个核心（0-31），**没有绑定到特定核心**
- 这排除了K8s CPU Manager静态绑定的嫌疑