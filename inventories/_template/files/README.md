客戶專屬檔案（例如憑證）放這裡。私鑰一定要先 `ansible-vault encrypt` 再 commit。
程式 repo 的範本這裡不放任何真實檔案。
各產品的 patch 放 `patch/<產品>/`，例如 `patch/m2k/`、`patch/mo/`，並在 `group_vars/<產品>.yml` 設定 `<產品>_patch_dir`。
