# Ansible 範本設計規範

Openfind 技術服務處 · 2026-10-28

## 1. 目的與適用範圍

本規範定義技術服務處所有 Ansible 自動化的目錄結構、role 設計、inventory 規劃與檢查方式。目標是讓五個產品、所有客戶環境共用同一套程式，新人或 AI 產生的範本長得都一樣，任何人接手都看得懂。

**適用對象**：所有撰寫或修改 Ansible 的同仁，以及同仁用來產生、修改 Ansible 的 AI 工具（Claude Code 等）。AI 產出的內容視同人寫的內容，一樣要符合本規範。

**涵蓋產品**

| 產品 | 代號 | role 與群組名稱 | 變數前綴 |
| --- | --- | --- | --- |
| Mail2000 | M2K | `m2k` | `m2k_` |
| MailGates | MG | `mg` | `mg_` |
| MailAudit | MO | `mo` | `mo_` |
| MailBase | MB | `mb` | `mb_` |
| SecuSharePro | SSP | `ssp` | `ssp_` |

名稱一律小寫，role 名稱、inventory 群組名稱、變數前綴三者相同。

## 2. 核心原則

後面每一章都是這五條的展開，遇到規範沒寫到的情況，回到這五條判斷。

1. **程式與客戶資料分開**：roles 與 playbooks 只寫一份、所有客戶共用；客戶的主機、帳號、差異設定只放在客戶 inventory。
2. **一個產品一個 role**：同一產品的安裝、patch、換憑證、檢查，都在同一個 role 裡，依工作拆成不同 task 檔。
3. **重複的功能抽成通用 role**：兩個以上產品都要做的事（換憑證、NTP、弱掃修補），寫成通用 role，由產品 role 帶參數呼叫。
4. **role 只放標準值，差異放 inventory**：所有客戶都一樣的值寫在 role 的 `defaults`，客戶不同的寫在客戶的 `group_vars`、`host_vars`。
5. **先驗證再上機**：任何變更都要通過 `ansible-lint`、`--syntax-check`、`--check --diff`，由人讀過差異才執行，對客戶環境更是如此。

## 3. Repo 結構

程式和客戶資料分成兩個 repo，都放 GitHub 的 `openfind-ts` 組織，權限分開控管：

| Repo | 權限 | 放什麼 |
| --- | --- | --- |
| `openfind-ts/openfind-ansible` | 公開（public） | roles、playbooks、inventory 範本，只有假資料 |
| `openfind-ts/customer-inventories` | 私有（private），只開給負責的同仁 | 每個客戶一個資料夾：主機、差異設定、加密過的密碼與憑證 |

**程式 repo：`openfind-ansible`**

```text
openfind-ansible/
├── CLAUDE.md               # 給 AI 的規則（第 10 章）
├── README.md
├── ansible.cfg
├── docs/                   # 本規範
├── roles/
│   ├── os_baseline/        # 通用：NTP、sshd、帳號、系統更新
│   ├── web_cert/           # 通用：Apache 換憑證
│   ├── weakscan/           # 通用：弱掃修補項目
│   ├── m2k/                # 產品 role
│   ├── mg/
│   ├── mo/
│   ├── mb/
│   └── ssp/
├── playbooks/              # 依「工作」分，不依產品分
│   ├── install.yml
│   ├── patch.yml
│   ├── renew_cert.yml
│   ├── weakscan_fix.yml
│   ├── healthcheck.yml
│   └── os_baseline.yml
├── inventories/
│   └── _template/          # 客戶 inventory 範本，只放假資料
└── .github/workflows/      # CI：lint 與 syntax-check
```

**客戶 repo：`customer-inventories`**

```text
customer-inventories/
├── customer-a/
├── customer-b/
└── lab/                    # 公司內部測試環境
```

規則：

- 程式 repo 是公開的，**不可出現任何真實客戶的主機名、IP、帳號、密碼**；範例一律用 `192.0.2.0/24`、`example.com` 這類保留用的假資料。
- 新增客戶時，從 `inventories/_template/` 複製一份到客戶 repo 再修改。
- 執行時在程式 repo 根目錄，用 `-i` 指向客戶 repo 的資料夾。

## 4. Role 設計

role 分成兩類：**產品 role**（五個，一個產品一個）與**通用 role**（跨產品共用的功能）。不得依工作拆成 `mg_install`、`mg_patch` 這類多個 role。

### 4.1 產品 role 的標準結構

五個產品 role 一律使用下列結構與檔名，該產品沒有的工作就不建那個檔，但不可自創別的名稱替代。

```text
roles/mg/
├── README.md               # 這個 role 做什麼、目前狀態
├── defaults/main.yml       # 產品標準值：路徑、執行帳號、重啟指令
├── meta/
│   ├── main.yml
│   └── argument_specs.yml  # 宣告每個 task 檔接受的變數
├── tasks/
│   ├── main.yml            # 只 include install.yml
│   ├── install.yml         # 安裝
│   ├── patch.yml           # 更新到指定版本
│   ├── cert.yml            # 換憑證（呼叫 web_cert）
│   ├── weakscan.yml        # 產品層的弱掃修補
│   └── check.yml           # 健康檢查，只讀不改
├── handlers/main.yml       # 重啟產品服務
├── templates/
└── files/
```

| task 檔 | 用途 | 規則 |
| --- | --- | --- |
| `install.yml` | 全新安裝 | 可重複執行，已安裝就不重裝 |
| `patch.yml` | 更新到 `<產品>_version` 指定的版本 | 已是該版本就跳過 |
| `cert.yml` | 換憑證 | 不自己寫換憑證邏輯，呼叫 `web_cert` |
| `weakscan.yml` | 產品自身的弱掃項目 | 作業系統層的項目放 `weakscan` 通用 role |
| `check.yml` | 服務、版本、憑證到期日檢查 | 只讀不改，任何時候都能安全執行 |

重啟服務一律寫成 handler，由 task 用 `notify` 觸發；不可用一個獨立 task 每次都重啟。

SSP 的換憑證方式與其他產品不同，尚未定義，所以 SSP 沒有 `cert.yml`，也不使用 `web_cert`。

### 4.2 通用 role

| role | 負責 | 使用產品 |
| --- | --- | --- |
| `os_baseline` | NTP、sshd 設定、管理帳號、作業系統更新 | 全部主機 |
| `web_cert` | 上傳憑證、修改 Apache SSL 設定、觸發重啟 | M2K、MG、MO、MB |
| `weakscan` | 作業系統層弱掃項目（SSH 演算法、OpenSSL 等） | 全部主機 |

通用 role 不可寫死任何產品的路徑或指令，全部由產品 role 帶入。例如 MG 的 `tasks/cert.yml`：

```yaml
- name: 更換 MailGates 憑證
  ansible.builtin.include_role:
    name: web_cert
  vars:
    web_cert_crt_src: "{{ mg_cert_src }}"
    web_cert_key_src: "{{ mg_key_src }}"
    web_cert_conf_dir: "{{ mg_conf_dir }}"
    web_cert_ssl_conf: "{{ mg_ssl_conf }}"
    web_cert_include: "{{ mg_ssl_include }}"
    web_cert_owner: "{{ mg_cert_owner }}"
    web_cert_restart_cmd: "{{ mg_restart_cmd }}"
```

新增一個產品要用的功能時，先確認其他產品是否也需要：兩個以上產品會用到，就做成通用 role。

### 4.3 產品版本差異

同一產品的新舊版本，用 `<產品>_version` 變數在 task 裡判斷，不可複製一份舊版 role。

```yaml
- name: 設定新版才有的參數
  ansible.builtin.lineinfile:
    path: "{{ mg_conf_dir }}/httpd.conf"
    line: "..."
  when: mg_version is version('6.0', '>=')
```

## 5. Playbook 設計

playbook 依「要做的工作」命名，不依產品命名。一個 playbook 裡每個產品一個 play，用 `hosts` 指定產品群組，用 `tasks_from` 指定要執行的 task 檔。

```yaml
# playbooks/patch.yml
- name: 更新 MG
  hosts: mg
  become: true
  serial: 1
  max_fail_percentage: 0
  tasks:
    - name: 執行 MG patch
      ansible.builtin.include_role:
        name: mg
        tasks_from: patch

- name: 更新 M2K
  hosts: m2k
  become: true
  serial: 1
  max_fail_percentage: 0
  tasks:
    - name: 執行 M2K patch
      ansible.builtin.include_role:
        name: m2k
        tasks_from: patch

# MO、MB、SSP 照同樣格式各一個 play
```

規則：

- **會中斷服務的工作**（patch、換憑證、重開機）一律加 `serial` 分批，並設 `max_fail_percentage: 0`，任一台失敗就停。`serial` 是 play 層級的設定，只能寫在 playbook，不能寫在 role 裡。
- playbook 只負責「哪些主機、用哪個 role、怎麼分批」，實際的步驟都寫在 role 裡。
- 不得在 playbook 寫死主機名、IP 或客戶名稱，用 `--limit` 挑選主機。

常用執行方式：

```bash
# 先預演
ansible-playbook -i ../customer-inventories/customer-a playbooks/patch.yml --check --diff
# 只跑 staging 環境的 MG
ansible-playbook -i ../customer-inventories/customer-a playbooks/patch.yml --limit 'mg:&staging'
```

## 6. 客戶 Inventory 規劃

每個客戶一個資料夾，結構固定如下。主機同時歸屬「產品群組」與「環境群組」兩種，兩者獨立。

```text
customer-a/
├── hosts.yml
├── files/                # 客戶專屬檔案，例如憑證；私鑰須 ansible-vault 加密
├── group_vars/
│   ├── all/
│   │   ├── main.yml      # 連線帳號、NTP 等所有主機共用設定
│   │   └── vault.yml     # 密碼，一定要用 ansible-vault 加密
│   ├── mg.yml            # 這個客戶的 MG 差異
│   └── m2k.yml
└── host_vars/
    └── mg01.yml          # 單台主機的例外
```

**群組命名規則**

| 類別 | 群組名稱 | 說明 |
| --- | --- | --- |
| 產品 | `m2k`、`mg`、`mo`、`mb`、`ssp` | 與 role 名稱相同，不可自訂別名 |
| 環境 | `prod`、`staging`、`dr` | 依客戶實際有的環境建立 |

```yaml
# hosts.yml
all:
  children:
    mg:
      hosts:
        mg01:
          ansible_host: 192.0.2.10
        mg02:
          ansible_host: 192.0.2.11
    m2k:
      hosts:
        m2k01:
          ansible_host: 192.0.2.20
    prod:
      hosts:
        mg01:
        m2k01:
    staging:
      hosts:
        mg02:
```

規則：

- 主機名稱用「產品代號＋流水號」，實際位址寫在 `ansible_host`。
- 客戶沒有的產品，就不建那個群組，也不建對應的 `group_vars` 檔。
- 一台主機裝了兩個產品時，同時放進兩個產品群組。
- 客戶的憑證放客戶資料夾的 `files/`，在 `group_vars/<產品>.yml` 用 `"{{ inventory_dir }}/files/<檔名>"` 指定。

## 7. 變數分層與命名規則

變數只能放在下表四個位置之一。前三列由上到下優先權遞增，下面的會覆蓋上面的；vault.yml 只放機密，不拿來覆蓋其他值。

| 位置 | 放什麼 | 例子 |
| --- | --- | --- |
| role 的 `defaults/main.yml` | 產品標準值，所有客戶都一樣 | `mg_conf_dir: /mailgates/httpd/conf` |
| 客戶 `group_vars/<產品>.yml` | 這個客戶該產品的差異 | `mg_version`、`mg_cert_src` |
| 客戶 `host_vars/<主機>.yml` | 單台主機例外 | 特殊安裝路徑 |
| 客戶 `group_vars/all/vault.yml` | 密碼、金鑰 | `vault_mg_admin_password` |

規則：

- **產品 role 的變數一律加產品前綴**（`m2k_`、`mg_`、`mo_`、`mb_`、`ssp_`），通用 role 加 role 名稱前綴（例如 `web_cert_`）。避免不同 role 的變數互相覆蓋。
- **role 用到的變數都要在 `defaults/main.yml` 給預設值**，沒有合理預設值的（例如密碼），在 `meta/argument_specs.yml` 標為 required。
- **尚未確認的標準值寫成 `"TODO"`**，role 用到時要先檢查並報錯停止，不可帶著猜測的值執行。
- **不可在 role 的 `vars/` 放客戶會改的值**。`vars/` 優先權高於 inventory，放在那裡客戶就覆蓋不了。
- **密碼變數加 `vault_` 前綴**，再在一般變數裡引用，例如 `mg_admin_password: "{{ vault_mg_admin_password }}"`，搜尋變數時一眼看得出哪些是機密。

`meta/argument_specs.yml` 範例：

```yaml
argument_specs:
  patch:
    short_description: 更新 MG 到指定版本
    options:
      mg_version:
        type: str
        required: true
        description: 目標版本，例如 6.2
```

## 8. 機密管理

密碼、金鑰、API token 只能以 ansible-vault 加密後放在客戶 repo，任何地方都不可出現明碼。

- 每個客戶一個 `group_vars/all/vault.yml`，用 `ansible-vault create` 建立、`ansible-vault edit` 修改。
- 客戶憑證的私鑰用 `ansible-vault encrypt files/<檔名>.key` 整檔加密後才 commit；`copy` 模組上傳時會自動解密。
- **vault 密碼不進 repo**：由負責該客戶的同仁保管，執行時用 `--ask-vault-pass` 輸入，或放在個人電腦上權限 600 的檔案，用 `--vault-password-file` 指定。
- 不同客戶使用不同的 vault 密碼。
- SSH 私鑰放在執行者的 `~/.ssh`，不放進任何 repo；客戶專用金鑰在客戶的 `group_vars/all/main.yml` 用 `ansible_ssh_private_key_file` 指定路徑。
- 會印出機密的 task 加 `no_log: true`；上傳私鑰的 task 加 `diff: false`，避免 `--diff` 印出私鑰內容。

## 9. 品質檢查

每一筆變更都要通過下列檢查才能合併、才能對客戶主機執行。

| 關卡 | 指令 | 何時跑 |
| --- | --- | --- |
| 風格與常見錯誤 | `ansible-lint`（profile: production） | 提交前；CI 自動跑 |
| 語法 | `ansible-playbook -i inventories/_template <playbook> --syntax-check` | 提交前；CI 自動跑 |
| 預演 | `ansible-playbook <playbook> --check --diff` | 每次對客戶主機執行前 |
| idempotency | 同一 playbook 連跑兩次，第二次 `changed=0` | 在 lab 環境驗證新 role 時 |

規則：

- 程式 repo 設 GitHub Actions，每次 push 自動跑 `ansible-lint` 與 `--syntax-check`，沒過不可合併。
- 新的 role 或大幅修改，先在 `lab` inventory 的測試機跑過、驗證 idempotency，才用在客戶環境。
- 修改設定檔用 `lineinfile`、`template`、`blockinfile` 等模組，不可用 `shell`、`command` 執行 `sed`、`echo`。必須用 `command`、`shell` 時，要加 `changed_when` 或 `creates`，讓結果可以重複執行。
- 改重要設定檔時加 `backup: true`；有語法檢查工具的設定檔（例如 sshd_config）加 `validate`。

## 10. 給 AI 的規則

用 AI 產生或修改 Ansible 時，AI 必須遵守程式 repo 根目錄的 [`CLAUDE.md`](../CLAUDE.md)（Claude Code 啟動時會自動讀取；其他 AI 工具請貼在提示詞開頭）。產出仍由人依第 11 章審查。

## 11. 審查檢查清單

合併前，審查者逐項確認；任一項不符就退回修改。

**結構**

- [ ] 沒有新增依工作拆分的產品 role
- [ ] task 檔名稱符合第 4.1 節
- [ ] 兩個以上產品共用的功能已抽成通用 role
- [ ] 會中斷服務的 play 有 `serial` 與 `max_fail_percentage: 0`

**變數與機密**

- [ ] 變數前綴正確，role 用到的變數都有預設值或列在 argument_specs
- [ ] 程式 repo 內沒有真實客戶資料
- [ ] 密碼只在加密的 vault.yml，會印出機密的 task 有 `no_log: true`

**寫法**

- [ ] 模組使用 FQCN
- [ ] 沒有用 shell、command 改設定檔
- [ ] 重啟服務用 handler
- [ ] 重要設定檔有 `backup`，可驗證的有 `validate`

**驗證**

- [ ] CI 的 ansible-lint 與 syntax-check 通過
- [ ] 在 lab 環境跑過，第二次執行 `changed=0`
- [ ] AI 留下的 TODO 都已由人確認或處理

## 12. 產品標準值

各產品 role 的 `defaults/main.yml` 已填入下列值。標 TODO 的項目請各產品負責人確認後補上；補齊前，用到這些值的 task 會停下來報錯。

| 產品 | 安裝路徑 | 執行帳號 | Web 重啟指令 | 使用 web_cert | SSL 設定檔 |
| --- | --- | --- | --- | --- | --- |
| M2K | `/webmail` | webmail | `/webmail/tools/m2kctrl httpd restart` | 是 | TODO |
| MG | `/mailgates` | root | `/mailgates/httpd/bin/apachectl restart` | 是 | `conf/extra/mg_ssl.conf` |
| MO | `/mailgates` | root | `/mailgates/httpd/bin/apachectl restart` | 是 | TODO |
| MB | `/webmail` | root | `/webmail/httpd/bin/apachectl restart` | 是 | TODO |
| SSP | TODO | TODO | TODO | 否，換憑證方式不同 | — |

其他待決定：

- [ ] 是否將程式 repo 打包成內部 collection（例如 `openfind.products`），用版本號發佈
- [ ] 客戶 repo 的權限粒度：整個 repo 一組權限，或每個客戶一個 repo
