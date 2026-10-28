# openfind-ansible

Openfind 技術服務處的 Ansible 範本：五個產品（M2K、MG、MO、MB、SSP）、所有客戶共用同一套 roles 與 playbooks。
客戶的主機、差異設定與加密過的密碼放在另一個 repo `customer-inventories`，執行時用 `-i` 指定客戶。

設計規則見 [`docs/ansible-template-spec.md`](docs/ansible-template-spec.md)；給 AI 的規則在 [`CLAUDE.md`](CLAUDE.md)，
Claude Code 在這個目錄啟動時會自動讀取。

## 結構

```text
openfind-ansible/
├── CLAUDE.md                 # 給 AI 的規則
├── docs/                     # 完整設計規範
├── ansible.cfg
├── roles/
│   ├── m2k/ mg/ mo/ mb/ ssp/ # 產品 role：install、patch、cert、check
│   ├── web_cert/             # 通用：Apache 換憑證
│   ├── os_baseline/          # 通用：NTP、系統更新
│   └── weakscan/             # 通用：作業系統層弱掃（sshd 演算法）
├── playbooks/                # 依工作分：install、patch、renew_cert、healthcheck、weakscan_fix、os_baseline
├── inventories/_template/    # 客戶 inventory 範本，只有假資料
└── .github/workflows/        # CI：ansible-lint 與 syntax-check
```

## 準備 control node

```bash
# OEL／RHEL 9
sudo dnf install -y python3.12 python3.12-pip
python3.12 -m venv ~/ansible-venv && source ~/ansible-venv/bin/activate
pip install ansible-core ansible-lint

# 兩個 repo 放在同一層
git clone https://github.com/openfind-ts/openfind-ansible.git
git clone <customer-inventories 的網址> customer-inventories
```

## 執行

一律在 repo 根目錄執行，先預演、讀過 diff，再正式執行。

```bash
cd openfind-ansible

# 預演：只跑客戶 A 的 MG 換憑證
ansible-playbook -i ../customer-inventories/customer-a playbooks/renew_cert.yml \
  --limit mg --ask-vault-pass --check --diff

# 讀過 diff 確認無誤，正式執行
ansible-playbook -i ../customer-inventories/customer-a playbooks/renew_cert.yml \
  --limit mg --ask-vault-pass

# 只挑 staging 環境的 MG
--limit 'mg:&staging'
```

## 新增客戶

1. 把 `inventories/_template/` 複製到 `customer-inventories/<客戶名>/`。
2. 改 `hosts.yml`；客戶沒有的產品就刪掉那個群組和對應的 `group_vars` 檔。
3. 密碼用 `ansible-vault create group_vars/all/vault.yml` 建立，私鑰用 `ansible-vault encrypt files/<產品>.key` 加密後才 commit。
4. vault 密碼不進任何 repo，由負責該客戶的同仁保管。

## 各產品目前狀態

| 產品 | install / patch | cert（換憑證） | check |
| --- | --- | --- | --- |
| M2K | TODO | 待補 SSL 設定檔路徑 | 可用 |
| MG | TODO | 可用 | 可用 |
| MO | TODO | 待補 SSL 設定檔路徑 | 可用 |
| MB | TODO | 待補 SSL 設定檔路徑 | 可用 |
| SSP | TODO | 換憑證方式不同，尚未定義 | 待補安裝路徑 |

`defaults/main.yml` 裡值為 `"TODO"` 的項目，用到時 role 會直接停下來報錯，不會帶著錯誤的值修改主機。

## 修改前

```bash
ansible-lint
for pb in playbooks/*.yml; do ansible-playbook -i inventories/_template "$pb" --syntax-check; done
```

push 後 GitHub Actions 也會自動跑這兩項檢查。合併前請依規範第 11 章的審查清單逐項確認。
