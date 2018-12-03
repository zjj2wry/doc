## Tips

### quick debug API

hodor 支持 oidc 和 basic 认证方式，使用 basic 认证方式 debug 你的 API 
```go
package main

import (
	"encoding/base64"
	"fmt"
)

func main() {
	s := base64.StdEncoding.EncodeToString([]byte("admin:Pwd123456"))
	fmt.Println(s)
}

// output: YWRtaW46UHdkMTIzNDU2
```
使用 token 查看个人信息
```bash
➜  ~ curl -H 'Authorization: basic YWRtaW46UHdkMTIzNDU2' 192.168.22.160:30069/apis/cauth.auth.caicloud.io/v2/profile
{
  "lastModified": "2018-10-26T04:10:53Z",
  "createTime": "2018-10-26T04:10:53Z",
  "deleteTime": null,
  "username": "admin",
  "nickname": "admin",
  "email": "admin@caicloud.io",
  "emailVerified": true,
  "invalid": false,
  "role": "owner",
  "passwordReset": ""
 }
```

### quick switch cluster

kubectl config 命令支持配置多集群环境和快速切换至不同集群的配置

1.下载集群中的 kubeconfig 文件

```bash
scp root@192.168.17.32:/etc/kubernetes/kubectl.kubeconfig 17.32
scp root@192.168.22.160:/etc/kubernetes/kubectl.kubeconfig 22.160
```

2.修改 clusters、users、contexts 信息，因为不同集群里 clusters、users 的 key 是相同的，不修改合并 kubeconfig 的时候会冲突。另外 context 取个容易记的名称，比如下面的集群是 resource 组的，可以把 context 取名为 resource 或者集群 server 的地址 17.32

```bash
 apiVersion: v1
 kind: Config
 clusters
 - cluster:
     server: "https://192.168.21.34:6443"
-  name: control-res-auth
+  name: resource
 contexts:
 - context:
-    cluster: control-res-auth
-    user: kubectl
-  name: default
+    cluster: resource
+    user: resource
+  name: resource
 users:
-- name: kubectl
+- name: resource
   user:
     token: 9DSl5KqJEHDsmBweiC3mIiFDHd3K9VEO
```

3.合并 kubeconfig

```bash
KUBECONFIG=22.160:17.32 kubectl config view --raw> $HOME/.kube/config
```

4.切换集群

```bash
➜  ~ kubectl config get-contexts
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
          clever     clever     clever
          resource   resource   resource
➜  ~ kubectl config use-context clever
Switched to context "clever".
➜  ~ kubectl cluster-info
Kubernetes master is running at https://192.168.21.161:6443
```
