# pod (2) - 

## 学习目的：
> 理解Secret等核心projected volume如何使用，从而理解在应用中如何注入配置   
> 理解liveness等常见的k8s设计机制(都是套路)

## 链接：https://time.geekbang.org/column/article/40466

## Task1 : 构造一个secret 对象并找到其内容

假设需要构造一个name是mysecret账户，用户名为root，密码为123456

### Step 1.1: 生成base64的密钥
```
echo -n 'root' |base64
# cm9vdA==
echo -n '123456' |base64
# MTIzNDU2
```

### Step 1.2: 生成secret的yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  user: cm9vdA==
  pass: MTIzNDU2
```

存储为 `my-secret.yaml`

### Step 1.3: 生成secret对象
```
# kubectl delete -f my-secret.yaml
kubectl create -f my-secret.yaml
kubectl get secret
kubectl describe secrets/mysecret
```
应该能看到这个实体

### Step 1.4: 在pod中使用
pod文件参见 my-pod.yaml
其中注意
> 使用secret的, 目录挂在 /mypod-vol 下
> secret使用了name做查找 name: mysecret

```
kubectl delete -f ./my-pod.yaml
kubectl create -f ./my-pod.yaml
kubectl describe pods my-pod

```

### step 1.5：验证： 在容器中找到这个secret 
```
kubectl describe pod my-pod

kubectl attach -it my-pod -c my-container

/ # ls /mypod-vol
pass  user
/ # cat /mypod-vol/pass 
123456
/ # cat /mypod-vol/user 
root 
```

## Task 2: 自定义livenessProbe （健康检查）保活关键进程
创建 pod yaml文件： liveness-pod.yaml

```
kubectl delete -f ./liveness-pod.yaml
kubectl create -f ./liveness-pod.yaml
# sleep for a while
kubectl describe pods test-liveness-exec
# sleep for a while for restart
```

**注意到events中出现了 Unhealthy/Pulled/Created/Killing/Started 序列，说明被重启成功**

FirstSeen LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
2s        2s      1   {kubelet worker0}   spec.containers{liveness}   Warning     Unhealthy   Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory

## task 3: Pod 运行起来之后，我们查看一下这个 Pod 的 API 对象信息

```
$ kubectl create -f pod.yaml

$ kubectl get pod website -o yaml
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
  annotations:
    podpreset.admission.kubernetes.io/podpreset-allow-database: "resource version"
spec:
  containers:
    - name: website
      image: nginx
      volumeMounts:
        - mountPath: /cache
          name: cache-volume
      ports:
        - containerPort: 80
      env:
        - name: DB_PORT
          value: "6379"
  volumes:
    - name: cache-volume
      emptyDir: {}
```      
