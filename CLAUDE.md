# Ansible 範本規則（必須遵守）

這個 repo 是 Openfind 技術服務處的 Ansible 範本。完整規範在 `docs/ansible-template-spec.md`，以下是 AI 必須遵守的摘要。產出仍由人依規範第 11 章審查。

## 結構
- 產品 role 只有五個：m2k、mg、mo、mb、ssp。不可新增 mg_install、mg_patch 這類依工作拆分的 role。
- 產品 role 的 task 檔只用這些名稱：install.yml、patch.yml、cert.yml、weakscan.yml、check.yml。該產品沒有的工作就不建那個檔，不可自創別的名稱。
- 兩個以上產品共用的功能放通用 role（os_baseline、web_cert、weakscan），通用 role 不可寫死產品路徑。
- playbook 依工作命名，每個產品一個 play，用 include_role + tasks_from 呼叫產品 role。
- 會中斷服務的 play 必須有 serial 與 max_fail_percentage: 0。
- SSP 換憑證方式與其他產品不同，尚未定義：不可替 SSP 建 cert.yml，也不可讓 SSP 呼叫 web_cert。

## 變數
- 產品 role 變數加產品前綴（m2k_、mg_、mo_、mb_、ssp_），通用 role 加 role 名前綴。
- role 用到的變數都要在 defaults/main.yml 給預設值，必填變數寫進 meta/argument_specs.yml。
- 不可在 role 的 vars/ 放客戶會改的值。
- 密碼變數加 vault_ 前綴，只放在客戶的 group_vars/all/vault.yml。
- defaults 裡值為 "TODO" 的項目代表尚未確認，不可自行填入猜測的值；要用到時先問人。

## 禁止
- 不可在程式 repo 寫入真實客戶的主機名、IP、帳號、密碼；範例用 192.0.2.0/24 與 example.com。
- 不可用 shell、command 執行 sed、echo 修改設定檔，改用 lineinfile、template、blockinfile。
- 不可自行猜測產品的安裝參數、路徑、服務指令；不確定就寫 TODO 註解並提出問題。
- 不可用獨立 task 每次都重啟服務，重啟一律用 handler。

## 完成前
- 模組一律用 FQCN（例如 ansible.builtin.lineinfile）。
- 執行下列檢查，有錯修到通過：
  ```bash
  ansible-lint
  for pb in playbooks/*.yml; do ansible-playbook -i inventories/_template "$pb" --syntax-check; done
  ```
- 不可對客戶 inventory 執行不帶 --check 的 playbook。
- 列出改了哪些檔案、還有哪些 TODO 要人確認。

## 開新專案或補齊範本時
- 先讀 `docs/ansible-template-spec.md` 與各 role 的 README.md，沿用現有結構，不重新設計。
- 新客戶的 inventory 從 `inventories/_template/` 複製到 customer-inventories repo，不放在這個 repo。
