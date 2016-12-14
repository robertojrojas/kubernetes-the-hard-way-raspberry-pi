# Smoke Test

This lab walks you through a quick smoke test to make sure things are working.

## Test

```
kubectl run nginx --image=werwolfby/armhf-alpine-nginx --port=80 --replicas=2
```

```
deployment "nginx" created
```

```
kubectl get pods -o wide
```
```
NAME                     READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-2032906785-sokxz   1/1       Running   0          21s       10.200.1.2   worker1
nginx-2032906785-u8rzc   1/1       Running   0          21s       10.200.0.2   worker0
```

```
kubectl expose deployment nginx --type NodePort
```

```
service "nginx" exposed
```

> Note that --type=LoadBalancer will not work because it only works for cloud providers.

Grab the `NodePort` that was setup for the nginx service:

```
NODE_PORT=$(kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Grab the `EXTERNAL_IP` for one of the worker nodes:

```
Perform testing from worker raspbery pi
```

---

Test the nginx service using cURL:

```
curl http://${RASPBERRY_IP}:${NODE_PORT}
```

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
