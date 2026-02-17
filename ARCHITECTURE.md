# VMproxy 架構文件

開發參考用，記錄內部設計決策與實作細節。

## 腳本結構

```
vmproxy
├── A. 全域變數 + OS 偵測
├── B. 共用函數
│   ├── load_config / save_config     設定檔讀寫
│   ├── sed_i                         跨平台 sed -i wrapper
│   ├── test_host_reachable           TCP 端口連通性測試
│   ├── get_local_ip                  取得本機 IP
│   ├── scan_proxy_server             LAN 代理掃描
│   ├── config_proxy                  設定代理位址（含 auto 模式）
│   ├── detect_rc_file / get_init_files  Shell 設定檔偵測
│   └── SHELL_HOOK                    寫入 rc 的 source 片段
├── C. macOS 專用函數
│   ├── detect_services               偵測活躍網路介面
│   ├── mac_enable_proxy              networksetup 啟用
│   ├── mac_disable_proxy             networksetup 關閉
│   └── mac_show_status               networksetup 狀態
├── D. Linux 專用函數
│   ├── detect_de                     偵測桌面環境
│   ├── detect_interfaces             偵測活躍網路介面
│   ├── gnome_proxy_on/off/status     gsettings
│   ├── kde_proxy_on/off/status       kwriteconfig5/6
│   ├── linux_enable/disable/show     桌面環境調度
├── E. 調度函數
│   ├── enable_proxy                  啟用（含 fallback 掃描）
│   ├── disable_proxy                 關閉
│   ├── show_status                   狀態（含連通性檢查）
│   ├── init_shell / uninit_shell     Shell hook 管理
├── F. Doctor — 診斷與維護
│   ├── resolve_self_path             腳本自身路徑解析
│   ├── run_diagnosis                 全面診斷（設定檔/hook/連通性/版本）
│   ├── show_doctor_menu              互動式選單（依診斷結果動態顯示）
│   ├── do_update                     更新（下載 + 語法檢查 + 備份 + 覆蓋）
│   ├── do_rollback                   回滾（從 .bak 還原）
│   ├── do_cleanup                    清理環境（關代理 + 移 hook + 刪設定）
│   ├── do_fix_hook                   安裝 shell hook（呼叫 init_shell）
│   └── doctor                        子指令分發
└── G. 主入口                          case 分發
```

## 端口變數

| 變數 | 用途 | 來源 |
|---|---|---|
| `MIXED_PORT` | 混合端口，CLI 唯一操作的端口變數 | config 檔 / CLI 設定 |
| `HTTP_PORT` | HTTP/HTTPS 代理端口 | 預設 = MIXED_PORT |
| `SOCKS_PORT` | SOCKS 代理端口 | 預設 = MIXED_PORT |

**設計決策：**
- CLI (`vmproxy config`) 只操作 `MIXED_PORT`，`HTTP_PORT`/`SOCKS_PORT` 自動跟隨
- `save_config()` 寫入 `MIXED_PORT`，並以註解模板提示 `HTTP_PORT`/`SOCKS_PORT`
- `load_config()` 讀取所有三個變數，`HTTP_PORT`/`SOCKS_PORT` 缺省時 fallback 到 `MIXED_PORT`
- 用戶若需分端口，手動編輯 `~/.vmproxy.conf` 取消註解即可

**config 檔範例 (`~/.vmproxy.conf`)：**
```bash
PROXY_HOST="192.168.1.100"
MIXED_PORT="7890"

# 若 HTTP 與 SOCKS 需要不同端口，取消註解並修改：
# HTTP_PORT="7890"
# SOCKS_PORT="7891"
```

**各處使用：**
- `networksetup`：HTTP/HTTPS 用 `HTTP_PORT`，SOCKS 用 `SOCKS_PORT`
- `gsettings`：同上
- `kwriteconfig`：同上
- `env 檔`：`http_proxy`/`https_proxy` 用 `HTTP_PORT`，`all_proxy` 用 `SOCKS_PORT`
- `test_host_reachable` / `scan_proxy_server`：用 `MIXED_PORT`（代表主端口）

## LAN 掃描流程

`scan_proxy_server(port, range)` 兩階段掃描：

```
Phase 1: ARP 快掃
  arp -a → 解析所有 IP → 排除自己
    ├─ ≤30 台 → 並行 nc -z 測試
    └─ >30 台 → 提示用戶選擇
         ├─ 繼續掃描
         └─ 手動設定（中止）
  找到 → 回傳 IP

Phase 2: 子網全掃（ARP 未命中時）
  get_local_ip → 取得本機 IP → 推算 /24 前綴
  --range 參數覆蓋掃描範圍
  並行 nc -z 掃描 → 找到 → 回傳 IP
```

**stdout/stderr 分離：** 找到的 IP 輸出到 stdout，狀態訊息輸出到 stderr。呼叫端用 `$(scan_proxy_server ...)` 只捕獲 IP。

**並行實作：** 子 shell + `&` 背景執行，結果寫入 `$tmpdir/` 臨時檔，`wait` 後依序檢查。`trap ... RETURN` 自動清理。

**--range 格式：**
- `192.168.1.1-50` → 掃描 192.168.1.1 到 .50
- `1-50` → 使用本機 IP 的前三段 + .1 到 .50

## enable_proxy fallback 流程

```
load_config
  │
  ├─ test_host_reachable(PROXY_HOST, MIXED_PORT)
  │    ├─ 可達 → 正常啟用
  │    └─ 不可達 →
  │         scan_proxy_server(MIXED_PORT)
  │           ├─ 找到 → 臨時使用找到的 IP（不自動儲存）
  │           │         提示: vmproxy config <found_ip> 儲存
  │           └─ 找不到 → 報錯退出
  │
  ├─ 呼叫 mac/linux_enable_proxy
  ├─ 寫入 ~/.vmproxy_env
  └─ 完成
```

**注意：** fallback 掃描找到的 IP 只用於本次啟用，不自動寫入 config。`vmproxy config auto` 才會儲存。

## test_host_reachable 優先順序

1. `nc -z -w 2` — macOS 內建，Linux 大多有
2. `ncat -z -w 2` — nmap 套件附帶
3. `/dev/tcp` bash 內建 — 最後手段

## Shell init 多檔寫入

為解決 SSH 非互動命令 (`ssh host 'cmd'`) 不讀 `.bashrc` 的問題：

| Shell | 互動式 RC | 登入 Profile |
|---|---|---|
| bash | `.bashrc` | `.bash_profile` > `.bash_login` > `.profile`（只讀第一個存在的）|
| zsh | `.zshrc` | `.zprofile` |

`init_shell()` 同時寫入兩個檔案，`uninit_shell()` 同時清除。

## Doctor 子系統

`vmproxy doctor` 提供一站式維護工具，整合診斷、更新、回滾、清理。

### 設計決策

- `init` 保留頂層：安裝流程常用（install → init → config → on）
- `uninit` + `uninstall` 合併為 `doctor cleanup`：功能重疊（uninit 是 uninstall 的子集），"cleanup"（清理環境）比 "uninstall"（不會刪除腳本本身）更準確
- `update` / `rollback` 併入 doctor：版本管理屬於維護範疇

### 互動流程

```
vmproxy doctor
  │
  ├─ 1. 診斷（始終執行）
  │     ├─ 設定檔 ~/.vmproxy.conf
  │     ├─ 環境變數檔 ~/.vmproxy_env
  │     ├─ Shell hook（grep rc 檔）
  │     ├─ 代理連通性（test_host_reachable）
  │     ├─ 外網連通性（curl UPDATE_URL）
  │     └─ 版本比對（外網可達時下載遠端版本）
  │
  ├─ 2. 選單（僅在有可操作項時）
  │     ├─ u) 更新（UPDATE_AVAILABLE）
  │     ├─ r) 回滾（BAK_EXISTS）
  │     └─ f) 安裝 hook（HOOK_MISSING）
  │
  └─ 3. 執行用戶選擇
```

### 診斷 (run_diagnosis)

依序檢查六項，每項輸出 ✓/✗ 狀態，同時設定 flag 供選單判斷：

| 檢查項 | 方法 | Flag |
|---|---|---|
| 設定檔 | `[[ -f ~/.vmproxy.conf ]]` | — |
| 環境變數檔 | `[[ -f ~/.vmproxy_env ]]` | — |
| Shell hook | `grep -qF vmproxy_env` rc 檔 | `HOOK_MISSING=1` |
| 代理連通性 | `test_host_reachable` | — |
| 外網連通性 | `curl --max-time 3 $UPDATE_URL` | `INTERNET_OK=1` |
| 版本比對 | 下載遠端檔取 VERSION（外網可達時） | `UPDATE_AVAILABLE=1`、`BAK_EXISTS=1` |

外網連通性測試直接 curl `UPDATE_URL`（GitHub raw），一次請求同時確認外網可達與取得遠端版本。

### 更新流程 (do_update)

```
下載到暫存檔 → 取版本號 → 比對 → bash -n 語法檢查 → 備份 .bak → 覆蓋（自動判斷 sudo）
```

失敗時提示 `vmproxy on` 啟用代理；成功時提示 `vmproxy doctor rollback`。

### 回滾流程 (do_rollback)

`resolve_self_path()` 取得腳本路徑 → 檢查 `.bak` 存在 → 覆蓋回去（自動判斷 sudo）→ 顯示回滾後版本號。

### 路徑解析 (resolve_self_path)

`command -v "$SCRIPT_NAME"` 優先（PATH 中的實際路徑），fallback 到 `$0`。

## 跨平台差異

| 功能 | macOS (Darwin) | Linux |
|---|---|---|
| `sed -i` | `sed -i '' ...` | `sed -i ...` |
| 系統代理 | `networksetup` | `gsettings` / `kwriteconfig` |
| 網路介面偵測 | `networksetup -listallnetworkservices` | `ip link show` |
| 本機 IP | `route -n get default` + `ipconfig getifaddr` | `ip -4 route get 1.1.1.1` |
| ARP 表 | `arp -a` (BSD 格式) | `arp -a` (類似格式) |

## config_proxy 參數判斷

```
$2（第一個參數）
  ├─ "auto"           → 掃描模式
  ├─ 純數字 ^[0-9]+$  → 只改端口
  ├─ 含 ":"           → host:port 格式
  ├─ 含 "."           → IP 位址（$3 可選端口）
  └─ 其他             → 當作 hostname（$3 可選端口）
```
