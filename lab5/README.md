使用 nslookup 查詢 headless service：
```bash
kubectl run nslookup --rm -it --image=busybox --restart=Never -- nslookup lab2-headless-service.default.svc.cluster.local
```
觀察結果與使用 `kubectl get pod -o wide` 的結果是否一致。

