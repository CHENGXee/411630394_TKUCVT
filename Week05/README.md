# W05｜把容器拆開來看：Namespace / Cgroups / Union FS / OCI

## Docker 環境

- Storage Driver: overlay2
- Cgroup Driver: systemd
- Cgroup Version: 2
- Default Runtime: runc

## Namespace 觀察

### 六種 namespace 用途（用自己的話）
- PID：隔離 process 的編號空間。容器內的 process 自己有一套 PID，從 1 開始重算；在 host 上看同一支 process 卻是另一個大數字。兩邊看的是同一個 process，只是視角不同。
- NET：隔離網路裝置、IP 位址、路由表、iptables 規則、port 空間。容器有自己的 eth0 和 lo，跟 host 的網路介面完全分開，不會互相干擾。
- MNT：隔離 mount point 與檔案系統的視圖。容器有自己的根目錄 `/`，看不到 host 原本掛載的磁碟和目錄，反之亦然。
- UTS：隔離 hostname 和 domain name。容器可以有自己的 hostname，改了不影響 host，host 改了也不影響容器。
- IPC：隔離 System V IPC 和 POSIX message queue，也就是共享記憶體和 semaphore。不同容器的程序無法透過這類機制直接通訊，避免意外的跨容器資料交換。
- USER：隔離 UID / GID 的對映關係。容器內的 root（UID 0）可以對應到 host 上的一個非特權使用者，讓容器內的「管理員」在 host 上其實沒有真正的 root 權限，提升安全性。

### Host vs 容器 inode 對照
| Namespace | Host PID 1 inode | 容器 sleep inode | 一樣嗎？ |
|-----------|-----------------|-----------------|---------|
| pid       | 4026531836      | 4026532605      | 否      |
| net       | 4026531833      | 4026532607      | 否      |
| mnt       | 4026531832      | 4026532602      | 否      |
| uts       | 4026531838      | 4026532603      | 否      |
| ipc       | 4026531839      | 4026532604      | 否      |
| user      | 4026531837      | 4026531837      | 是      |

### 容器內 `ps aux` 輸出
```bash
xiu@app:~$ docker exec -it ns-demo ps aux

PID   USER     TIME  COMMAND

    1 root      0:00 sleep 3600

   15 root      0:00 ps aux
```
 **Q: 為什麼只看到自己的 process？**
- 因為容器被放進了一個獨立的 PID namespace。這個 namespace 裡只有容器自己啟動的 process，kernel 不會讓它看到這個 namespace 以外的任何 process。host 上跑了上百個 process，在容器的視角裡全部都「不存在」。sleep 在容器內是 PID 1，但從 host 的 /proc 看，它的 PID 是一個大數字——兩個視角，同一支程式。

## Cgroups 實驗

### 容器內讀到的限制
- memory.max：268435456
- cpu.max：50000 100000

### Host 端對照
- memory.max：268435456
- cpu.max：50000 100000
- memory.current：397312

### OOM 故障三階段
| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m）|
|---|---|---|---|
| 容器 exit code | - | 137| 0|
| OOMKilled | - | true| false|
| dmesg 關鍵字 | 無 OOM | `Memory cgroup out of memory: Killed process ... dd`| 無 OOM |

## Image 分層

### `docker image inspect nginx:1.27-alpine` layer 數量
**共8層**
```bash
"sha256:08000c18d16dadf9553d747a58cf44023423a9ab010aab96cf263d2216b8b350"
"sha256:d71eae0084c1aa823dd8fb2ecf8604d5c0f4911226c042bb1f8297e819f4b192"
"sha256:c56f134d380585340a68d0db2f2c170641a1c0ff72ccf2438cf2f693df756a85"
"sha256:e244aa659f612a80c40dd8645812301e3def6b15ec67b9e486ed2201172b51d1"
"sha256:b8d7d1d2263425d6044e059b2810017d062d659b9b755241f3747eda77726250"
"sha256:811a4dbbf4a5309e4390cf655c12db92e1a4304fb9d9731f83e7b02e95a617c6"
"sha256:947e805a4ac71f68e6703550c0b36c2aa2e554c4fa670ca2da6a25c6d7dccb66"
"sha256:0d853d50b128aa460b47e7121849463a14b18d4fd976caf5014744aae24d28aa"
```

### 兩個同源 image 共享 layer 的證據
**nginx:1.27-alpine**
```bash
"sha256:08000c18d16dadf9553d747a58cf44023423a9ab010aab96cf263d2216b8b350"
"sha256:d71eae0084c1aa823dd8fb2ecf8604d5c0f4911226c042bb1f8297e819f4b192"
"sha256:c56f134d380585340a68d0db2f2c170641a1c0ff72ccf2438cf2f693df756a85"
"sha256:e244aa659f612a80c40dd8645812301e3def6b15ec67b9e486ed2201172b51d1"
"sha256:b8d7d1d2263425d6044e059b2810017d062d659b9b755241f3747eda77726250"
"sha256:811a4dbbf4a5309e4390cf655c12db92e1a4304fb9d9731f83e7b02e95a617c6"
"sha256:947e805a4ac71f68e6703550c0b36c2aa2e554c4fa670ca2da6a25c6d7dccb66"
"sha256:0d853d50b128aa460b47e7121849463a14b18d4fd976caf5014744aae24d28aa"
```

**nginx:1.26-alpine**
```bash
"sha256:08000c18d16dadf9553d747a58cf44023423a9ab010aab96cf263d2216b8b350" -> 相同
"sha256:d71eae0084c1aa823dd8fb2ecf8604d5c0f4911226c042bb1f8297e819f4b192" -> 相同
"sha256:c56f134d380585340a68d0db2f2c170641a1c0ff72ccf2438cf2f693df756a85" -> 相同
"sha256:ed2f467e1cfcfea2cff2f48b21b86e763979ee599591f3632b44899f26ce583b" -> 不相同
"sha256:6f197061abd698a3eaf862a101d043b50b9162024cdf830e7cfb75131a9f3725" -> 不相同
"sha256:51b6aefac2f5df9fa2c24d782ef818b0b96238af2511eb60f79a58d1c839513a" -> 不相同
"sha256:6dba76576010ad0450285be4d174f5084b0bf597a68f31f8ad597fab0f032f3d" -> 不相同
"sha256:a0636672c7fc32af4d1022152a8e32256abd648fb01f48f33023839e65c6d1cb" -> 不相同
```

|符號|意義|說明|
|---|---|---|
|A|Added（新增）|這個路徑在唯讀層裡不存在，是這個容器的可寫層新建的|
|C|Changed（變更）|目錄本身被改過——通常是因為目錄內有檔案被新增、修改或刪除|
|D|Deleted（刪除）|原本在唯讀層裡存在的檔案，被這個容器刪掉了|

### `docker diff` 輸出範例與解讀
```bash
C /etc
C /etc/nginx
C /etc/nginx/conf.d
A /etc/nginx/conf.d/custom.conf
D /etc/nginx/conf.d/default.conf
C /run
A /run/nginx.pid
C /var
C /var/cache
C /var/cache/nginx
A /var/cache/nginx/scgi_temp
A /var/cache/nginx/uwsgi_temp
A /var/cache/nginx/client_temp
A /var/cache/nginx/fastcgi_temp
A /var/cache/nginx/proxy_temp
C /tmp
A /tmp/hello.txt
```

## OCI 呼叫鏈
- **dockerd → containerd → containerd-shim → runc 各自負責什麼**

    每次執行 docker run，這個指令要穿過四到五層軟體才真正把 process 起來：
    - docker CLI：接收你下的指令，透過 Unix socket（`/var/run/docker.sock`）把請求以 HTTP 送給 dockerd。
    - dockerd：Docker 的主要 daemon，負責使用者面的高層邏輯——image build、網路管理（bridge、overlay）、volume 管理、API 服務。它本身不直接管理容器的生命週期，而是透過 gRPC 把請求轉給 containerd。
    - containerd：負責容器的生命週期管理——image 拉取、儲存、快照（snapshot）、以及「建立 / 啟動 / 停止容器」的實際流程。它遵循 OCI Runtime Spec，呼叫 runc 來真正建立容器。
    - containerd-shim（每個容器一支）：在 runc 跑完 clone() 建立容器之後，shim 接手持有容器 process 的 stdio 和 exit code。這樣即使 containerd 重啟，容器本身不會跟著死掉——shim 一直在那邊陪著容器 process。
    - runc：OCI Runtime Spec 的參考實作，真正動手的那個。它讀取 OCI bundle 裡的 config.json，呼叫 Linux 的 clone() 系統呼叫建立各種 namespace，把 cgroup 限制寫進 /sys/fs/cgroup/，然後 exec() 跑起容器的主程式。runc 跑完就退出，容器 process 由 shim 接管。

- **OCI Runtime Spec config.json 的關鍵欄位**

    config.json 是一份 JSON，描述「如何從這個 rootfs 建出一個容器」：
    - process.args：要執行的命令，例如 ["sleep", "3600"]。
    - linux.namespaces：列出要建立哪些 namespace（pid、network、mount、uts、ipc）——這就是隔離的源頭。
    - linux.resources：cgroup 限制，包含 memory、cpu、pids。
    - root.path：容器 rootfs 的路徑，也就是 overlay2 merge 出來的統一視圖。

任何符合 OCI Runtime Spec 的 runtime（runc、crun、kata-runtime、gVisor 的 runsc）看到這份 config.json 都能跑出語意相同的容器——這就是 OCI 標準讓 Docker / Podman / Kubernetes 能互通的原因。

## 排錯紀錄
- 症狀：執行 nsenter -t $CPID -p -u -n -m -i sh 時出現 Operation not permitted，無法進入容器的 namespace。
- 診斷：進入其他 process 的 namespace 等同於跨越隔離邊界，需要 root 權限。忘記加 sudo。
- 修正：改為 sudo nsenter -t $CPID -p -u -n -m -i sh。
- 驗證：成功進入後執行 hostname 和 ip addr，確認看到的是容器內的 hostname 和網路介面，而不是 host 的。

## 想一想
**1. 容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？kill -9 1（在容器內）會發生什麼？**
- 不是同一支。容器裡的 PID 1 是在容器自己的 PID namespace 裡編號為 1 的 process（例如 sleep），host 的 PID 1 是 systemd，它們位於不同的 PID namespace，互不相關。
- 在容器內執行 kill -9 1 的效果取決於容器的主程式：如果主程式沒有特別處理 signal（例如直接跑 sleep），它會被 SIGKILL 殺掉，整個容器跟著退出。但因為 PID namespace 的隔離，這個 kill 不會影響到 host 的 systemd，也不會影響其他容器。這也是為什麼 Docker 建議用 --init 啟動容器——加上一個輕量的 init process（如 tini），讓它正確處理 signal 傳遞和 zombie process 回收，避免 PID 1 被意外殺掉或 signal 沒人接的問題。

**2. 兩個容器都基於 ubuntu:24.04，磁碟空間是吃兩份還是共用？怎麼驗證？**
- 共用。overlay2 用 sha256 雜湊定址 layer，相同內容的 layer 在 /var/lib/docker/overlay2/ 下只存一份，兩個容器的 lowerdir 都指向同一組唯讀目錄。
- 驗證方法：先跑 sudo du -sh /var/lib/docker/overlay2/ 記下初始大小，然後拉第二個同源 image 並啟動容器，再次比較大小——增加量會遠小於 ubuntu:24.04 image 本身（約 78 MB），因為共享底層 layer 根本沒有複製。再用 docker image inspect ubuntu:24.04 --format '{{json .RootFS.Layers}}' 對兩個基於它的 image 做比較，前面幾個 sha256 相同就是共用的直接證據。

**3. 如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？這個限制跟 VM 差在哪？**
- 嚴格說不能。容器的隔離建立在 namespace 和 cgroup 這兩個 kernel 機制上，所有容器共用同一份 host kernel。如果 kernel 本身有 privilege escalation 漏洞（例如 Dirty COW、CVE-2022-0847 等），惡意容器可以利用漏洞逃出 namespace，取得 host 的 root 權限，影響所有容器和 host 本身。
這跟 VM 的差別在於：VM 裡的 Guest OS 有自己完整的 kernel，即使 Guest kernel 被打穿，攻擊者還要再突破 Hypervisor 這層才能影響 host，隔離邊界多一層。
- 這就是為什麼 Kata Containers 和 Firecracker 要存在——它們讓每個容器跑在一個輕量的 VM 裡（有獨立的 Guest kernel），對外仍然用 OCI 介面，兼顧容器的快速啟動和 VM 級別的強隔離。對安全要求高的場景（多租戶雲端、跑不受信任的程式碼），這類強隔離方案比純 namespace 容器更適合。