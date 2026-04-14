# W03｜多 VM 架構：分層管理與最小暴露設計

## 網路配置

| VM | 角色 | 網卡 | 模式 | IP | 開放埠與來源 |
|---|---|---|---|---|---|
| bastion | 跳板機 | NIC 1 | NAT | 192.168.79.128 | — |
| bastion | 跳板機 | NIC 2 | Host-only | 192.168.254.128 | SSH from any |
| app | 應用層 | NIC 1 | Host-only | 192.168.254.129 | SSH from 192.168.254.0/24 |
| db | 資料層 | NIC 1 | Host-only | 192.168.254.130 | SSH from app + bastion |

## SSH 金鑰認證

- 金鑰類型：**ed25519**
- 公鑰部署到：`app` 和 `db` 的 `~/.ssh/authorized_keys`
- 免密碼登入驗證：
  - bastion → app：`ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIaRnwOpyQYDCniK+5qyTc7cAWp0QCbQ+lia/NHnuGtt bastion-key`
  - bastion → db：`ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIaRnwOpyQYDCniK+5qyTc7cAWp0QCbQ+lia/NHnuGtt bastion-key`

## 防火牆規則

### app 的 ufw status

```
狀態: 啓用
日誌: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
新建設定檔案: skip

至                          動作          來自
--                         --            --
22/tcp                     ALLOW IN      192.168.254.0/24
```

### db 的 ufw status

```
狀態: 啓用
日誌: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
新建設定檔案: skip

至                          動作          來自
--                         --            --
22/tcp                     ALLOW IN      192.168.254.128
22/tcp                     ALLOW IN      192.168.254.129
```

### 防火牆確實在擋的證據
```
curl: (28) Connection timed out after 5002 milliseconds
```

## ProxyJump 跳板連線 (windows)

- 指令：

    ```powershell
    @"
    Host bastion
        HostName 192.168.254.128
        User xi

    Host app
        HostName 192.168.254.129
        User xi
        ProxyJump bastion

    Host db
        HostName 192.168.254.130
        User xi
        ProxyJump bastion
    "@ | Add-Content "$env:USERPROFILE\.ssh\config"
    ```

- 驗證輸出：

    ```powershell
    $ ssh app "hostname"
    # app

    $ ssh db "hostname"
    # db
    ```

- SCP 傳檔驗證：

    ```powershell
    "Test file via ProxyJump" | Out-File -FilePath "$env:TEMP\proxy-test.txt" -Encoding utf8

    # 傳到 app
    scp $env:TEMP\proxy-test.txt app:/tmp/
    ssh app "cat /tmp/proxy-test.txt"
    # 輸出：Test file via ProxyJump

    # 傳到 db
    scp $env:TEMP\proxy-test.txt db:/tmp/
    ssh db "cat /tmp/proxy-test.txt"
    # 輸出：Test file via ProxyJump
    ```

## 故障場景一：防火牆全封鎖

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| app ufw status | active + rules | deny all | active + rules |
| bastion ping app | 成功 | 成功 | 成功 |
| bastion SSH app | 成功 | **timed out** | 成功 |

## 故障場景二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp \| grep :22 | 有監聽 | 無監聽 | 有監聽 |
| bastion ping app | 成功 | 成功 | 成功 |
| bastion SSH app | 成功 | **refused** | 成功 |

## timeout vs refused 差異

`Connection timed out`：封包送出去之後沒有任何回應。代表封包在途中被防火牆丟棄，排錯方向是檢查防火牆規則（`sudo ufw status`）。

`Connection refused`：目標機器有收到請求，但立刻回了一個「拒絕」。代表網路是通的，但目標 port 上沒有服務在監聽，排錯方向是確認服務是否在跑（`ss -tlnp`）。

## 網路拓樸圖
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week03/res/Network.png?raw=true)

## 排錯紀錄
- 症狀：Windows 上貼多行 PowerShell here-string 出現語法錯誤
- 診斷：PowerShell here-string 規則嚴格，`"@` 結尾符號必須單獨置於行首，前方不可有任何空白；且 `@"` 後面必須立即換行
- 修正：將 `@"..."@` 內容重新排版，確保 `"@` 單獨佔一行且頂格；或改用分號串接成單行指令
- 驗證：重新執行後無語法錯誤，`~/.ssh/config` 內容正確寫入

## 設計決策
**Q: 為什麼 db 允許 bastion 直連而不是只允許從 app 跳？**
1. 管理彈性：bastion 作為唯一對外入口，必須保有直接管理 db 的能力。若 app 服務異常，管理員仍可從 bastion 緊急存取 db，不會因 app 故障而失去控制權。
2. 故障隔離：強制所有 db 操作都經過 app，會讓 app 成為單點故障。app 一旦掛掉，db 的管理通道也跟著斷，反而降低了整體韌性。