# Netexec（nxc）/ CrackMapExec（cme）クイックリファレンス

## 概要

netexec（`nxc`）は CrackMapExec（`cme`）の後継ツール。SMB・WinRM・LDAP 等の複数プロトコルで認証確認・パスワードスプレー・情報列挙を行える。

Kali Linux 2023.4以降は標準搭載。それ以前の環境では `pipx install netexec` でインストールする。古いKaliでは `crackmapexec`（`cme`）コマンドが使える場合があるが構文が一部異なる。

---

## 環境前提

| 環境 | コマンド | インストール方法 |
|------|---------|----------------|
| Kali 2023.4以降 | `nxc` | 標準搭載（インストール不要） |
| それ以前のKali | `nxc` | `pipx install netexec` |
| 古いKali（CME時代） | `cme` | プリインストール済みの場合あり（構文一部異なる） |

```bash
# バージョン確認
nxc --version
```

---

## よく使うコマンド

### SMB 認証確認

```bash
# 単一ユーザーで認証確認
nxc smb [IP] -u [USER] -p '[PASSWORD]'

# ドメインを指定する場合
nxc smb [IP] -u [USER] -p '[PASSWORD]' -d [DOMAIN]

# ハッシュで認証（Pass-The-Hash）
nxc smb [IP] -u [USER] -H '[NTLM_HASH]'
```

**出力の読み方：**
- `[+]` → 認証成功
- `[-]` → 認証失敗
- `(Pwn3d!)` → 管理者権限あり（PSExec / WMI でコマンド実行可能）

---

### WinRM 認証確認

```bash
# WinRM（5985ポート）で認証確認
nxc winrm [IP] -u [USER] -p '[PASSWORD]'

# [+] [IP] で PWNED! が出れば evil-winrm で接続可能
```

---

### LDAP 認証確認

```bash
# LDAP認証確認（AD環境）
nxc ldap [IP] -u [USER] -p '[PASSWORD]' -d [DOMAIN]
```

---

### パスワードスプレー

```bash
# ユーザーリストに対して単一パスワードを試す
# [Kali] 以下はKali（攻撃側）のマシンで実行する。
nxc smb [IP] -u users.txt -p '[PASSWORD]' --continue-on-success

# 複数パスワードとユーザーリストの組み合わせ
nxc smb [IP] -u users.txt -p passwords.txt --continue-on-success
```

> **注意：** アカウントロックアウトポリシーに注意する。`--continue-on-success` で全ユーザーを試しても、
> ロックアウト閾値を超えると正規ユーザーが締め出される。AD環境では事前にロックアウトポリシーを確認する。

---

### Pass-The-Hash（SMB）

```bash
# NTLM ハッシュで認証
nxc smb [IP] -u [USER] -H '[NTLM_HASH]'

# 管理者ハッシュで Pwn3d! が出た場合、secretsdump 等に進む
```

---

### 共有フォルダの列挙

```bash
# 認証済みで共有一覧を取得
nxc smb [IP] -u [USER] -p '[PASSWORD]' --shares
```

---

## 注意点・落とし穴

- CrackMapExec（`cme`）と netexec（`nxc`）は構文がほぼ同じだが、オプション名が一部変わっている場合がある
- `--continue-on-success` を忘れると最初の成功でスキャンが止まる
- AD環境での大量スプレーはセキュリティログ（イベントID 4625）に記録される

---

## 関連技術
- 前：認証情報の発見 → `../02_Initial_Access/Credential_Discovery.md`
- 後：取得した認証情報でのAD攻略 → `../00_Playbook/Windows_AD_Attack_Flow.md`
