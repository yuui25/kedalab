# 判断の地図（詰まったら最初に戻る・資格演習チューニング）

資格演習は「技術」ではなく**判断**の試験。見えたサービスごとに
「**固定手順を少数だけ試す → 出なければ深追いせず撤退して次へ**」を回す。
技術の辞書（コマンド）は各技術ファイルにある。ここに置くのは**判断**だけ。

> **このツリーは自分で埋めるのが本体。** 他人のツリーを写しても試験本番の緊張下では効かない。
> 各セルは演習を回しながら、自分の経験で上書きしていく（Web 行は粒度の見本。中身は書き換える）。

---

## 貫く不変則（最重要・ここだけは先に固定）

1. **詰まり＝列挙不足。** 答えはたいてい「既に見つけたのに読んでいない／試していないもの」の中にある。 → 自分の言葉で言い換える: `____`
2. **cred は通貨。** 新しい cred / user 名 / 情報を拾うたびに、**全サービスに試し直す**。 → 自分の言葉で: `____`

---

## 判断プロトコル（高プレッシャー下の人為ミス防止）

- **初期侵入タイムボックス**：1ターゲットあたり `____` 分で foothold が出なければ**撤退して次のターゲットへ**（ラビットホール対策）。
- **撤退した経路は「保留」に記録**し、全ターゲット一巡してから戻る。
- **詰まったら離席（拡散モード）**：PC を物理的に離れ、戻ったら**既取得データ（cred / loot / enum 出力）を読み直す**。昇格・前進のヒントは既に手元にあることが多い。
- **休憩は判断精度を守る投資**。焦り・視野狭窄（トンネリング）を感じたら、進めるより休む。

---

## ① 初期侵入ツリー：ポート → 固定手順（3つまで）→ 撤退条件

| ポート / サービス | 固定で試す手順（3つまで）| 撤退条件（次へ行く基準）| kedalab |
|---|---|---|---|
| 80/443 Web | **記入例**: ①whatweb+ソース+robots ②dir/vhost fuzz ③アプリ名+版→searchsploit | **記入例**: 版特定済み & 既知 CVE 無し & 明らかな入力点無しを確認したら保留 | `Web_Recon_Flow.md` |
| 21 FTP | `____` | `____` | `../02_Initial_Access/FTP.md` |
| 22 SSH（出口寄り）| `____`（cred 入手後に使う／版はユーザー列挙 CVE 用）| `____` | `../02_Initial_Access/SSH.md` |
| 25 SMTP | `____`（VRFY/RCPT でユーザー名列挙）| `____` | `../02_Initial_Access/Mail_Services.md` |
| 139/445 SMB | `____`（null session / 共有 / 版→CVE / ユーザー収集）| `____` | `../01_Reconnaissance/SMB_Enumeration.md` |
| 88/389/636 Kerb/LDAP | `____`（ユーザー列挙→AS-REP / 匿名バインド）| `____` | `Windows_AD_Attack_Flow.md` |
| 161/udp SNMP | `____`（community `public` で内部丸見え・UDP は忘れがち）| `____` | `../01_Reconnaissance/SNMP_Enumeration.md` |
| 3306/1433/5432 DB | `____`（初期 cred→MSSQL なら xp_cmdshell）| `____` | `../02_Initial_Access/MSSQL_Exploitation.md` |
| 5985 WinRM（出口）| `____`（cred 入手後のシェル取得口）| `____` | `../02_Initial_Access/WinRM.md` |
| 3389 RDP（出口）| `____`（cred で入る）| `____` | `../02_Initial_Access/RDP.md` |

> **出口（SSH / WinRM / RDP）は単体で初期侵入にならない。** cred を別サービスで拾ってから使う。ここに来て「cred が無い」なら ① の列挙に戻る。

---

## ③④ 侵入後ツリー：ローカル列挙 → 昇格

入ったら即、自動列挙を回して**出力を読む**（詰まりの大半はここの読み落とし）。

| OS | 回すもの | 昇格の撤退条件 | kedalab |
|---|---|---|---|
| Linux | `____`（id / sudo -l / SUID / cron / cap / 書込可 / 設定内 cred / 内部待受 / 版）→ linpeas | `____` | `../03_Post_Access_Linux/Enumeration_Checklist.md` |
| Windows | `____`（whoami /priv・/groups / systeminfo / サービス / タスク / AlwaysInstallElevated / 保存 cred）→ winpeas | `____` | `../04_Post_Access_Windows_AD/Enumeration_Checklist.md` |

> **昇格は閃きではなく消し込み。** peas が赤く出した行を1つ選んで潰す。出なければ列挙の読み落とし＝③に戻る。

---

## ⑥ AD 横展開（1回目に落ちやすい地点）

AD セットの**初期侵入**ツリーを特に厚く埋める（ラビットホールの常連）。

- 資格情報ダンプ（secretsdump / mimikatz）→ password spray / Pass-the-Hash → BloodHound で経路 → 次ホストで ① に戻る。
- 撤退条件：`____`
- → `Windows_AD_Attack_Flow.md` Step 4-7

---

## 関連技術

- 前：OS 判定 → [`00_OS_Identification.md`](./00_OS_Identification.md)
- 関連：web が開いていたら → [`Web_Recon_Flow.md`](./Web_Recon_Flow.md)
- 後（OS 別の本流）→ [`Linux_Attack_Flow.md`](./Linux_Attack_Flow.md) / [`Windows_AD_Attack_Flow.md`](./Windows_AD_Attack_Flow.md)
- 侵入後列挙 → [`../03_Post_Access_Linux/Enumeration_Checklist.md`](../03_Post_Access_Linux/Enumeration_Checklist.md) / [`../04_Post_Access_Windows_AD/Enumeration_Checklist.md`](../04_Post_Access_Windows_AD/Enumeration_Checklist.md)
