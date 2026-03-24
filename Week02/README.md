# W02｜VMware 網路模式與雙 VM 排錯

## 網路配置

| VM | 網卡 | 模式 | IP | 用途 |
|---|---|---|---|---|
| dev-a | NIC 1 | NAT | 192.168.79.128 | 上網 |
| dev-a | NIC 2 | Host-only | 192.168.254.128 | 內網互連 |
| server-b | NIC 1 | Host-only | 192.168.254.129 | 內網互連 |

## 連線驗證紀錄

- [x] dev-a NAT 可上網：`ping google.com` 輸出
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week02/res/1.png?raw=true)
- [x] 雙向互 ping 成功：貼上雙方 `ping` 輸出
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week02/res/2.1.png?raw=true)
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week02/res/2.2.png?raw=true)
- [x] SSH 連線成功：`ssh <user>@<ip> "hostname"` 輸出
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week02/res/3.png?raw=true)
- [x] SCP 傳檔成功：`cat /tmp/test-from-dev.txt` 在 server-b 上的輸出
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week02/res/4.png?raw=true)
- [x] server-b 不能上網：`ping 8.8.8.8` 失敗輸出
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week02/res/5.png?raw=true)

## 故障演練一：介面停用

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| server-b 介面狀態 | UP | DOWN | UP |
| dev-a ping server-b | 成功 | 失敗 | 成功 |
| dev-a SSH server-b | 成功 | 失敗 | 成功 |

## 故障演練二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | 有監聽 |
| dev-a ping server-b | 成功 | 成功 | 成功 |
| dev-a SSH server-b | 成功 | Connection refused | 成功 |

## 排錯順序
遇到連線問題時，固定從 L2 往上排：
- L2（介面層）— 先確認介面有沒有活著
```bash
ip address show
```
看目標介面是否顯示 UP，以及有沒有拿到 IP。介面 `DOWN` 或沒有 IP，後面兩層不用看，直接在這層修。

- L3（網路層）— 路由對了再 ping
```bash
ip route show
ping -c 4 <對端 IP>
```
先確認有沒有預設路由或對應網段的路由。`ping` 通代表 L3 可達，但不代表服務可用。`ping` 失敗有兩種原因：路由不存在、或對端介面 `DOWN`。

- L4（服務層）— 最後才查服務
```bash
ss -tlnp | grep :22
ssh <user>@<ip> "hostname"
```
`ping` 通但 SSH 出現 Connection refused，代表 L3 沒問題、L4 服務沒開。用 `ss` 確認 port 22 是否在監聽。

## 網路拓樸圖
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week02/res/Network.png?raw=true)

## 排錯紀錄
- 症狀： 步驟 6 雙向互 `ping`，從 `dev-a` ping `server-b` 失敗，但 `server-b` 的 `ip address show` 看起來有 IP。

- 診斷： 先跑 `ip address show` 確認兩台的 Host-only 介面，發現兩台的 IP 網段不一樣，`dev-a` 是 `192.168.56.x`，`server-b` 是 `192.168.254.x`，根本不在同一個 VMnet。

- 修正： 進 VMware → Edit → Virtual Network Editor 確認 VMnet1 的網段設定，把 `server-b` 的 Host-only 網卡指定到和 dev-a 同一個 VMnet1，重啟 VM 取得新 IP。

- 驗證： 兩台 `ip address show` 確認都在 `192.168.254.0/24` 網段，再跑雙向 `ping` 成功。

## 設計決策
**Q: 為什麼 server-b 只設 Host-only 不給 NAT？**
1. `server-b` 的定位是內網服務機，給它 NAT 網卡會讓它有獨立的對外路由，Host-only 的隔離性就沒有意義了。
2. 職責分離比較清楚：`dev-a` 負責對外，`server-b` 只在內網提供服務。需要在 `server-b` 裝套件時，用臨時加 NAT 卡再移除、或從 `dev-a` SCP 傳入的方式處理，不讓它常態保有上網能力。