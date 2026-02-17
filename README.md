# VMproxy

讓 UTM / 虛擬機透過宿主機的代理上網 — 一鍵設定，無需手動修改環境變數。

## 使用情境

```
┌──────────────┐         ┌──────────────┐
│   虛擬機 VM   │ ──────▶ │  宿主機 Host  │ ──────▶ Internet
│  (Guest OS)  │  proxy  │ (代理服務端)   │
└──────────────┘         └──────────────┘
```

在 UTM 等虛擬化環境下，Guest OS 需要透過 Host 上運行的代理（如 Clash、V2Ray 等）來存取外部網路。這組腳本可以快速設定 Guest 的系統代理與 shell 環境變數，指向 Host 的代理端口。

## 包含腳本

| 腳本 | 適用系統 | 系統代理設定方式 |
|---|---|---|
| `macproxy` | macOS | `networksetup` |
| `linuxproxy` | Linux | GNOME (`gsettings`)、KDE (`kwriteconfig5/6`)、純 CLI (環境變數) |

## 安裝

將腳本複製到 `$PATH` 中的目錄：

```bash
# macOS
sudo cp macproxy /usr/local/bin/
sudo chmod +x /usr/local/bin/macproxy

# Linux
sudo cp linuxproxy /usr/local/bin/
sudo chmod +x /usr/local/bin/linuxproxy
```

## 用法

以下以 `macproxy` 為例，`linuxproxy` 指令完全相同。

```bash
# 設定代理位址（預設 192.168.1.1:7890）
macproxy config 192.168.1.100 7890
macproxy config 192.168.1.100:1080

# 啟用 / 關閉代理
macproxy on
macproxy off

# 查看狀態
macproxy status

# 將自動載入寫入 shell 設定檔（新終端自動生效）
macproxy init

# 從 shell 設定檔移除
macproxy uninit
```

執行 `on` 後，需在當前終端手動載入一次：

```bash
source ~/.macproxy_env    # macOS
source ~/.linuxproxy_env  # Linux
```

或執行 `init` 後開新終端，即可自動載入。

## 設定檔

| 檔案 | 用途 |
|---|---|
| `~/.macproxy.conf` / `~/.linuxproxy.conf` | 儲存代理位址與連接埠 |
| `~/.macproxy_env` / `~/.linuxproxy_env` | 代理啟用時產生的環境變數檔，由 shell rc 載入 |

## 注意事項

- `linuxproxy init` 會同時寫入 `.bashrc` 與 `.profile`（或 `.bash_profile`），以確保 SSH 非互動命令也能載入代理變數。
- 代理位址應填寫宿主機在虛擬機網段中的 IP（例如 UTM 預設的 `192.168.64.1` 或自訂的橋接 IP）。
- 宿主機的代理軟體需開啟「允許區域網路連線」。

## Author

藥アイ & ChatGPT & Claude
