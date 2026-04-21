# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | 系統級設定檔目錄，存放各服務的靜態設定 | 存放 `daemon.json`，用於自訂 Docker daemon 行為（如 log driver、registry mirror 等） |
| /var/lib/docker/ | 程式運行產生的持久性狀態資料 | 存放映像（images）、容器（containers）、volumes 及 overlay2 層資料 |
| /usr/bin/docker | 使用者可執行的應用程式 | Docker CLI 工具本體，使用者輸入 `docker` 指令時執行的就是這個檔案 |
| /run/docker.sock | 執行期暫存資料（PID、socket 等），開機建立、關機消失 | Docker daemon 的 Unix socket，CLI 透過此 socket 向 daemon 發送 API 請求 |

## Docker 系統資訊

- Storage Driver：`overlayfs`
- Docker Root Dir：`/var/lib/docker`
- 拉取映像前 /var/lib/docker/ 大小：`404K`
- 拉取映像後 /var/lib/docker/ 大小：`408K`

## 權限結構

### Docker Socket 權限解讀

```
srw-rw---- 1 root docker 0  4月 21 20:14 /var/run/docker.sock
```

| 欄位 | 值 | 說明 |
|---|---|---|
| 檔案類型 | `s` | Unix socket，不是普通檔案（`-`）也不是目錄（`d`） |
| owner 權限 | `rw-` | owner 為 `root`，有讀寫權限，無執行 |
| group 權限 | `rw-` | group 為 `docker`，docker 群組成員有讀寫權限 |
| others 權限 | `---` | 不是 root 且不在 docker 群組的使用者，完全無法存取此 socket |

### 使用者群組

```
uid=1000(xi) gid=1000(xi) groups=1000(xi),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),114(lpadmin)
```

### 安全意涵

Docker daemon 本身以 `root` 身份運行，而 `/run/docker.sock` 是控制 daemon 的唯一入口。能讀寫這個 socket，就等於能對 daemon 下任何命令——包括用 `-v` 將 Host 的任意目錄掛載進容器。

若使用者已在 docker 群組，可執行：
```bash
docker run --rm -v /etc/shadow:/host-shadow:ro alpine cat /host-shadow
```
這條命令會把 Host 的 `/etc/shadow` 掛進容器並讀取，完全不需要 `sudo`。這證明了 **docker group 成員實際上擁有等同 root 的系統存取能力**。

因此在生產環境中，不應該隨意將使用者加入 docker group；應維持每次 `sudo docker` 的方式，讓每次提權都有意識、可追蹤。

## 程序與服務管理

### systemctl status docker

```
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-04-21 20:15:16 CST; 3min 41s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 1622 (dockerd)
      Tasks: 11
     Memory: 153.1M (peak: 153.6M)
        CPU: 6.317s
     CGroup: /system.slice/docker.service
             └─1622 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```
### journalctl 日誌分析

```
4月 21 20:15:08  Starting docker.service...
4月 21 20:15:10  Starting up
4月 21 20:15:10  OTEL tracing is not configured, using no-op tracer provider
4月 21 20:15:10  CDI directory does not exist, skipping（/etc/cdi、/var/run/cdi）
4月 21 20:15:10  detected 127.0.0.53 nameserver, assuming systemd-resolved
4月 21 20:15:10  Creating a containerd client（address=/run/containerd/containerd.sock）
4月 21 20:15:10  Loading containers: start.
4月 21 20:15:10  NRI is disabled
4月 21 20:15:11  Deleting nftables IPv4/IPv6 rules
4月 21 20:15:16  Loading containers: done.
4月 21 20:15:16  Docker daemon（version=29.3.0, storage-driver=overlayfs）
4月 21 20:15:16  Initializing buildkit → Completed buildkit initialization
4月 21 20:15:16  Daemon has completed initialization
4月 21 20:15:16  API listen on /run/docker.sock
4月 21 20:15:16  Started docker.service.
4月 21 20:16:43  image pulled（nginx:latest）
```

- `daemon` 啟動完整流程約需 8 秒
- `CDI directory does not exist` 是正常訊息，CDI 是容器設備介面，未設定時跳過
- `Deleting nftables rules: exit status 1` 是正常的——daemon 啟動時嘗試清除舊的防火牆規則，若規則本就不存在則報錯，不影響運作
- `API listen on /run/docker.sock` 這行出現代表 daemon 已就緒，可以接受 CLI 請求
- 最後一行記錄了 `nginx:latest` 被拉取的事件

### CLI vs Daemon 差異

- **CLI** 只是一個命令列工具，執行 `docker` 時才會跑，跑完就結束。它本身不做任何容器管理，只是把使用者的輸入翻譯成 API 請求，透過 `/run/docker.sock` 送給 daemon。
- **Daemon** 是 systemd 管理的背景服務，從開機就一直在跑，負責實際的映像下載、容器建立、網路設定等所有工作。

## 環境變數

- $PATH：`/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin`
- which docker：`/usr/bin/docker`
- 容器內外環境變數差異觀察：Host 的 `$PATH` 包含 `/snap/bin`、`/usr/games` 等針對這台機器設定的路徑；容器內只有最基本的系統路徑，沒有繼承 Host 的任何 shell 設定。`$HOSTNAME` 是容器 ID 的前段，`$HOME` 是 root 的家目錄。這說明容器是完全隔離的執行環境，不繼承 Host 的使用者設定。

## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| systemctl status docker | active | inactive | active |
| docker --version | 正常 | 正常 | 正常 |
| docker ps | 正常 | Cannot connect | 正常 |
| ps aux grep dockerd | 有 process | 無 process | 有 process |

## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ls -la docker.sock 權限 | srw-rw---- | srw------- | srw-rw---- |
| docker ps（不加 sudo） | 正常 | permission denied | 正常 |
| sudo docker ps | 正常 | 正常 | 正常 |
| systemctl status docker | active | active | active |

## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` | daemon 沒有在執行，socket 不存在或無人監聽 | `systemctl status docker` 確認 daemon 狀態 → 若 inactive 則 `sudo systemctl start docker` |
| `permission denied while trying to connect to the Docker daemon socket` | daemon 在跑，但當前使用者對 socket 沒有存取權限 | `ls -la /var/run/docker.sock` 確認 group 是否為 docker、`id` 確認使用者是否在 docker 群組 |

## 排錯紀錄

**症狀：** 執行 `docker run --rm alpine env` 回傳 `permission denied while trying to connect to the docker API at unix:///var/run/docker.sock`

**診斷：**
1. 先確認錯誤類型——是 `permission denied` 而非 `Cannot connect`，代表 daemon 在跑，是權限問題
2. 執行 `sudo docker ps`，成功，確認 daemon 正常
3. 執行 `ls -la /var/run/docker.sock`，確認 group 為 `docker`、group 權限為 `rw-`
4. 執行 `id`，發現使用者 `xi` 的 groups 清單中沒有 `docker`——root cause 確認

**修正：**
```bash
sudo usermod -aG docker xi
exit   # 登出
# 重新 SSH 登入
```

**驗證：**
```bash
id     # 確認 groups 包含 docker
docker ps          # 不加 sudo，正常回傳
docker run --rm hello-world   # 出現 Hello from Docker!
```

## 設計決策

**Q: 為什麼教學環境用 `usermod -aG docker $USER` 而不是每次 sudo？**

- **好處**：不用每次打 `sudo`，降低操作摩擦，專注在指令邏輯本身
- **風險**：docker group 成員等同 root 權限，但沒有 sudo 提示提醒你正在提權，是隱性的高權限
- **邊界**：教學機 / 個人開發機可接受；生產環境或多人共用機器不應這樣做，應維持 `sudo docker` 或改用 rootless Docker