# W01｜虛擬化概論、環境建置與 Snapshot 機制

## 環境資訊
- Host OS：Windows 11
- VM 名稱：vct-w01-411630394
- Ubuntu 版本：24.04
- Docker 版本：Docker version 29.3.0, build 5927d80
- Docker Compose 版本：v5.1.0

## VM 資源配置驗證

| 項目 | VMware 設定值 | VM 內命令 | VM 內輸出 |
|---|---|---|---|
| CPU | 2 vCPU | `lscpu \| grep "^CPU(s)"` | 2 |
| 記憶體 | 4 GB | `free -h \| grep Mem` | 3.8Gi |
| 磁碟 | 40 GB | `df -h /` | 40G |
| Hypervisor | VMware | `lscpu \| grep Hypervisor` | v5.1.0 |

## 四層驗收證據
- [x] ① Repository：`cat /etc/apt/sources.list.d/docker.list` 輸出
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week01/res/1.png?raw=true)
- [x] ② Engine：`dpkg -l | grep docker-ce` 輸出
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week01/res/2.png?raw=true)
- [x] ③ Daemon：`sudo systemctl status docker` 顯示 active
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week01/res/3.png?raw=true)
- [x] ④ 端到端：`sudo docker run hello-world` 成功輸出
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week01/res/4.png?raw=true)
- [x] Compose：`docker compose version` 可執行
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week01/res/5.png?raw=true)

## 容器操作紀錄
- [x] nginx：`sudo docker run -d -p 8080:80 nginx` + `curl localhost:8080` 輸出
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week01/res/6.png?raw=true)
- [x] alpine：`sudo docker run -it --rm alpine /bin/sh` 內部命令與輸出
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week01/res/7.png?raw=true)
- [x] 映像列表：`sudo docker images` 輸出
![](https://github.com/CHENGXee/411630394_TKUCVT/blob/main/Week01/res/8.png?raw=true)

## Snapshot 清單

| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | Docker 安裝完成、四層驗證通過後，尚未進行任何額外設定時 | 代表 OS 健康、Docker 可運作的最乾淨起點 | `hostnamectl`、`ip route`、`sudo docker --version`、`docker compose version`、`sudo systemctl status docker`、`sudo docker run --rm hello-world` 全部通過 |
| docker-ready | nginx / alpine 映像拉取完畢、nginx curl 驗證通過後 | 代表完整交付環境就緒（Engine + 映像均就位）| `sudo systemctl status docker`、`sudo docker run --rm hello-world`、`sudo docker images`（確認 nginx、alpine 均在）、`curl http://localhost:8080` 通過 |

## 故障演練三階段對照

| 項目 | 故障前（基線） | 故障中（注入後） | 回復後 |
|---|---|---|---|
| docker.list 存在 | 是 | 否 | 是 |
| apt-cache policy 有候選版本 | 是 | 否 | 是 |
| docker 重裝可行 | 是 | 否 | 是 |
| hello-world 成功 | 是 | N/A | 是 |
| nginx curl 成功 | 是 | N/A | 是 |

## 手動修復 vs Snapshot 回復

| 面向 | 手動修復 | Snapshot 回復 |
|---|---|---|
| 所需時間 | 約 30 秒 | 約 1–2 分鐘 |
| 適用情境 | 根因明確、改動單一、修復步驟可逆 | 根因不明或改動範圍廣、多個設定檔狀態不確定 |
| 風險 | 若記憶不完整、改動多處，可能修到一半引入新問題；人為失誤率較高 | 回復到的節點本身必須是健康的；若 `snapshot` 建立時環境已有問題，回復也沒用 |

## Snapshot 保留策略
- 新增條件： 每次安裝新工具、大幅修改系統設定、或即將進行破壞性實驗前，且當前環境已通過全套驗證命令（`hello-world`、`docker ps`、`systemctl status docker` 等）。
- 保留上限： 最多保留 3 個活躍 snapshot。超過時，刪除最舊且已被更新節點取代的 `snapshot`，避免差異磁碟鏈過長拖慢 I/O。
- 刪除條件： 確認有更新且驗證過的節點存在，且舊節點所代表的狀態不再需要回復時，刪除最舊的一個。禁止在「不確定舊節點是否還需要」時貿然刪除。

## 最小可重現命令鏈
```bash
# ── 【故障前基線】確認環境健康 ──
echo "=== 故障前 ==="
ls /etc/apt/sources.list.d/
apt-cache policy docker-ce | head -10
sudo systemctl status docker --no-pager
sudo docker run --rm hello-world

# ── 【注入故障】移除 docker APT source ──
sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken
sudo apt update

# ── 【故障觀測】確認 APT source 消失 ──
echo "=== 故障中 ==="
ls /etc/apt/sources.list.d/
apt-cache policy docker-ce | head -10
sudo apt -y install docker-ce 2>&1 | tail -5

# ── 【觸發 Snapshot 回復】關機後在 VMware 回復至 docker-ready ──
sudo poweroff
# （開機後繼續）

# ── 【回復後驗證】所有狀態必須與故障前一致 ──
echo "=== 回復後 ==="
ls /etc/apt/sources.list.d/
cat /etc/apt/sources.list.d/docker.list
sudo apt update
sudo systemctl status docker --no-pager
sudo docker --version
docker compose version
sudo docker run --rm hello-world
sudo docker images

# ── 【功能驗證】端到端服務仍可運作 ──
sudo docker run -d --name test-nginx -p 8080:80 nginx:latest
curl http://localhost:8080
sudo docker stop test-nginx && sudo docker rm test-nginx
```

## 排錯紀錄
- 症狀： 執行 `sudo apt update` 後出現 `NO_PUBKEY` 錯誤，或 `apt-cache policy docker-ce` 無候選版本，即使 `docker.list` 檔案存在。
- 診斷： 先確認 `docker.list` 內容是否正確（`cat /etc/apt/sources.list.d/docker.list`），再確認 GPG key 是否成功匯入（`ls -la /etc/apt/keyrings/docker.gpg`）。發現 `docker.gpg` 存在但權限為 600，APT 以其他身份讀取時失敗。
- 修正： 執行 `sudo chmod a+r /etc/apt/keyrings/docker.gpg` 修正權限，再重跑 `sudo apt update`。
- 驗證： `apt update` 輸出中出現 https://download.docker.com 來源，且 `apt-cache policy docker-ce` 顯示有效候選版本。

## 設計決策
選擇先 `sudo poweroff` 關機再回復 `snapshot`，而不是直接在 VM 執行中回復。主要考量是：熱回復雖然快，但 VM 還在跑的時候記憶體狀態不確定，回復後可能有殘留，不好判斷到底有沒有真的回到乾淨狀態。關機回復雖然要多等一兩分鐘重開機，但服務是從頭冷啟動的，驗證結果比較可信。