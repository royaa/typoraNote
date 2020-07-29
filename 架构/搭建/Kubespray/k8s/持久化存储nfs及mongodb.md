> https://www.cnblogs.com/panwenbin-logs/p/12196286.html
>
> https://blog.csdn.net/dkfajsldfsdfsd/article/details/81319735

```shell
#service端：nfs安装配置完成后，检测配置是否正确
exportfs -a
# client端验证
showmount -e <NFS server name>
```



# 创建命名空间

```shell
vim namespace.yml

---
apiVersion: v1
kind: Namespace
metadata:
  name: mongo
  labels:
    name: mongo
```



# 创建权限

```shell
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: mongo

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-client-provisioner-cluster-role
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nfs-client-provisioner-cluster-role-binding
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: mongo
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-cluster-role
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: nfs-client-provisioner-role
  namespace: mongo
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nfs-client-provisioner-role-binding
  namespace: mongo
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: mongo
roleRef:
  kind: Role
  name: nfs-client-provisioner-role
  apiGroup: rbac.authorization.k8s.io

```

# 启动nfs provisioner

# 启动nfs provisioner

```shell
vim nfs-provisioner.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  namespace:
    mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes   # 此处不能更改，这是镜像默认生成pv地址，当时改了这块，死活不生效，pvc pv又是创建成功，查了好久
          env:
            - name: PROVISIONER_NAME
              value: qgg-nfs-storage
            - name: NFS_SERVER
              value: 10.200.120.240
            - name: NFS_PATH
              value: /data/nfs
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.200.120.240
            path: /data/nfs
```



```shell
# 获取对应的pod
kubectl get pod | grep nfs-client-provisioner
nfs-client-provisioner-68b78f9b59-jvkgq   1/1     Running     0          31m
# 进pod执行
kubectl exec -it nfs-client-provisioner-68b78f9b59-jvkgq -- df
10.200.120.240:/data/nfs 29624320   5813248  23811072  20% /persistentvolumes
```



# 创建storage class

```shell
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
  namespace: mongo
provisioner: qgg-nfs-storage # 此处要与上面deployment env中 PROVISIONER_NAME 保持一致 
parameters:
  archiveOnDelete: "false"
```



# 创建pvc

```shell
vim test-claim.yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  namespace: mongo
  #annotations:
  #  volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  storageClassName: managed-nfs-storage   # 以前是annotations的方式，现在是这种方式，貌似以前的还可以使用
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```



```shell
# 查看pvc
kubectl get pvc

test-claim   Bound    pvc-75142613-c16e-419f-8b4e-61286e1ec13a   1Mi        RWX            managed-nfs-storage   32m

# 查看自动生成的pv, 取上面生成的volume查询  
kubectl get pv | grep pvc-75142613-c16e-419f-8b4e-61286e1ec13a

pvc-75142613-c16e-419f-8b4e-61286e1ec13a   1Mi        RWX            Delete           Bound    mongo/test-claim   managed-nfs-storage            31m

# 查看pv目录是否生成 ： [命名空间]-[pvc名称]-pv实例
find /data/nfs/ -name *pvc-75142613-c16e-419f-8b4e-61286e1ec13a

/data/nfs/mongo-test-claim-pvc-75142613-c16e-419f-8b4e-61286e1ec13a
```



# 写个pod测试

```shell
vim test-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-pod
      image: busybox
      command:
        - "/bin/sh"
      args:
        - "-c"
        - "touch /mnt/SUCCESS && exit 0 || exit 1"
      volumeMounts:
        - name: nfs-pvc
          mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim
```



```shell
# 查看pod是否执行成功
kubectl get pod test-pod
test-pod                                  0/1     Completed   0          13m
# 查看是否生成SUCCESS
ll /data/nfs/mongo-test-claim-pvc-75142613-c16e-419f-8b4e-61286e1ec13a/
```



# 创建 mongodb headlessservice

```shell
vim headlessservice.yml

apiVersion: v1
kind: Service
metadata:
  name: mongo
  namespace: mongo
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
```



# 启动mongodb

```shell
vim statefulset.yml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
  namespace: mongo
spec:
  serviceName: "mongo"
  selector:
    matchLabels:
      role: mongo
  replicas: 3
  template:
    metadata:
      labels:
        role: mongo
        environment: prod
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: mongo
          image: mongo:3.4
          command:
            - mongod
            - "--replSet"
            - rs0
            - "--bind_ip"
            - 0.0.0.0
            - "--smallfiles"
            - "--noprealloc"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-persistent-storage
              mountPath: /data/db
        - name: mongo-sidecar
          image: cvallance/mongo-k8s-sidecar
          env:
            - name: MONGO_SIDECAR_POD_LABELS
              value: "role=mongo,environment=prod"
  volumeClaimTemplates:
  - metadata:
      name: mongo-persistent-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: managed-nfs-storage
      resources:
        requests:
          storage: 2Gi
```



查看集群发现是不可用状态

```shell
kubectl exec -it mongo-0 -n mongo -- mongo

Defaulting container name to mongo.
Use 'kubectl describe pod/mongo-0 -n mongo' to see all of the containers in this pod.
MongoDB shell version v3.4.24
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.4.24
Server has startup warnings:
2020-07-13T09:58:08.943+0000 I CONTROL  [initandlisten]
2020-07-13T09:58:08.943+0000 I CONTROL  [initandlisten] ** WARNING: Access control is not enabled for the database.
2020-07-13T09:58:08.943+0000 I CONTROL  [initandlisten] **          Read and write access to data and configuration is unrestricted.
2020-07-13T09:58:08.943+0000 I CONTROL  [initandlisten] ** WARNING: You are running this process as the root user, which is not recommended.
2020-07-13T09:58:08.943+0000 I CONTROL  [initandlisten]
2020-07-13T09:58:08.944+0000 I CONTROL  [initandlisten]
2020-07-13T09:58:08.944+0000 I CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/enabled is 'always'.
2020-07-13T09:58:08.944+0000 I CONTROL  [initandlisten] **        We suggest setting it to 'never'
2020-07-13T09:58:08.944+0000 I CONTROL  [initandlisten]
2020-07-13T09:58:08.944+0000 I CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/defrag is 'always'.
2020-07-13T09:58:08.944+0000 I CONTROL  [initandlisten] **        We suggest setting it to 'never'
2020-07-13T09:58:08.944+0000 I CONTROL  [initandlisten]
rs0:SECONDARY> rs.status()
{
        "info" : "run rs.initiate(...) if not yet done for the set",
        "ok" : 0,
        "errmsg" : "no replset config has been received",
        "code" : 94,
        "codeName" : "NotYetInitialized"
}
> 
```



应该是`mongo k8s sidecar`没有正确的配置，查看其日志：



```shell
kubectl logs mongo-0 mongo-sidecar -n mongo

Error in workloop { [Error: [object Object]]
  message:
   { kind: 'Status',
     apiVersion: 'v1',
     metadata: {},
     status: 'Failure',
     message:
      'pods is forbidden: User "system:serviceaccount:mongo:default" cannot list resource "pods" in API group "" at the cluster scope',
     reason: 'Forbidden',
     details: { kind: 'pods' },
     code: 403 },
  statusCode: 403 }
```

信息显示默认分配的sa账号没有list此namespace下pods的权限, 需要给默认的sa账号提权，增加list pods的权限

# 授权mongo

```shell
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mongo
  name: mongo-pod-read
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: mongo-pod-read
  namespace: mongo
subjects:
- kind: ServiceAccount
  name: default
  namespace: mongo
roleRef:
  kind: Role
  name: mongo-pod-read
  apiGroup: rbac.authorization.k8s.io
```



仍然报错， 给此sa更大的权限，这里使用默认的clusterrole view权限进行赋权



```shell
vim rbac.yml

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: mongo-default-view
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    name: default
    namespace: mongo
```



pod在创建后，是无法更换sa账号与sa权限的，所以需要重建pod



```shell
# 查看statefulset
kubectl get statefulset -n mongo
NAME    READY   AGE
mongo   3/3     28m

# scale 将副本置0
kubectl scale statefulset mongo -n mongo --replicas=0
statefulset.apps/mongo scaled

# scale 将副本置3 重建
kubectl scale statefulset mongo -n mongo --replicas=3
statefulset.apps/mongo scaled

# 查看所有
kubectl get all

NAME                                          READY   STATUS      RESTARTS   AGE
pod/mongo-0                                   2/2     Running     0          15m
pod/mongo-1                                   2/2     Running     0          15m
pod/mongo-2                                   2/2     Running     0          15m

NAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
service/mongo   ClusterIP   None         <none>        27017/TCP   63m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-client-provisioner   1/1     1            1           3h51m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/nfs-client-provisioner-68b78f9b59   1         1         1       3h51m

NAME                     READY   AGE
statefulset.apps/mongo   3/3     30m

# 查看集群状态，发现状态已经正常
kubectl exec -it mongo-0 -n mongo -- mongo
rs0:PRIMARY> rs.status()

# 扩容
kubectl scale statefulset mongo --replicas=4 -n mongo

# 使用/访问
mongo cluster访问默认连接为：

mongodb://mongo1,mongo2,mongo3:27017/dbname_?
在kubernetes中最常用的FQDN连接服务的连接为：

#appName.$HeadlessServiceName.$Namespace.svc.cluster.local
因为我们采用statefulset部署的pod，所以命名均有规则，所以实际上如果连接4副本的mongodb cluster，上面的默认连接该为（默认为namespace之外）：

mongodb://mongo-0.mongo.mongo.svc.cluster.local:27017,mongo-1.mongo.mongo.svc.cluster.local:27017,mongo-2.mongo.mongo.svc.cluster.local:27017,mongo-3.mongo.mongo.svc.cluster.local:27017/?replicaSet=rs0
```

# 暴露外网访问

```shell
vim /etc/kubernetes/addons/ingress_nginx/cm-tcp-services.yml

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
data:
  27017: "mongo/mongo:27017"
```

