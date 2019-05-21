# 如何通过JWT Token访问Kubernetes dashboard？

之前安装[Docker for windows](docker-for-windows.md)一文里提到如何安装免身份验证的dashboard，但是更多的场景是我们需要一个身份验证，那么如何安装一个带身份验证的dashboard，并让我们顺利登陆呢？以下是操作过程。

## 安装标准版的dashboard

如果你之前已经安装过免登陆验证版本的dashboard，请先删除，命令如下：

```shell
$ kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.0/src/deploy/alternative/kubernetes-dashboard.yaml
serviceaccount "kubernetes-dashboard" deleted
role.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" deleted
rolebinding.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" deleted
deployment.apps "kubernetes-dashboard" deleted
service "kubernetes-dashboard" deleted
```

删除成功后可以通过以下命令验证是否还有dashboard的pod在运行：

```shell
$ kubectl get pods  --namespace=kube-system
NAME                                         READY     STATUS    RESTARTS   AGE
etcd-docker-for-desktop                      1/1       Running   20         42d
heapster-7ff8d6bf9f-zxgxz                    1/1       Running   10         42d
kube-apiserver-docker-for-desktop            1/1       Running   11         42d
kube-controller-manager-docker-for-desktop   1/1       Running   10         42d
kube-dns-86f4d74b45-nkdft                    3/3       Running   30         42d
kube-proxy-sslt5                             1/1       Running   10         42d
kube-scheduler-docker-for-desktop            1/1       Running   10         42d
monitoring-grafana-68b57d754-xhhsd           1/1       Running   10         42d
monitoring-influxdb-cc95575b9-8dj2m          1/1       Running   10         42d
```

待清除完旧版的dashboard组件后，可以通过如下命令安装最新的标准版dashboard：

```shell
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml
secret "kubernetes-dashboard-certs" created
secret "kubernetes-dashboard-csrf" created
serviceaccount "kubernetes-dashboard" created
role.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
rolebinding.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
deployment.apps "kubernetes-dashboard" created
service "kubernetes-dashboard" created
```

执行完毕后可以通过查看`kube-system`命名空间下的pod检查，发现会多出一个dashboard相关的pod。此时输入命令`kubectl proxy`即可开启dashboard，打开浏览器并输入`http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login`，如果出现正常的登陆页，并提供kubeconfig和令牌登陆选项，则表示dashboard组件工作正常。那么我们怎么通过令牌登陆呢？

大部分情况下，dashboard安装完毕后，组件会为我们创建好相关账号与权限，如上文输出的账号`kubernetes-dashboard`和权限`kubernetes-dashboard-minimal`，我们直接使用即可。使用方法见下文的[获取Bearer Token](#%E8%8E%B7%E5%8F%96bearer-token)。

## 可选：创建管理员账号（ServiceAccount）

我们先使用yaml文件创建一个管理员账号，方法如下：

```shell
$ cat > dashboard-adminuser.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system

$ kubectl apply -f dashboard-adminuser.yaml
serviceaccount "admin-user" created
```

## 可选：绑定管理员账号（ClusterRoleBinding）

大部分情况下，使用`kops`，`kubeadm`等主流工具创建的kubernetes集群，都会包含`ClusterRole`, `admin-Role`，我们可以直接通过`ClusterRoleBinding`将角色与我们刚刚创建的账号绑定起来。

```shell
$ cat > dashboard-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system

$ kubectl apply -f dashboard-rolebinding.yaml
clusterrolebinding.rbac.authorization.k8s.io "admin-user" created
```

### 如何查看现有的Role与ClusterRole？

```shell
$ kubectl get role -n kube-system

$ kubectl get clusterrole -n kube-system
```

## 获取Bearer Token

账号绑定成功后，即可通过以下命令获取到Token。例如我们想要获取dashboard组件创建的`kubernetes-dashboard`账号的token，可以执行如下命令：

> PS：查看其它账号，如`admin-user`，将命令中的`kubernetes-dashboard`替换成`admin-user`即可。

```shell
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep kubernetes-dashboard | awk '{print $1}')
Name:         kubernetes-dashboard-certs
Namespace:    kube-system
Labels:       k8s-app=kubernetes-dashboard
Annotations:
Type:         Opaque

Data
====


Name:         kubernetes-dashboard-csrf
Namespace:    kube-system
Labels:       k8s-app=kubernetes-dashboard
Annotations:
Type:         Opaque

Data
====
csrf:  0 bytes


Name:         kubernetes-dashboard-key-holder
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
priv:  1679 bytes
pub:   459 bytes


Name:         kubernetes-dashboard-token-s89lb
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=kubernetes-dashboard
              kubernetes.io/service-account.uid=7953fee9-7b9f-11e9-8c7c-00155d0ac810

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC10b2tlbi1zODlsYiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6Ijc5NTNmZWU5LTdiOWYtMTFlOS04YzdjLTAwMTU1ZDBhYzgxMCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.zjBAcuZWk9Txdz3NIlAW_dOvNzC6vSg8ZTJSXxlD9z7Q7bAMVFm1s2ka0PCY9aWo3Vq02Gt0cxYEYD9YdHtvk6U-IN8wX24GyryNKpt8rChHXrQh7YKtwAQ40O5FO5yVTku2RrCo5KyW_y1e8HwFvsuC0tvtHsAxax5lvRYLB1BH7No8LpjyR1aGpxn-Tyog8fOezBTRMqibiDth2e6zhDxcqZgmXO-CbnD0K0OSO2PHT4ajdJWpomPbsHxg3Vf4X0oVBm4KC4eXqFwEsRG9EuTAyqkdEkl8de5evz2S9IW0FHk5_kFDfnSMpprn1bK56Vqw_UmyKc6wnO1U4tuP2g
```

将输出部分token的值拷贝出来，该值应该是一个标准的jwt token，在有些console环境下直接拷贝屏幕上因内容过长而换行的值容易出错，譬如出现换行符，或者其他特殊字符，导致jwt token被破坏的情况，这时可以考虑将token输出到文件，然后在文件里将token值拷贝出来。

> PS: token登陆老是失败？打开https://jwt.io，然后将拷贝出来的jwt token贴进去检查下，大部分情况都是token被破坏导致的。

## 验证登陆

打开浏览器并输入`http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login`，在登陆页选择令牌，然后粘贴刚刚拷贝出来的jwt token，完成登陆。

> PS: 登陆后如果出现大量黄色的警告信息，则大部分是因为账号权限不正确导致。**如无必要，建议使用dashboard自带账号即可**。

```txt
configmaps is forbidden: User "system:anonymous" cannot list configmaps in the namespace "default"

persistentvolumeclaims is forbidden: User "system:anonymous" cannot list persistentvolumeclaims in the namespace "default"

secrets is forbidden: User "system:anonymous" cannot list secrets in the namespace "default"

services is forbidden: User "system:anonymous" cannot list services in the namespace "default"

ingresses.extensions is forbidden: User "system:anonymous" cannot list ingresses.extensions in the namespace "default"

daemonsets.apps is forbidden: User "system:anonymous" cannot list daemonsets.apps in the namespace "default"

pods is forbidden: User "system:anonymous" cannot list pods in the namespace "default"

events is forbidden: User "system:anonymous" cannot list events in the namespace "default"

deployments.apps is forbidden: User "system:anonymous" cannot list deployments.apps in the namespace "default"

replicasets.apps is forbidden: User "system:anonymous" cannot list replicasets.apps in the namespace "default"

jobs.batch is forbidden: User "system:anonymous" cannot list jobs.batch in the namespace "default"

cronjobs.batch is forbidden: User "system:anonymous" cannot list cronjobs.batch in the namespace "default"

replicationcontrollers is forbidden: User "system:anonymous" cannot list replicationcontrollers in the namespace "default"

statefulsets.apps is forbidden: User "system:anonymous" cannot list statefulsets.apps in the namespace "default"
```