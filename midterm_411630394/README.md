# 期中實作 — 411630394 鄭丞希

## 1. 架構與 IP 表
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/midterm_411630394/network-diagram.png?raw=true)

## 2. Part A：VM 與網路
| VM | 網卡 | 模式 | IP |
|---|---|---|---|
| bastion | NIC 1 | NAT | 192.168.79.128 |
| bastion | NIC 2 | Host-only | 192.168.254.128 |
| app | NIC 1 | Host-only | 192.168.254.129 |
| host | NIC 1 | Host-only | 192.168.254.130 |

## 3. Part B：金鑰、ufw、ProxyJump
### bastion ufw status
```
狀態： 啓用

至                          動作          來自
-                          --          --
22/tcp                     ALLOW       Anywhere                  
22/tcp (v6)                ALLOW       Anywhere (v6) 
```

### app ufw status
```
狀態： 啓用

至                          動作          來自
-                          --          --
22/tcp                     ALLOW       192.168.254.0/24          
22/tcp                     ALLOW       192.168.254.128 
```

### ssh app
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/midterm_411630394/screenshots/ssh-proxyjump.png?raw=true)

## 4. Part C：Docker 服務
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/midterm_411630394/screenshots/docker-running.png?raw=true)

![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/midterm_411630394/screenshots/docker-running_2.png?raw=true)

## 5. Part D：故障演練
### 故障 1：F1
- 注入方式：關閉 app `Host-only` 網卡
    ```bash
    sudo ip link set ens33 down
    ```
- 故障前：
    - **Host:** `ssh app` 可以正常連線
    - **Host:** `ping 192.168.254.129` 正常
    ![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/midterm_411630394/screenshots/fault-A-before.png?raw=true)
- 故障中：
    - **Host:** `ssh app` timeout
    - **Host:** `ping 192.168.254.129` 不通
    - ![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/midterm_411630394/screenshots/fault-A-during.png?raw=true)
- 回復後：
    將 app 網卡重啟 `sudo ip link set ens33 up`
    - **Host:** `ssh app` 可以正常連線
    - **Host:** `ping 192.168.254.129` 正常
    ![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/midterm_411630394/screenshots/fault-A-after.png?raw=true)
- 診斷推論：
    **本次故障為網路層問題**
    - 判斷依據：
        1. `ping` 無法連線 → 表示問題發生在 L2/L3
        2. 檢查 `ip link` 發現網卡為 DOWN
        3. 因此確定為網卡關閉導致

### 故障 2：F2
- 注入方式：移除 app SSH 允許規則
    ```bash
    sudo ufw default deny incoming
    sudo ufw delete allow 22/tcp
    ```
- 故障前：
    - **Host:** `ssh app` 可以正常連線
    - **Host:** `ping 192.168.254.129` 正常
    ![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/midterm_411630394/screenshots/fault-B-before.png?raw=true)
- 故障中：
    - **Host:** `ssh app` timeout
    - **Host:** `ping 192.168.254.129` 正常
    - ![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/midterm_411630394/screenshots/fault-B-during.png?raw=true)
- 回復後：
    重新允許 bastion SSH `sudo ufw allow from 192.168.254.128 to any port 22 proto tcp`
    - **Host:** `ssh app` 可以正常連線
    - **Host:** `ping 192.168.254.129` 正常
    ![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/midterm_411630394/screenshots/fault-B-after.png?raw=true)
- 診斷推論：
    **本次故障為防火牆問題**
    - 判斷依據：
        1. `ping` 可以通 → 網路層正常
        2. `ssh` timeout → 表示封包被阻擋
        3. 使用 `sudo ufw status` 發現 22 port 未開放
        4. 因此確定為防火牆阻擋 SSH

### 症狀辨識
|診斷步驟|F1|F2|
| :--- | :--- | :--- |
| ping app | timeout | **通** |
| ssh app | timeout | timeout |
| 結論 | L2/L3 問題，介面或路由壞了 | L4 問題，防火牆擋了 TCP 22 |
## 6. 反思（200 字）
這次做完，對「分層隔離」或「timeout 不等於壞了」的理解有什麼改變？

## 7. Bonus 1

### DockerFile
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

### .dockerignore
```dockerignore
.git
node_modules
*.log
```

### 建立 Image 與執行 Container
```bash
docker build -t midterm-web .
docker run -d --name web2 -p 8081:80 midterm-web
```

### docker history
```
IMAGE          CREATED         CREATED BY                                       SIZE      COMMENT
d299a34e79d1   9 minutes ago   EXPOSE [80/tcp]                                  0B        buildkit.dockerfile.v0
<missing>      9 minutes ago   COPY index.html /usr/share/nginx/html/index.…   24.6kB    buildkit.dockerfile.v0
```

### 服務驗證
```bash
curl http://192.168.254.129:8081
```

![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/midterm_411630394/screenshots/bonus.png?raw=true)