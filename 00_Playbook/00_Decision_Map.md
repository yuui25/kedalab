# 判断の地図（詰まったら最初に戻る・資格演習チューニング）

資格演習は「技術」ではなく**判断**の試験。見えたサービスごとに
「**最初の一手を打つ → 固定手順を少数試す → 出なければ深追いせず撤退して次へ**」を回す。
各行に**最初に打つコマンド**を置く。詳しいオプション・分岐は各技術ファイルへ。

---

## 貫く不変則（最重要）

1. **詰まり＝列挙不足。** 答えはたいてい「既に見つけたのに読んでいない／試していないもの」の中にある。
2. **cred は通貨。** 新しい cred / user 名 / 情報を拾うたびに、**全サービスに試し直す**。

---

## 判断プロトコル（高プレッシャー下の人為ミス防止）

- **初期侵入タイムボックス**：1ターゲット **目安 20〜30 分**（自分で調整）。foothold が出なければ**撤退して次へ**（ラビットホール対策）。
- **撤退した経路は「保留」に記録**し、全ターゲット一巡してから戻る。
- **詰まったら離席（拡散モード）**：PC を物理的に離れ、戻ったら**既取得データ（cred / loot / enum 出力）を読み直す**。昇格・前進のヒントは既に手元にあることが多い。
- **休憩は判断精度を守る投資**。焦り・視野狭窄（トンネリング）を感じたら、進めるより休む。

---

## ① 初期侵入ツリー：ポート → 最初のコマンド → 続けて試す / 撤退

| ポート / サービス | 最初に打つコマンド | 続けて試す → 撤退条件 | kedalab |
|---|---|---|---|
| 80/443 Web | `whatweb http://[TARGET]` ＋ ブラウザでソース ＋ `curl -s http://[TARGET]/robots.txt` | dir/vhost fuzz（`feroxbuster -u http://[TARGET]`）→ アプリ名+版で `searchsploit`。**版特定済 & 既知 CVE 無 & 明らかな入力点無**を確認したら保留 | `Web_Recon_Flow.md` |
| 21 FTP | `nmap -p21 --script ftp-anon [TARGET]`（or `ftp [TARGET]` → `anonymous`）| 読めたら全 DL・版→`searchsploit`。書込可なら webroot 判定。匿名不可＆版に CVE 無で保留 | `../02_Initial_Access/FTP.md` |
| 22 SSH（出口）| `nc -nv [TARGET] 22`（バナーのみ）| **単体では入口にしない。** cred 入手後 `ssh [USER]@[TARGET]`。版が古ければユーザー列挙 CVE | `../02_Initial_Access/SSH.md` |
| 25 SMTP | `smtp-user-enum -M RCPT -U users.txt -D [DOMAIN] -t [TARGET]` | VRFY/EXPN/RCPT でユーザー名収集 → spray 用リストへ。列挙不可で保留 | `../02_Initial_Access/Mail_Services.md` |
| 139/445 SMB | `nxc smb [TARGET] -u '' -p ''` ＋ `smbclient -L //[TARGET]/ -N` | `enum4linux-ng -A [TARGET]`・共有読取・版→既知 CVE。null 不可＆共有無で保留 | `../01_Reconnaissance/SMB_Enumeration.md` |
| 88/389/636 Kerb/LDAP | `nxc ldap [TARGET]` ＋ `ldapsearch -x -H ldap://[TARGET] -s base` | user 列挙 → AS-REP `impacket-GetNPUsers [DOMAIN]/ -usersfile users.txt -no-pass`・匿名バインド → AD 本流へ | `Windows_AD_Attack_Flow.md` |
| 161/udp SNMP | `snmpwalk -v2c -c public [TARGET]`（**UDP・忘れがち**）| 当たれば内部情報/プロセス/cred 片が丸見え。`public` 不可なら `onesixtyone` で community brute | `../01_Reconnaissance/SNMP_Enumeration.md` |
| 3306/1433/5432 DB | `mysql -h [TARGET] -u root` ／ `impacket-mssqlclient [USER]@[TARGET]` ／ `psql -h [TARGET] -U postgres` | デフォルト/初期 cred → MSSQL は `xp_cmdshell` でコマンド実行。認証通らず保留 | `../02_Initial_Access/MSSQL_Exploitation.md` |
| 5985 WinRM（出口）| `nxc winrm [TARGET] -u [USER] -p [PASS]` | `(Pwn3d!)` なら `evil-winrm -i [TARGET] -u [USER] -p [PASS]`。**cred 無ければ ① に戻る** | `../02_Initial_Access/WinRM.md` |
| 3389 RDP（出口）| `nmap -p3389 --script rdp-ntlm-info [TARGET]` | cred 入手後 `xfreerdp /u:[USER] /p:[PASS] /v:[TARGET]` | `../02_Initial_Access/RDP.md` |

> **出口（SSH / WinRM / RDP）は単体で初期侵入にならない。** cred を別サービスで拾ってから使う。ここで「cred が無い」なら ① の列挙に戻る。

---

## ③④ 侵入後ツリー：ローカル列挙 → 昇格

入ったら即、自動列挙を回して**出力を読む**（詰まりの大半はここの読み落とし）。

| OS | 最初に打つコマンド | 続けて試す → 撤退条件 | kedalab |
|---|---|---|---|
| Linux | `id; sudo -l; find / -perm -4000 -type f 2>/dev/null` → `./linpeas.sh` | sudo/SUID/cap/cron/書込可/設定内 cred を1つ潰す。出なければ内部待受 `ss -tlnp` と設定ファイルの読み落とし＝③へ | `../03_Post_Access_Linux/Enumeration_Checklist.md` |
| Windows | `whoami /priv & whoami /groups & systeminfo` → `.\winPEAS.exe` | SeImpersonate→Potato / unquoted path / AlwaysInstallElevated / 保存 cred を1つ潰す。出なければ保存 cred・パッチ差分の読み落とし | `../04_Post_Access_Windows_AD/Enumeration_Checklist.md` |

> **昇格は閃きではなく消し込み。** peas が赤く出した行を1つ選んで潰す。出なければ列挙の読み落とし＝③に戻る。

---

## ⑥ AD 横展開（1回目に落ちやすい地点）

- 最初のコマンド：`nxc smb [TARGETS] -u [USER] -p [PASS]`（拾った cred を全ホストに）→ `(Pwn3d!)` を探す
- 続けて：secretsdump で cred ダンプ → password spray / Pass-the-Hash → BloodHound で経路 → 次ホストで ① に戻る
- 撤退条件：spray も PtH も BloodHound 経路も空振りなら、現ホストの loot（設定 / ブラウザ / メモリ）に別 cred が無いか＝列挙に戻る
- → `Windows_AD_Attack_Flow.md` Step 4-7

---

## 関連技術

- 前：OS 判定 → [`00_OS_Identification.md`](./00_OS_Identification.md)
- 関連：web が開いていたら → [`Web_Recon_Flow.md`](./Web_Recon_Flow.md)
- 後（OS 別の本流）→ [`Linux_Attack_Flow.md`](./Linux_Attack_Flow.md) / [`Windows_AD_Attack_Flow.md`](./Windows_AD_Attack_Flow.md)
- 侵入後列挙 → [`../03_Post_Access_Linux/Enumeration_Checklist.md`](../03_Post_Access_Linux/Enumeration_Checklist.md) / [`../04_Post_Access_Windows_AD/Enumeration_Checklist.md`](../04_Post_Access_Windows_AD/Enumeration_Checklist.md)
