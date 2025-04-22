若 Pod A 要連接到 Service，則會經過以下的路徑：
```
[Pod A] --> my-service:80
         |
         v
[nat: PREROUTING]
         |
         v
[KUBE-SVC-XXXXXX]
         |
         v
[select KUBE-SEP-XXX]
         |
         v
[DNAT to Pod IP:PORT]
         |
         v
[Pod B (10.244.2.5:8080)]
```
至於 iptalbes 如何完成以下路徑，我們可以使用以下命令來觀察 NAT Table 的規則：

```
sudo iptables-legacy -t nat -L -n --line-numbers | grep KUBE
```

PREOUTING 與 OUTPUT 兩個鏈的規則會在 Pod A 的流量進入 NAT 時被觸發，然後會轉到 KUBE-SERVICES 鏈。KUBE-SERVICES 鏈會根據服務的 ClusterIP 和端口號選擇一個 KUBE-SVC 鏈，然後再選擇一個 KUBE-SEP 鏈，最後將流量 DNAT 到 Pod B 的 IP 和端口。

```
Chain PREROUTING (policy ACCEPT)
num  target     prot opt source               destination         
1    cali-PREROUTING  all  --  0.0.0.0/0            0.0.0.0/0            /* cali:6gwbT8clXdHdC1b1 */
2    KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination         
1    cali-OUTPUT  all  --  0.0.0.0/0            0.0.0.0/0            /* cali:tVnHkvAo15HuiPy0 */
2    KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
```

參考上方 NAT Table 的 PREROUTING 和 OUTPUT 鏈，前者用來處理跨節點的封包，後者用來處理本地 Pod 的封包。這兩個鏈都會將流量轉發到 KUBE-SERVICES 鏈，然後 KUBE-SERVICES 鏈會根據服務的 ClusterIP 和端口號選擇一個 KUBE-SVC 鏈。


```
Chain KUBE-SERVICES (2 references)
num  target     prot opt source               destination         
1    KUBE-SVC-EX4PWVDXGVVZMQ46  tcp  --  0.0.0.0/0            10.152.183.229       /* monitoring/tempo:grpc-tempo-jaeger cluster IP */ tcp dpt:14250
2    KUBE-SVC-3CW43ZE66AH4BLOU  tcp  --  0.0.0.0/0            10.152.183.221       /* monitoring/loki:http-metrics cluster IP */ tcp dpt:3100
3    KUBE-SVC-2ICPDK7FCUBBQ2U5  tcp  --  0.0.0.0/0            10.152.183.178       /* monitoring/prometheus-stack-kube-prom-alertmanager:http-web cluster IP */ tcp dpt:9093
4    KUBE-SVC-XSUB4FTDU6UIN2DA  tcp  --  0.0.0.0/0            10.152.183.229       /* monitoring/tempo:tempo-otlp-legacy cluster IP */ tcp dpt:55680
5    KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  0.0.0.0/0            10.152.183.1         /* default/kubernetes:https cluster IP */ tcp dpt:443
```

上方的 KUBE-SERVICES 有兩個參照，分別是 PREROUTING 和 OUTPUT 鏈。這兩個鏈會將流量轉發到 KUBE-SVC 鏈，然後 KUBE-SVC 鏈會根據服務的 ClusterIP 和端口號選擇一個 KUBE-SEP 鏈。
若封包的目的地是 10.152.183.1:443，則會轉到 KUBE-SVC-NPX46M4PTMTKRN6Y 鏈：

```
Chain KUBE-SVC-NPX46M4PTMTKRN6Y (1 references)
num  target     prot opt source               destination         
1    KUBE-MARK-MASQ  tcp  -- !10.1.0.0/16          10.152.183.1         /* default/kubernetes:https cluster IP */ tcp dpt:443
2    KUBE-SEP-SZEZPYI4DJWRAXYB  all  --  0.0.0.0/0            0.0.0.0/0            /* default/kubernetes:https -> 10.10.2.58:16443 */
```

KUBE-SVC-NPX46M4PTMTKRN6Y 鏈有兩個規則，第一個規則會將流量標記為 KUBE-MARK-MASQ，第二個規則會將流量轉發到 KUBE-SEP 鏈。前者會對來源不屬於叢集的流量進行標記，讓節點本身或是叢集外的流量可以被正確地回應：

```
封包（非 Pod → Service）
   ↓
KUBE-SVC-XXX
   ↓（來源非 Pod）
KUBE-MARK-MASQ（打上 mark）
   ↓
KUBE-SEP-XXX（DNAT 到 Pod）
   ↓
KUBE-POSTROUTING
   ↓（match mark）
MASQUERADE（SNAT 成 Node IP）
```

最後，讓我們觀察 KUBE-SEP-SZEZPYI4DJWRAXYB 鏈的規則：

```
Chain KUBE-SEP-SZEZPYI4DJWRAXYB (1 references)
num  target     prot opt source               destination         
1    KUBE-MARK-MASQ  all  --  10.10.2.58           0.0.0.0/0            /* default/kubernetes:https */
2    DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/kubernetes:https */ tcp to:10.10.2.58:16443
```

第一個規則會將流量標記為 KUBE-MARK-MASQ，用途是處理節點本身發出的流量，避免非對稱路由的問題，舉例來說：
假設這台 node（IP: 10.10.2.58）上，跑了某個應用程式，它要打 Kubernetes API：
curl https://10.152.183.1:443  # ClusterIP of kubernetes service
這流量會：

```
透過 iptables 的 KUBE-SERVICES → 找到 KUBE-SVC-...
最後 DNAT 到這台機器自己的 IP：10.10.2.58:16443
```

但因為 source IP 也是 10.10.2.58，目的 IP 也是 10.10.2.58， 看起來像是直接連自己，路由表就直接送回去了，完全不經過 DNAT 回程路徑，這會讓：
- 封包來的路徑：經過 iptables + DNAT
- 回應走的路徑：直接送回去（不走 iptables）

由於 TCP 連線完全對不上，導致 connection reset。

第二個規則會將流量 DNAT 到 Pod 的 IP 和端口，這樣就可以將流量轉發到 Pod 上了。這個規則的作用是將流量轉發到 Pod 的 IP 和端口，讓 Pod 可以接收到來自 Service 的請求。

