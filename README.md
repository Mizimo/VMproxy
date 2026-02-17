# VMproxy

讓虛擬機透過區域網路中的代理伺服器上網 — 一鍵設定，支援自動偵測。

## 使用情境

```
┌──────────────┐         ┌──────────────┐
│   虛擬機 VM   │ ──────▶ │  代理伺服器    │ ──────▶ Internet
│  (Guest OS)  │  proxy  │ (Host 或 LAN) │
└──────────────┘         └──────────────┘
```

在 UTM 等虛擬化環境下，Guest OS 需要透過區域網路中的代理（如 Clash、V2Ray 等）來存取外部網路。`vmproxy` 可以快速設定 Guest 的系統代理與 shell 環境變數，指向代理伺服器的端口。

當網路環境變動（例如從朋友家到自己家），可透過 `vmproxy config auto` 重新掃描 LAN 找到代理伺服器。

## 支援平台

| Guest OS | 系統代理設定方式 |
|---|---|
| macOS | `networksetup` |
| Linux | GNOME (`gsettings`)、KDE (`kwriteconfig5/6`)、純 CLI (環境變數) |

腳本透過 `uname -s` 自動偵測當前系統，無需手動選擇。

## 安裝

`vmproxy` 無法自動將自己加入 PATH，請手動安裝。

### 方法一：複製到系統目錄

```bash
sudo cp vmproxy /usr/local/bin/
sudo chmod +x /usr/local/bin/vmproxy
```

### 方法二：複製到個人目錄

```bash
mkdir -p ~/.local/bin
cp vmproxy ~/.local/bin/
chmod +x ~/.local/bin/vmproxy
```

若 `~/.local/bin` 不在 PATH 中，需在 shell 設定檔加入：

```bash
export PATH="$HOME/.local/bin:$PATH"
```

## 用法

```bash
# 將自動載入寫入 shell 設定檔（新終端自動生效）
vmproxy init

# 設定代理位址（預設 192.168.1.1:7890）
vmproxy config 192.168.1.100 7890
vmproxy config 192.168.1.100:1080
vmproxy config 7890                    # 只改端口

# 自動偵測區域網路中的代理伺服器
vmproxy config auto                    # 掃描整個 LAN
vmproxy config auto 7891               # 用指定端口掃描
vmproxy config auto --range 192.168.1.1-50  # 限定掃描範圍

# 啟用 / 關閉代理
vmproxy on
vmproxy off

# 查看狀態（含即時連通性檢查）
vmproxy status

# 診斷、更新、維護
vmproxy doctor                           # 互動式：全面診斷 + 選單
vmproxy doctor update                    # 更新至最新版本
vmproxy doctor rollback                  # 回滾至上一版本
vmproxy doctor cleanup                   # 清理環境（還原網路 + 移除設定）
```

執行 `on` 後，需在當前終端手動載入一次：

```bash
source ~/.vmproxy_env
```

或執行 `init` 後開新終端，即可自動載入。

## 自動偵測

`vmproxy on` 啟用時會先測試已設定的代理伺服器是否可達。若連線失敗，自動掃描區域網路：

1. **ARP 快掃** — 從 ARP 表取得已知設備，並行測試代理端口
2. **子網全掃** — ARP 未命中時，掃描整個 /24 子網（並行 `nc -z`，約 2 秒完成）

若 ARP 表中設備超過 30 台（例如公司網路），會提示選擇繼續掃描或手動設定。

也可以手動觸發掃描：

```bash
vmproxy config auto
```

## 設定檔

| 檔案 | 用途 |
|---|---|
| `~/.vmproxy.conf` | 儲存代理位址與連接埠 |
| `~/.vmproxy_env` | 代理啟用時產生的環境變數檔，由 shell rc 載入 |

## 從舊版遷移 (macproxy / linuxproxy)

如果之前使用過 `macproxy` 或 `linuxproxy`：

```bash
# 1. 移除舊的 shell hook
macproxy uninit    # 或 linuxproxy uninit

# 2. 刪除舊的設定檔
rm -f ~/.macproxy.conf ~/.macproxy_env
rm -f ~/.linuxproxy.conf ~/.linuxproxy_env

# 3. 安裝 vmproxy 後重新設定
vmproxy config <代理伺服器IP> <port>
vmproxy init
vmproxy on

# 如果之後需要完整清理 vmproxy:
vmproxy doctor cleanup
```

## 注意事項

- `vmproxy init` 會同時寫入多個 shell 設定檔（`.bashrc` + `.bash_profile`，或 `.zshrc` + `.zprofile`），確保 SSH 非互動命令也能載入代理變數。
- 代理伺服器需開啟「允許區域網路連線」。
- VM 需使用橋接網路（與代理伺服器在同一個 LAN）才能使用自動偵測功能。

## Author

藥アイ & ChatGPT & Claude
