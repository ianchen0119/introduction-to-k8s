如果你使用的是 microk8s，可以用以下命令安裝 Multus：
```bash
$ microk8s enable multus
```
否則，請使用以下命令安裝 Multus：
```bash
$ kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/refs/heads/master/deployments/multus-daemonset.yml
```

請確認 Multus 正常工作，再嘗試後續的實驗：
```bash
$ kubectl get pods -n kube-system -l app=multus
```
結果中的 `READY` 欄位應該顯示為 `1/1`，表示 Multus DaemonSet 正常運行。

接著，在 HOST 使用以下命令新增 dummy interface：
```bash
$ sudo ip link add dummy-lab6 type dummy
$ sudo ip link set dummy-lab6 up
```

新增需要的網路裝置後，即可部署 nginx pod：
```bash
$ kubectl apply -f deployment.yaml
```

檢查 nginx pod 是否正常運行：
```bash
$ kubectl get po
NAME                                           READY   STATUS    RESTARTS   AGE
lab6-deployment-679b58b87d-lbgl4               1/1     Running   0          12s
```

接著，使用以下命令檢查 nginx pod 的網路設定：
```bash
$ kubectl exec -it lab6-deployment-679b58b87d-lbgl4  -- bash
$ apt-get update && apt-get install -y iproute2
$ ip r
default via 169.254.1.1 dev eth0 
10.10.20.0/24 dev eth1 
10.10.30.0/24 dev eth1 proto kernel scope link src 10.10.30.100 
169.254.1.1 dev eth0 scope link 
$ exit
```
