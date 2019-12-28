# Kubernetes Security Context 

A security context defines privilege and access control settings for a Pod or Container. Security context settings include - 

* ACL - chown / chmod 
* SELINUX
* Privileged Mode / Unprivileged mode 
* SeCOMP
* AppArmor
* AllowPrivilegeEscalation

## Demo 1 - 

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

## Demo 2 - Security context for containers 

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
