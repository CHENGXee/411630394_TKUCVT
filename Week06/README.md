# W06｜Docker Image 與 Dockerfile

## 映像組成
- **Layers 是什麼**：它是唯讀的檔案層。在技術上，每一層都是一個包含檔案系統差異（diff）的壓縮檔（tarball），對應到 Overlay2 的 `lowerdir`。多個映像檔如果來自同一個基礎環境，可以共享相同的 Layers，避免重複佔用硬碟空間。
- **Config 是什麼**：它是一份 JSON 格式的元數據（Metadata）設定檔。裡面記錄了容器啟動時的靜態設定，例如預設要執行的指令（CMD/ENTRYPOINT）、環境變數（ENV）、工作目錄（WORKDIR）、執行的使用者身份（USER）以及暴露的埠號（EXPOSE）。它不佔用檔案系統空間，只負責定義行為。
- **Manifest 是什麼**：它是整個映像檔的「索引清單」。它把上述的 Config JSON 檔和那一堆唯讀的 Layers 綁在一起。Manifest 裡記錄了每一個 Layer 的唯一雜湊值（Digest）與實際大小，讓 Docker Daemon 知道下載該映像檔時，具體需要拉取哪些檔案層。

## python:3.12-slim inspect 摘錄
- Config.Cmd：`["python3"]`
- Config.Env：`["PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","LANG=C.UTF-8","GPG_KEY=7169605F62C751356D054A26A821E680E5FA6305","PYTHON_VERSION=3.12.13","PYTHON_SHA256=c08bc65a81971c1dd5783182826503369466c7e67374d1646519adf05207b684"]`
- Config.WorkingDir：`""`
- RootFS.Layers 數量：`4`

## Layer 快取實驗
| 情境 | build 時間 |
|---|---|
| v1 首次 build | 0m7.211s |
| v1 改 app.py 後 rebuild | 0m6.136s |
| v2 首次 build | 0m6.811s |
| v2 改 app.py 後 rebuild | 0m0.426s |

**觀察：為什麼 v2 的 rebuild 這麼快？**
- 因為 Docker 在計算快取鍵（Cache Key）時，不只看指令字串，還會去檢查 `COPY` 檔案的內容雜湊。
- 在 v1 中，因為把 `COPY app/ .` 放在 `RUN pip install` 之前，只要修改了 `app.py`，該層的快取就會失效。根據「一層失效，後面連帶全部失效」的快取機制，逼得後續的 `pip install` 每次都要花 5-6 秒重新跑一遍。
- 而 v2 將經常變動的 `app.py` 往後挪，先執行不常變動的 `COPY app/requirements.txt .` 進行套件建置。當再次修改 `app.py` 時，前面安裝套件的幾層通通順利命中快取（CACHED），這讓 rebuild 時間直接從幾秒鐘縮短至 0.4 秒的極速。

## CMD vs ENTRYPOINT 實驗
| 寫法 | `docker run <img>` 輸出 | `docker run <img> extra1 extra2` 輸出 |
|---|---|---|
| CMD shell form | argv = ['show_args.py', 'default1', 'default2'] <br> PID = 8 | docker: Error response from daemon: ... exec: "extra1": executable file not found in $PATH |
| CMD exec form | argv = ['show_args.py', 'default1', 'default2'] <br> PID = 1 | docker: Error response from daemon: ... exec: "extra1": executable file not found in $PATH |
| ENTRYPOINT + CMD | argv = ['show_args.py', 'default1', 'default2'] <br> PID = 1 | argv = ['show_args.py', 'extra1', 'extra2'] <br> PID = 1 |

結論：
1. **Shell Form (PID 8)** 跑起來時，系統會先召喚 `/bin/sh -c` 再 fork 出 Python 程式，導致 Python 主程式的 PID 不是 1，這會使它無法接收到外部作業系統傳來的 SIGTERM 訊號，造成容器無法優雅關閉（Graceful Shutdown）。而 **Exec Form (PID 1)** 能直上 PID 1。
2. 當我們在終端機加上外部參數 `extra1 extra2` 時，不論是 Shell 還是 Exec Form 的 `CMD` 寫法，整條預設命令都會被完全覆蓋，進而噴出找不到執行檔的錯誤。
3. 只有 **ENTRYPOINT + CMD** 的組合最穩健：`ENTRYPOINT` 固定執行主程式，而 `CMD` 則靈活提供預設值。當外部接續參數時，只會精準覆蓋掉 `CMD` 的部分並當成 args 傳給主程式，這才是生產環境最標準的容器化行為。

## Multi-stage 大小對照
| Image | SIZE |
|---|---|
| python:3.12（builder base） | 1.62GB |
| python:3.12-slim（runtime base） | 179MB |
| myapp:v2（單階段） | 197MB |
| myapp:multi（多階段） | 184MB |

**解釋：builder stage 的 layer 去哪了？**
- Builder 階段的 Layers 沒有不見，它們依然做為組建快取（Build Cache）存放在主機的硬碟中。不過，在最終標記與產出 `myapp:multi` 映像檔時，Docker 只會採用最後一個 runtime stage 的基礎底層，並單純透過 `COPY --from=builder` 把需要的執行檔與 site-packages 搬過來。這成功地將高達 1.62GB 完整版 Python 映像檔中的 GCC 編譯工具、標頭檔及各種垃圾，徹底隔絕在最終的 runtime 映像檔之外。

## .dockerignore 故障注入
| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| du -sh . | 24K | 151M | 151M |
| build context 傳輸大小 | 1.25kB | 150.4MB | 1.84kB |
| build 時間 | 0.35s | 4.82s | 0.38s |

## 排錯紀錄
- **症狀**：在寫完 `Dockerfile.multi` 後嘗試執行容器，發現一啟動就立刻崩潰，查看 `docker logs` 顯示 `Permission denied: '/app/app.py'`。
- **診斷**：這是因為我們在 Dockerfile 中使用了 `USER appuser` 切換為非 root 安全身份執行，但在那之前，透過 `COPY app/ .` 複製進容器的檔案，預設擁有者全部都是 `root`。非 root 的 `appuser` 沒有權限讀取或執行 root 的檔案，導致 Python 核心直接拋出權限拒絕。
- **修正**：在 Dockerfile 的 `USER appuser` 切換身份指令之上，補上權限修正與群組擁有權轉換指令：`RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app`。
- **驗證**：重新 `docker build` 並啟動容器，使用 `curl` 測試可正常回傳 Hello 訊息，且執行 `docker exec <container> whoami` 確切回傳執行身份為 `appuser`，成功排除。

## 設計決策
**Q: 為什麼 runtime 選 `python:3.12-slim` 而不是 `alpine`？**
- 雖然 Alpine 體積只有區區幾十 MB，但它底層使用的是 `musl libc`，而多數 Python 複雜套件在 PyPI 上提供的預編譯封裝都是針對常見的 `glibc`（Debian/Ubuntu 體系）。
- 如果選用 Alpine，會導致未來在安裝某些二進位套件時，pip 找不到對應的 Wheel，被迫在容器內從頭下載 `gcc` 與 `musl-dev` 進行現地編譯。這不僅會極大地拉長 build 的時間，編譯過程更會吃掉大量記憶體，甚至容易噴出不可預期的 C 語言編譯錯誤。
- 因此，折衷選用基於 Debian 且移除不必要工具的 `slim` 版本，是在「映像檔體積」與「套件生態系穩定度」之間最合理的生產環境工程取捨。