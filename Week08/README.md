# W08｜容器生產實踐

## Healthcheck 故障測試
- 停 db 後幾秒被標 unhealthy：36 秒
- 對應的 log 訊息：`"GET /healthz HTTP/1.1" 503`

## Log 失控估算
- noisy 容器 30s log 大小：1605529862 bytes
- 預估 24h 大小：約 4306.37 GB
- 套 rotation 後穩定上限：整個容器目錄 du 量到 4.8 MB

## 資源限制實驗
| 實驗 | 命令 | 觀察結果 | 對應 cgroup 檔 | 值 |
|---|---|---|---|---|
| OOM | stress-ng --vm 1 --vm-bytes 200m | exit 137, OOMKilled=true | memory.max | 134217728 |
| CPU throttle | stress-ng --cpu 4 | docker stats CPU% ≈ 50% | cpu.max | 50000 100000 |

## 權限四階對照
| 階梯 | id | CapEff | NoNewPrivs | curl /healthz |
|---|---|---|---|---|
| 0 | uid=0 | 00000000a80425fb | 0 | 200 |
| 1 | uid=1000 | 0000000000000000 | 0 | 200 |
| 2 | uid=1000 | 0000000000000000 | 0 | 200 |
| 3 | uid=1000 | 0000000000000000 | 0 | 200 |
| 4 | uid=1000 | 0000000000000000 | 1 | 200 |

## 排錯紀錄
- 症狀：跑權限四階那組指令時，五階每一階都吐 `docker: invalid reference format`，往上翻還看到 `No such image: sha256:...`
- 診斷：image 名稱是用 `IMG=$(docker compose images app | awk ...)` 擷取的，欄位對位沒對準，結果 IMG 被指派成空字串，等於在跑 `docker run "" ...`，自然無效
- 修正：捨棄解析、直接把名稱寫死成 `IMG=w08-app`
- 驗證：重跑五階，tier0 到 tier4 全部正常印出 id 與 CapEff、NoNewPrivs，資料完整無缺

## 設計決策
數值是先看 docker stats 的實測再往上抓邊際決定的。app 閒置時記憶體實測落在約 22.6 MiB，配 256m 等於預留了十倍以上的突波空間，足夠吸收尖峰又不至於浪費；cpus 給 0.5 是因為這支 Flask 幾乎沒有 CPU 密集運算，半顆核已經過剩。db 那側因為 postgres 從初始化到查詢都比較重，所以放寬到 512m 與 1.0 顆。至於 tmpfs，是被 read_only: true 逼出來的：根檔案系統轉唯讀後，任何執行期想落地的路徑都會回 errno 30，於是我替兩個確定會被寫到的位置掛了記憶體暫存——/tmp 接標準函式庫的暫存檔，/home/appuser/.cache 接 Python 執行期的快取。這兩處停機後都不需要保留，用 tmpfs 比 named volume 更快、也不會在 host 落下任何殘跡，所以刻意不走持久化那條路。