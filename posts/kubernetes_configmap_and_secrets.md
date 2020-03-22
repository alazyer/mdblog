# Kubernetes Configmap & Secret

[toc]

## 1. 二者异同

Configmap和Secret都是K8S中用来保存Key：value形式键值对的资源类型。

Configmap一般用于保存非敏感数据

Secret主要用于保存敏感类型数据，因为保存的数据会经过base64加密，被认为是安全的（比cm原生字符串安全）``

另外Secret依据存储的数据格式不同，可以分为多种类型：

- Opaque：与CM一样就是简单的key：value，但是value是经过base64加密的，一般用作密码保存
- kubernetes.io/.dockerconfigjson: 存储docker镜像用户密码信息

## 2. 使用限制

configmap和secret中存储数据有大小，由于存在[etcd限制](https://github.com/kubernetes/kubernetes/issues/19781),因而最多报错1MB数据

文件挂载的话，会清楚容器中目标位置原有文件

## 3. Pod中使用场景

### 3.1 数据准备

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO
---
apiVersion: v1
kind: Secret
metadata:
  name: dbsecret
  namespace: default
type: Opaque
data:
  # admin
  username: YWRtaW4K
  # password
  password: cGFzc3dvcmQK
```

### 3.2 环境变量引用

环境变量引用的话，有两个方式：

- 通过valueFrom引用configmap中某个key的值作为环境变量的值，这时环境变量的名字是通过有env.name定义的
- 通过envFrom将configmap的key，value键值对引用为环境变量。这时环境变量的名字就是configmap中的key

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        # Define the environment variable
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: env-config
              key: log_level
      # Use all variables defined in configmap
      envFrom:
        - configMapRef:
            name: special-config
  restartPolicy: Never
```

上面这种定义的话，`test-container`就会有三个环境变量：

- `LOG_LEVEL`: 值来自`env-config`中`log_level`的值，具体的值为`INFO`
- `SPECIAL_LEVEL`: key来自`special-config`，值为其在cm中的值，具体的值为`very`
- `SPECIAL_TYPE`: 类似`SPECIAL_LEVEL`，具体的值为`charm`

### 3.3 文件挂载

#### 3.3.1 使用方式

需要先根据cm或者secret声明一个pod volume，然后在pod的container中声明VolumeMount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
      - name: db-credentail-volume
        mountPath: /etc/db
      - name: env-volume
        mountPath: /var/log
  volumes:
    - name: config-volume
      configMap:
        # Provide the name of the ConfigMap containing the files you want
        # to add to the container
        name: special-config
    - name: env-volume
      configMap:
        name: env-config
        items:
        - key: LOG_LEVEL
          path: level
    - name: db-credential-volume
      secret:
        secretName: dbsecret
  restartPolicy: Never
```

上面这种定义的话，`test-container`就会有五个文件挂载：

- `/var/log/level`: 文件内容为`INFO`

- `/etc/config/SPECIAL_LEVEL`: 文件内容为`very`
- `/etc/config/SPECIAL_TYPE`: 文件内容为`charm`
- `/etc/db/username`: 文件内容为`admin`
- `/etc/db/password`: 文件内容为`password`

另外一个比较特殊的是使用`subPath`方式进行挂载。这个挂载适用于同一个volume被多个container挂载到不同的path下

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
  volumes:
    - name: site-data
      configMap:
        # Provide the name of the ConfigMap containing the files you want
        # to add to the container
        name: special-config
```

上面这种定义的话，`mysql`容器就会有两个文件挂载：

- `/var/lib/mysql/mysql/`: 文件内容为`INFO`

- `/var/lib/mysql/mysql/SPECIAL_LEVEL`: 文件内容为`very`
- `/var/lib/mysql/mysql/SPECIAL_TYPE`: 文件内容为`charm`

而`php`容器也会有两个文件挂载:

- `/var/www/html/SPECIAL_LEVEL`: 文件内容为`very`

- `/var/www/html/SPECIAL_TYPE`: 文件内容为`charm`

### 3.3.2镜像拉取凭证

在container中使用的镜像是非公有镜像源时，需要指明`imagePullSecrets`以实现从成功私有镜像源中拉取镜像的目的。

```bash
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  imagePullSecrets:
  - name: regcred
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        # Define the environment variable
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: env-config
              key: log_level
      # Use all variables defined in configmap
      envFrom:
        - configMapRef:
            name: special-config
```

## 4. 常见问题

4.1 关于`subPath`

- 使用subpath的话，不会自动更新

- 不使用subpath的话，会过一段时间自动更新。更新间隔参见下面引用

> When a ConfigMap already being consumed in a volume is updated, projected keys are eventually updated as well. Kubelet is checking whether the mounted ConfigMap is fresh on every periodic sync. However, it is using its local ttl-based cache for getting the current value of the ConfigMap. As a result, the total delay from the moment when the ConfigMap is updated to the moment when new keys are projected to the pod can be as long as **kubelet sync period** (1 minute by default) + **ttl of ConfigMaps cache** (1 minute by default) in kubelet. You can trigger an immediate refresh by updating one of the pod’s annotations.