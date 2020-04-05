# Kubernetes安装local-path-provisioner基于HostPath使用动态PV

#### 获取[local-path-provisioner](https://github.com/rancher/local-path-provisioner)

```sh
git clone https://github.com/rancher/local-path-provisioner.git
```

#### 修改local-path-storage.yaml

```sh
vi local-path-provisioner/deploy/local-path-storage.yaml
```

修改前（修改部分）：

```yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: local-path-config
  namespace: local-path-storage
data:
  config.json: |-
        {
                "nodePathMap":[
                {
                        "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
                        "paths":["/opt/local-path-provisioner"]
                }
                ]
        }


```

修改后（修改部分）：

```yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: local-path-config
  namespace: local-path-storage
data:
  config.json: |-
        {
                "nodePathMap":[
                {
                        "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
                        "paths":["/u01/local-path-provisioner"]
                }
                ]
        }


```

#### 创建文件路径

```sh
mkdir -p /u01/local-path-provisioner
```

```sh
chmod 777 /u01/local-path-provisioner
```

#### 创建namespace

```sh
kubectl create ns local-path-storage
```

#### 发布local-path-storage

```sh
kubectl apply -f local-path-provisioner/deploy/local-path-storage.yaml -n local-path-storage
```

#### 确认发布结果

确认结果：

```sh
kubectl get po -n local-path-storage
```

结果如下：

```
NAME                                      READY   STATUS    RESTARTS   AGE
local-path-provisioner-54bbdbb5cc-7d8kw   1/1     Running   0          18m
```

确认结果：

```sh
kubectl get storageclass
```

结果如下：

```
NAME         PROVISIONER             AGE
local-path   rancher.io/local-path   18m
```

#### 创建PVC和POD进行测试

pvc.yam如下：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-path-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 2Gi
```

pod.yaml如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
  namespace: default
spec:
  containers:
  - name: volume-test
    image: nginx:stable-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: volv
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: volv
    persistentVolumeClaim:
      claimName: local-path-pvc
```

发布PVC和POD：

```sh
kubectl create -f local-path-provisioner/examples/pvc.yaml
```

```sh
kubectl create -f local-path-provisioner/examples/pod.yaml
```

确认结果：

```sh
kubectl get pv
```

结果如下：

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                   STORAGECLASS      REASON   AGE
pvc-adeb9540-5ad8-11ea-9cf7-02001700ae85   2Gi        RWO            Delete           Bound       default/local-path-pvc                  local-path                 18m
```

确认结果：

```sh
kubectl get pvc
```

结果如下：

```
[opc@k8sinstance workspace]$ kubectl get pvc -n default
NAME                        STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
local-path-pvc              Bound     pvc-adeb9540-5ad8-11ea-9cf7-02001700ae85   2Gi        RWO            local-path     21m
```

删除测试的PVC和POD：

```sh
kubectl delete -f local-path-provisioner/examples/pod.yaml
```

```sh
kubectl delete -f local-path-provisioner/examples/pvc.yaml
```

#### 设置为default的storageclass

```sh
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

```sh
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.beta.kubernetes.io/is-default-class":"true"}}}'
```

#### 卸载

```sh
kubectl delete -f local-path-provisioner/deploy/local-path-storage.yaml
```

完结！


