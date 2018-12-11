## 常用命令
### 查看 nvidia gpu 信息
```
[root@gpu2]:~# nvidia-smi
Sun Dec  9 00:18:27 2018
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 384.98                 Driver Version: 384.98                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GT 1030     Off  | 00000000:01:00.0 Off |                  N/A |
| 32%   30C    P8   ERR! /  30W |      0MiB /  1998MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

Fan: N/A是风扇转速，从0到100%之间变动，这个速度是计算机期望的风扇转速，实际情况下如果风扇堵转，可能打不到显示的转速。有的设备不会返回转速，因为它不依赖风扇冷却而是通过其他外设保持低温（比如我们实验室的服务器是常年放在空调房间里的）。 

Temp: 是温度，单位摄氏度。 

Perf: 是性能状态，从P0到P12，P0表示最大性能，P12表示状态最小性能。

Pwr: 能耗，上方的Persistence-M：是持续模式的状态，持续模式虽然耗能大，但是在新的GPU应用启动时，花费的时间更少，这里显示的是off的状态。

Bus-Id: 是涉及GPU总线的东西，domain:bus:device.function 

Display Active: 表示GPU的显示是否初始化。 

Memory Usage: 显存使用率。 

GPU-Util: GPU利用率。 

Compute M: 计算模式。 

Processes: 每个进程占用的显存使用率。

显卡是由GPU和显存等组成的，显存和GPU的关系有点类似于内存和CPU的关系. 显存可以看成是空间，类似于内存。显存用于存放模型，数据
显存越大，所能运行的网络也就越大。GPU计算单元类似于CPU中的核，用来进行数值计算。衡量计算量的单位是flop： the number of floating-point multiplication-adds，浮点数先乘后加算一个flop。计算能力越强大，速度越快。衡量计算能力的单位是flops: 每秒能执行的flop数量。

### 周期查看 gpu 监控信息
```bash
watch -n 1 nvidia-smi
```

### python 工具查看 gpu 信息
```bash
pip install gpustat
```

## k8s nvidia GPU device plugin
### 依赖 nvidia GPU runtime

可以在有 gpu 的节点上使用 nvidia GPU runtime
```bash
[root@gpu2]:~# cat /etc/docker/daemon.json
{
  "storage-driver": "overlay",
  "log-driver": "json-file",
  "default-ulimit": [
    "nofile=655350"
   ],
  "log-opts": {
    "max-file": "2",
    "max-size": "64m"
    },
  "default-runtime": "nvidia",
  "runtimes": {
      "nvidia": {
          "path": "/usr/bin/nvidia-container-runtime",
          "runtimeArgs": []
      }
  },
  "debug": false
}
```

### daemonset 部署
需要通过 hostpath 将容器中 device-plugin 挂到节点上，可以通过 nodeSelector 在特定的 label 的节点允许 device plugin
```yaml
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  template:
    metadata:
      creationTimestamp: null
      labels:
        name: nvidia-device-plugin-ds
    spec:
      containers:
      - image: test.caicloudprivatetest.com/release/nvidia-k8s-device-plugin:1.10
        imagePullPolicy: IfNotPresent
        name: nvidia-device-plugin-ctr
        resources: {}
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/lib/kubelet/device-plugins
          name: device-plugin
      dnsPolicy: ClusterFirst
      nodeSelector:
        caicloud.io/nvidia-gpu: "true"
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - hostPath:
          path: /var/lib/kubelet/device-plugins
          type: ""
        name: device-plugin
```
