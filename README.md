# VMproxy

讓虛擬機透過宿主機的代理上網 — 一鍵設定，無需手動修改環境變數。

## 使用情境

```
┌──────────────┐         ┌──────────────┐
│   虛擬機 VM   │ ──────▶ │  宿主機 Host  │ ──────▶ Internet
│  (Guest OS)  │  proxy  │ (代理服務端)   │
└──────────────┘         └──────────────┘
```

在 UTM 等虛擬化環境下，Guest OS 需要透過 Host 上運行的代理（如 Clash、V2Ray 等）來存取外部網路。`vmproxy` 可以快速設定 Guest 的系統代理與 shell 環境變數，指向 Host 的代理端口。

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
# 設定代理位址（預設 192.168.1.1:7890）
vmproxy config 192.168.1.100 7890
vmproxy config 192.168.1.100:1080

# 啟用 / 關閉代理
vmproxy on
vmproxy off

# 查看狀態
vmproxy status

# 將自動載入寫入 shell 設定檔（新終端自動生效）
vmproxy init

# 從 shell 設定檔移除
vmproxy uninit
```

執行 `on` 後，需在當前終端手動載入一次：

```bash
source ~/.vmproxy_env
```

或執行 `init` 後開新終端，即可自動載入。

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
vmproxy config <你的宿主機IP> <port>
vmproxy init
vmproxy on
```

## 注意事項

- `vmproxy init` 會同時寫入多個 shell 設定檔（`.bashrc` + `.bash_profile`，或 `.zshrc` + `.zprofile`），確保 SSH 非互動命令也能載入代理變數。
- 代理位址應填寫宿主機在虛擬機網段中的 IP（例如 UTM 預設的 `192.168.64.1` 或自訂的橋接 IP）。
- 宿主機的代理軟體需開啟「允許區域網路連線」。

## Author

藥アイ & ChatGPT & Claude
