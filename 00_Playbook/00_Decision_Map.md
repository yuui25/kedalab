# 判断の地図（決定木・上から機械的に実行する）

**使い方**：各ノードの「打つ」を実行 → 「結果」のどれに当たるか見て → 指定された**次ノードへ飛ぶ**。
判断はこの表がする。あなたは**実行と分岐だけ**。結果欄に無いものが出たら、それがバグ（分岐の抜け）なので記録する。

> **メモ用語**：`TODO`＝まだ着手していない open ポートの一覧。`保留`＝時間切れで撤退した経路（一巡後に戻る）。`cred`＝ユーザー名／パスワード／ハッシュ。

---

## メインループ（これを回す）

1. **N0** を実行（スキャン）
2. **N1** で open ポートを各ノードに割り当て、`TODO` を作る
3. `TODO` を優先順（**web → 情報が取れる口 → 出口**）に1つ取り、担当ノードへ
4. 1ノード **20〜30 分**で何も出なければ `保留` に記録して 3 へ（次の `TODO`）
5. foothold（シェル）取れたら → **P1**
6. 新しい cred を拾ったら（どのノードでも）→ 即 **CRED**
7. `TODO` 空 & シェル無し → **S1**（詰まり時）
8. シェル複数 & AD 環境 → **A1**（横展開）

---

## N0 — スキャン（IP を渡されたら最初に打つ）

```bash
nmap -sC -sV -oA nmap_quick [TARGET]                       # クイック
sudo nmap -p- --min-rate 5000 -oA nmap_allports [TARGET]   # 全ポート
sudo nmap -sU --top-ports 20 -oA nmap_udp [TARGET]         # UDP（SNMP 等）
searchsploit --nmap nmap_allports.xml                      # 版→既知CVE
```
**結果**：常に → **N1**

---

## N1 — 割り当て（観測 → どのノードへ）

open ポートを下表で担当ノードに割り当て、`TODO` に積む。OS と /etc/hosts もここで確定。

| 観測 | 示唆 | 割り当て先 |
|---|---|---|
| 80 / 443 / 8080 | web | **W1**（最優先）|
| 21 | FTP | **F1** |
| 139 / 445 | SMB | **B1** |
| 88 / 389 / 636 | Kerberos/LDAP＝**AD 確定** | **L1** |
| 161/udp | SNMP | **M1** |
| 3306 / 1433 / 5432 | DB | **D1** |
| 5985 / 3389 / 22 | 出口（cred 要）| **X1**（後回し）|
| `Service Info: OS: Windows` / 445 / 3389 / 5985 | Windows | OS=Win 控える |
| `http-title: Did not follow redirect to http://[名前]/` | vhost 名 | `echo '[TARGET] [名前]' \| sudo tee -a /etc/hosts` して再アクセス |
| title/`http-server` に製品名+版 | CVE 候補 | `searchsploit [製品名] [版]`（`--nmap` が拾わない）|
| open が 2〜3 個だけ | 面が狭い＝深掘り | 「少ない＝楽」ではない。各版 `searchsploit` 必須 |

→ メインループ 3 へ

---

## W1 — Web（80/443）

**打つ**：`whatweb http://[TARGET]` ＋ ブラウザでソース ＋ `curl -s http://[TARGET]/robots.txt`
**結果**：
- 製品名+版が出た → `searchsploit [製品名] [版]` → CVE あり → その exploit（詳細 `Web_Recon_Flow.md`）／無し → 次へ
- ログインフォーム/入力点/API がある → `Web_Recon_Flow.md`（SQLi・LFI・upload・IDOR 等の分岐）
- robots/dir に隠しパス（`feroxbuster -u http://[TARGET]`）→ 見つけた物を**読む**（止めない）
- 20〜30 分で何も無い → `保留` → メインループ 4

詳細な場合分け → `Web_Recon_Flow.md`

---

## F1 — FTP（21）

**打つ**：`nmap -p21 --script ftp-anon [TARGET]`（or `ftp [TARGET]` → `anonymous`）
**結果**：
- 匿名で入れた & ファイルあり → 全 DL → 中に cred → **CRED**／webroot と同一か判定（`../02_Initial_Access/FTP.md` §2）
- 書き込み可 & webroot 一致 → webshell 設置 → ブラウザ実行 → **P1**
- 匿名不可 → 版 → `searchsploit`。CVE 無し → `保留` → メインループ 4

詳細 → `../02_Initial_Access/FTP.md`

---

## B1 — SMB（139/445）

**打つ**：`nxc smb [TARGET] -u '' -p ''` ＋ `smbclient -L //[TARGET]/ -N`
**結果**：
- 共有が読めた → 中の設定/スクリプト/cred を回収 → cred あれば **CRED**
- ユーザー名が取れた（`--users` / RID brute）→ ユーザーリスト化（spray 用）
- 版が古い（例 EOL）→ `searchsploit`（EternalBlue 等）
- `STATUS_ACCESS_DENIED` / null 拒否 & 共有無 → `保留` → メインループ 4

詳細 → `../01_Reconnaissance/SMB_Enumeration.md`

---

## L1 — Kerberos/LDAP（88/389/636）＝AD

**打つ**：`nxc ldap [TARGET]` ＋ `ldapsearch -x -H ldap://[TARGET] -s base`
**結果**：
- 匿名バインドで情報が出た → ドメイン名/ユーザー列挙
- ユーザー名が取れた → AS-REP：`impacket-GetNPUsers [DOMAIN]/ -usersfile users.txt -no-pass` → ハッシュ出たらクラック → **CRED**
- 何も出ない → `保留`。cred を得たら `Windows_AD_Attack_Flow.md` 本流へ

詳細 → `Windows_AD_Attack_Flow.md`

---

## M1 — SNMP（161/udp・忘れがち）

**打つ**：`snmpwalk -v2c -c public [TARGET]`
**結果**：
- 応答あり → プロセス/ユーザー/内部情報/cred 片を回収 → cred あれば **CRED**
- `public` 不可 → `onesixtyone -c community.txt [TARGET]` で community brute → 当たれば再 walk
- 全滅 → `保留` → メインループ 4

詳細 → `../01_Reconnaissance/SNMP_Enumeration.md`

---

## D1 — DB（3306/1433/5432）

**打つ**：`mysql -h [TARGET] -u root` ／ `impacket-mssqlclient [USER]@[TARGET]` ／ `psql -h [TARGET] -U postgres`
**結果**：
- デフォルト/初期 cred で入れた → MSSQL なら `xp_cmdshell` でコマンド実行 → **P1**／他 DB は中の cred を回収 → **CRED**
- 認証通らない → 別で拾った cred を試す（**CRED** 経由）／無ければ `保留`

詳細 → `../02_Initial_Access/MSSQL_Exploitation.md`

---

## X1 — 出口（SSH 22 / WinRM 5985 / RDP 3389）

**前提**：ここは cred が無いと動かない。`cred` を持っていない場合は着手せず、他ノードで cred を取ってから来る。
**打つ**（cred 取得後）：
- WinRM：`nxc winrm [TARGET] -u [USER] -p [PASS]` → `(Pwn3d!)` なら `evil-winrm -i [TARGET] -u [USER] -p [PASS]`
- SSH：`ssh [USER]@[TARGET]`
- RDP：`xfreerdp /u:[USER] /p:[PASS] /v:[TARGET]`

**結果**：
- シェル取得 → **P1**
- 認証通る が `(Pwn3d!)` 無し（WinRM）→ 別ユーザー/グループ経路（`../02_Initial_Access/WinRM.md`）
- cred がまだ無い → メインループ 3（別ノードで cred を探す）

---

## CRED — 新しい cred/user を拾ったら（最重要サブルーチン・不変則②）

**毎回必ず呼ぶ**。cred は通貨。1つ拾ったら**全ホスト×全出口**に試し直す。

**打つ**：
```bash
nxc smb   [TARGETS] -u [USER] -p [PASS]
nxc winrm [TARGETS] -u [USER] -p [PASS]
nxc ldap  [TARGETS] -u [USER] -p [PASS]
# ハッシュなら -H、SSH は手で ssh [USER]@[TARGET]
```
**結果**：
- `(Pwn3d!)` → その出口でシェル（**X1**）→ **P1**
- `[+]` だけ（権限不足）→ そのサービスにログインは可、シェル不可。別ノードの昇格材料に
- 全部 `[-]` → cred を保管（後で別ホストに流用）→ 呼び出し元へ戻る

---

## P1 — 侵入後（ローカル列挙 → 昇格）

**打つ**：
- Linux：`id; sudo -l; find / -perm -4000 -type f 2>/dev/null` → `./linpeas.sh`
- Windows：`whoami /priv & whoami /groups & systeminfo` → `.\winPEAS.exe`

**結果**：
- peas が赤行を出した → その昇格ベクタを**1つ**潰す（SeImpersonate→Potato / SUID / sudo / unquoted path / AlwaysInstallElevated）→ 詳細 `../03_Post_Access_Linux/Enumeration_Checklist.md` / `../04_Post_Access_Windows_AD/Enumeration_Checklist.md`
- 設定ファイル/履歴/メモリに cred 発見 → **CRED**
- root/SYSTEM 取れた → loot 回収 → **CRED**（他ホスト用）→ AD なら **A1**
- 何も出ない → **S1**（列挙の読み落とし）

---

## A1 — AD 横展開

**打つ**：`nxc smb [TARGETS] -u [USER] -p [PASS]`（拾った cred を全ホストに）→ `(Pwn3d!)` を探す
**結果**：
- `(Pwn3d!)` あり → `secretsdump` で cred ダンプ → **CRED** → 次ホストで **P1**
- 経路が見えない → BloodHound で最短経路を確認 → ACE/委任を辿る
- 全滅 → **S1**

→ `Windows_AD_Attack_Flow.md` Step 4-7

---

## S1 — 詰まり時（必ずここに戻る・不変則①）

**順に機械的に確認**（詰まり＝列挙不足。答えは既に手元にある）：

1. **離席**（拡散モード）。戻って**既取得データ（cred/loot/enum 出力）を読み直す**
2. 未処理の `TODO`/`保留` は無いか → あれば担当ノードへ
3. **UDP/全ポートの取りこぼし**は無いか → **N0** を再実行
4. 拾った cred を**全出口に試したか** → **CRED** を再実行
5. それでも無ければ、各サービスの**版を深掘り**（exploit-db の該当 PoC を読む）

---

## 関連技術

- 前：OS 判定 → [`00_OS_Identification.md`](./00_OS_Identification.md)
- 関連：web が開いていたら → [`Web_Recon_Flow.md`](./Web_Recon_Flow.md)
- 後（OS 別の本流）→ [`Linux_Attack_Flow.md`](./Linux_Attack_Flow.md) / [`Windows_AD_Attack_Flow.md`](./Windows_AD_Attack_Flow.md)
- 侵入後列挙 → [`../03_Post_Access_Linux/Enumeration_Checklist.md`](../03_Post_Access_Linux/Enumeration_Checklist.md) / [`../04_Post_Access_Windows_AD/Enumeration_Checklist.md`](../04_Post_Access_Windows_AD/Enumeration_Checklist.md)
