# Kubernetes Security Context 

A security context defines privilege and access control settings for a Pod or Container. Security context settings include - 

* ACL - chown / chmod 
* SELINUX
* Privileged Mode / Unprivileged mode 
* SeCOMP
* AppArmor
* AllowPrivilegeEscalation

The different fields associated with security contexts or Pod Security Policies are as below - 

| Control Aspect | Field Names |
| ------ | ------ |
|Running of privileged containers|	privileged|
|Usage of the root namespaces|	hostPID, hostIPC|
|Usage of host networking and ports|	hostNetwork, hostPorts|
|Usage of volume types|	volumes|
|Usage of the host filesystem|	allowedHostPaths|
|White list of FlexVolume drivers|	allowedFlexVolumes|
|Allocating an FSGroup that owns the pod’s volumes|	fsGroup|
|Requiring the use of a read only root file system	|readOnlyRootFilesystem|
|The user and group IDs of the container	|runAsUser, supplementalGroups|
|Restricting escalation to root privileges	|allowPrivilegeEscalation, defaultAllowPrivilegeEscalation|
|Linux capabilities	|defaultAddCapabilities, requiredDropCapabilities, allowedCapabilities|
|The SELinux context of the container	|seLinux|
|The AppArmor profile used by containers|	annotations|
|The seccomp profile used by containers	|annotations|
|The sysctl profile used by containers|	annotations|



## Demo 1 

A pod without security context 

```
vi simplepod.yaml


apiVersion: v1
kind: Pod
metadata:
  name: simplepod
spec:
  containers:
  - image: busybox
    name: busybox
    args:
    - sleep
    - "3600"

```

Create the pod - 

```
kubectl get pods 
NAME        READY   STATUS    RESTARTS   AGE
simplepod   1/1     Running   0          6s
```

Verify the devices attached to the pod - 

```
kubectl exec simplepod -it -- ls /dev
core             null             shm              termination-log
fd               ptmx             stderr           tty
full             pts              stdin            urandom
mqueue           random           stdout           zero
```
---

## Demo 2 

Create a pod with privileges at your podspec

```
vi privilegedpod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: privilegedpod
spec:
  containers:
  - image: busybox
    name: busybox
    args:
    - sleep
    - "3600"
    securityContext:
      privileged: true


kubectl create -f privilegedpod.yaml 
```

Verify the devices now - 

```
kubectl exec privilegedpod -it -- ls /dev
```


---

## Demo 3 

Create a pod with security contexts set at pod as well as container 

```
vi pod3.yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod3
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
  containers:
  - image: busybox
    name: busybox
    args:
    - sleep
    - "3600"
    securityContext:
      runAsUser: 2000
      readOnlyRootFilesystem: true

kubectl create -f pod3.yaml 
```

Verify if container securitycontext has overridden the pod securitycontext

```
kubectl exec pod3 -it -- /bin/sh

ps -ef 
PID   USER     TIME  COMMAND
    1 2000      0:00 sleep 3600
   18 2000      0:00 /bin/sh
   23 2000      0:00 ps -ef


```

---

## Demo 4 - 

Create a file securitycontext1.yaml 
```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: gcr.io/google-samples/node-hello:1.0
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo
    securityContext:
      allowPrivilegeEscalation: false

```

Create the pod - ` kubectl create -f securitycontext1.yaml` 

```
kubectl get pods 
NAME                    READY   STATUS    RESTARTS   AGE
security-context-demo   1/1     Running   0          10m
```

Verify if the corresponding security contexts are applied - 

```
kubectl exec -it security-context-demo bash 

root@master:~# kubectl exec -it security-context-demo bash 
I have no name!@security-context-demo:/$ 
```

Verify the UID - 

```
 ps -ef 
UID        PID  PPID  C STIME TTY          TIME CMD
1000         1     0  0 14:09 ?        00:00:00 /bin/sh -c node server.js

```

Verify the files inside /data/demo - all files created inside this directory must have the ownership of 1000:2000


```
cd /data/demo
touch test 
ls -ltra test 
-rw-r--r-- 1 1000 2000    0 Dec 28 14:10 test

```

Exit the container 


--- 

## Demo 5 - Security context for containers dropping capabilities

Create a file - demo2.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: flask-cap
  namespace: default
spec:
  containers:
  - image: mateobur/flask
    name: flask-cap
    securityContext:
      capabilities:
        drop:
          - NET_RAW
          - CHOWN

```

Apply the file - kubectl create -f `demo2.yaml` 

Enter the pod - 

```
kubectl exec -it flask-cap bash
```

Verify the your capabilites are dropped 

```

root@flask-cap:/# ping 8.8.8.8
ping: Lacking privilege for raw socket.
root@flask-cap:/# chown root:root * 
chown: changing ownership of 'sys': Read-only file system

```
