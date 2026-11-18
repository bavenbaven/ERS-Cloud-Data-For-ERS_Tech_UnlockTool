# ERS Tech MDM — APK 与密钥加密备份（KV）

这个目录是 `ERS Tech Unlock Tool` 的 MDM 管控 APK 与签名密钥的**离线备份**，
放在云端是为了防止本地硬盘损坏导致密钥丢失。

## 内容

| 路径 | 说明 | 是否敏感 |
|---|---|---|
| `ERS_MDM_Lock_v<ver>.apk` | 自研管控 APK（`com.erstech.mdmlock`） | 否（本来就会装到客户机上） |
| `manifest.json` | 版本 / versionCode / SHA-256 / 证书指纹 | 否 |
| `targets.json` | APK 会锁定/冻结的包清单 | 否 |
| `public.pem` | 授权验签公钥（APK 内置的就是它） | 否 |
| `backup/ers_mdm.jks.ersk` | APK 签名密钥的**加密**容器 | 加密后可公开 |
| `backup/ers_license_private.pem.ersk` | 授权签名私钥的**加密**容器 | 加密后可公开 |
| `backup/backup.json` | 两个容器的算法、哈希、恢复命令 | 否 |

## 为什么加密后可以公开

`.ersk` = `ERSK1` 容器：PBKDF2-HMAC-SHA256（300,000 次）派生 64 字节 →
前 32 字节做 AES-256-CBC 加密，后 32 字节做 HMAC-SHA256 认证（先加密后认证）。
没有口令无法还原；**口令不在本仓库**（本地 `D:\ERS_MDM_KEY_BACKUP\PASSPHRASE.txt` + 密码管理器）。

## 恢复

```powershell
# APK 签名密钥（丢了就无法覆盖升级已安装的手机）
powershell -ExecutionPolicy Bypass -File mdm_app\tools\keystore_backup.ps1 `
    -Action Restore -In mdm_app\backup\ers_mdm.jks.ersk `
    -Out mdm_app\keystore\ers_mdm.jks -Passphrase "<口令>" -Force

# 授权签名私钥（丢了就签不出新卡密）
powershell -ExecutionPolicy Bypass -File mdm_app\tools\keystore_backup.ps1 `
    -Action Restore -In mdm_app\backup\ers_license_private.pem.ersk `
    -Out server\keys\private.pem -Passphrase "<口令>" -Force
```

恢复后用 `backup.json` 里的 `sourceSha256` 核对文件哈希，必须完全一致。

## 更新这个备份

```powershell
# 1) 重新生成加密容器（同一口令）
powershell -ExecutionPolicy Bypass -File mdm_app\tools\keystore_backup.ps1 -Action Backup `
    -In mdm_app\keystore\ers_mdm.jks -Out mdm_app\backup\ers_mdm.jks.ersk -Passphrase "<口令>" -Force
powershell -ExecutionPolicy Bypass -File mdm_app\tools\keystore_backup.ps1 -Action Backup `
    -In server\keys\private.pem -Out mdm_app\backup\ers_license_private.pem.ersk -Passphrase "<口令>" -Force

# 2) 同步到本目录并提交
robocopy mdm_app\backup cloud_dist\mdm\backup /E /NFL /NDL /NJH /NJS
copy data\mdm\ERS_MDM_Lock_v*.apk cloud_dist\mdm\
copy mdm_app\releases\manifest.json cloud_dist\mdm\
copy data\mdm\targets.json cloud_dist\mdm\
copy server\keys\public.pem cloud_dist\mdm\
```
