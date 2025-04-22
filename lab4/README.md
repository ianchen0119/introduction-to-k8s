透過 NodePort 存取 nginx
```bash
$ curl 127.0.0.1:30080 
```

使用 nslookup 查詢 NodePort 的 IP
```bash
kubectl run nslookup --rm -it --image=busybox --restart=Never -- nslookup lab2-nodeport-service.default.svc.cluster.local
```
觀察結果與使用 `kubectl get svc` 的結果是否一致。

使用 nslookup 查詢 ClusterIP 的 IP
```bash
kubectl run nslookup --rm -it --image=busybox --restart=Never -- nslookup lab2-clusterip-service.default.svc.cluster.local
```
觀察結果與使用 `kubectl get svc` 的結果是否一致。