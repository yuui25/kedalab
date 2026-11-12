# Web 初動フロー（web ポートが開いていたら最初に開く）

OS を問わず、Web ポート（80 / 443 / 8080 等）が開いていたら**ここから始める**。
Web 列挙はシェル取得でも網羅診断でも共通の初動で、OS 依存ではない。
ここで**何が見えたか**に応じて各経路へ分岐する。

> **なぜ OS 判定より先に web を見るのか:** 認証情報を持たない段階では、Web が唯一または最短の初期侵入面になることが多い（経験則）。WinRM（5985）/ SSH（22）/ RDP（3389）等は**認証情報を入手した後の経路**なので、web を飛ばして単一ポートの技術ファイルに直行しない。
> → この取りこぼしの詳細: [`00_OS_Identification.md`](./00_OS_Identification.md) の「判定結果に応じた次のステップ」内の警告。

---

## 着火条件

- ポートスキャンで `80` / `443` / `8080` / `8443` / `3000` / `8888` 等に HTTP(S) を検出（**OS 問わず・非標準ポート含む**）
- OS 判定が Windows でも Linux でも、web が開いていればまずここを通す

> **80/443 のみ開いていて SSH 等が無い場合も、このフローを進める。** Web 脆弱性を通じたシェル取得が主経路になる。
> **非標準ポート（3000, 8080, 8443, 8888 等）の HTTP も同じフロー。** `nmap -sC -sV` がバナーからアプリ名を識別した場合（例：`Grafana` / `Jenkins` / `Gitea`）は、**即そのアプリ名 + バージョンで `searchsploit` / NVD を検索**する。

---

## 目的で読み分け（最初の分岐）

| 目的 | この先 |
|------|--------|
| **シェル取得**（ペネトレ・シェル取得目的）| このファイルを上から進める。cred / 脆弱性が出たら下の「見えたもの → 次の経路」で分岐 |
| **Web 脆弱性の網羅的洗い出し**（Web 診断スコープ・シェル取得しない）| [`Web_Vuln_Flow.md`](./Web_Vuln_Flow.md) へ |

---

## Step 1 — アクセス前の確認（/etc/hosts）

> **【先に確認】nmap の出力に `Location: http://[ドメイン名]/` やホスト名らしき文字列が含まれる場合は `/etc/hosts` に追記してからアクセスする。**
> ```bash
> # [Attacker] /etc/hosts への追記（IP とドメイン名は環境に合わせて変更）
> echo '[IP] [ドメイン名]' | sudo tee -a /etc/hosts
> ```
> IP 直打ちと `/etc/hosts` 追記後のドメイン名アクセスで画面が変わる場合（302 リダイレクト・別コンテンツ等）は、ドメイン名でのアクセスを基本にする。
> → 検出の詳細: [`../01_Reconnaissance/Network_Scanning.md`](../01_Reconnaissance/Network_Scanning.md)（観点・着眼点テーブル）

---

## Step 2 — web 初動列挙（上から順に）

1. ブラウザでトップページを確認 → 使用技術・フレームワーク・エンドポイントの把握
2. **`/robots.txt` を確認する（最初の一手）**
   - `Disallow:` エントリが「隠しパス」の地図になる
   - 管理画面・CMS インストールパスが見つかることがある
   - nmap `-sC` でもスキャン結果に自動表示される
   → 詳細: [`../01_Reconnaissance/Web_Enumeration.md`](../01_Reconnaissance/Web_Enumeration.md)（robots.txt の確認）
3. **Web アプリのバージョンを特定する**
   - ヘッダー・フッター・ログインページ・`/api/health` 等にバージョンが表示されていないか確認
   - 判明したら即 `searchsploit [アプリ名] [バージョン]` で既知脆弱性を検索
   - CVE が見つかった場合、パストラバーサル / RCE 等の深刻な脆弱性が優先
4. gobuster / ffuf でディレクトリ列挙 → [`../01_Reconnaissance/Web_Enumeration.md`](../01_Reconnaissance/Web_Enumeration.md)（ディレクトリ・エンドポイントの列挙）
5. **エンドポイントの ID やパラメータに連番・予測可能な値がないか確認** → IDOR の可能性
6. vhost（仮想ホスト）のファジングを検討
7. **JS ソースに難読化コードがある場合はデコードして API エンドポイントを発見する**
   - `eval(function(p,a,c,k,e,d){...})` が見えたらブラウザ Console で `eval → console.log` に置換
   - API レスポンスのエンコーディング種別（ROT13 / Base64 等）も確認する
   → [`../02_Initial_Access/Web_Vulnerabilities/JS_Obfuscation.md`](../02_Initial_Access/Web_Vulnerabilities/JS_Obfuscation.md)
8. **API が `username`, `host`, `ip` 等のパラメータを受け取る場合はコマンドインジェクションを試みる**
   - まず管理者 API への権限昇格（`is_admin=1` 等のパラメータ改ざん）を確認
   - `; id` で注入テスト → リバースシェルへ
   → [`../02_Initial_Access/Web_Vulnerabilities/Command_Injection.md`](../02_Initial_Access/Web_Vulnerabilities/Command_Injection.md)
9. **URL を入力して何かを生成・取得するフォームがある場合（PDF 生成・プレビュー・スクリーンショット等）**
   - レスポンスヘッダーで使用言語・ライブラリを確認（`X-Runtime: Ruby` / `X-Powered-By` 等）
   - 生成物（PDF・画像等）のメタデータからライブラリ名・バージョンを確認（`exiftool` / strings）
   - バージョンが特定できたら即 `searchsploit [ライブラリ名] [バージョン]` で CVE を検索
   - **PDFKit 0.8.6 未満なら CVE-2022-25765（バックティック URL 注入 → RCE）が有効**（修正版は 0.8.6。0.8.6 自体はパッチ済みなので除外）
   - SSRF として内部ネットワーク探索にも使える可能性がある
   → [`../02_Initial_Access/Web_Vulnerabilities/Command_Injection.md`](../02_Initial_Access/Web_Vulnerabilities/Command_Injection.md)（PDFKit セクション）

---

## 見えたもの → 次の経路

| 観測 | 次に開く |
|------|---------|
| robots / dir 列挙で管理画面・隠しパス | [`../01_Reconnaissance/Web_Enumeration.md`](../01_Reconnaissance/Web_Enumeration.md)（列挙継続） |
| アプリ名 + バージョンが判明 | `searchsploit [アプリ名] [版]` → [`../05_Tools_Reference/Searchsploit.md`](../05_Tools_Reference/Searchsploit.md) / エッジ機器なら [`../02_Initial_Access/Edge_Appliance_CVEs.md`](../02_Initial_Access/Edge_Appliance_CVEs.md) |
| 個別脆弱性の兆候（SQLi / SSRF / CmdInj / LFI / IDOR 等）| [`../02_Initial_Access/Web_Vulnerabilities/`](../02_Initial_Access/Web_Vulnerabilities/) の該当ファイル |
| キャプチャした req/res をまず読みたい | [`../01_Reconnaissance/Web_Response_Triage.md`](../01_Reconnaissance/Web_Response_Triage.md)（初動 5 分） |
| **web から認証情報を入手した** | **OS に応じて戻る** → Windows なら [`Windows_AD_Attack_Flow.md`](./Windows_AD_Attack_Flow.md)（WinRM / SMB 等で使う）、Linux なら [`Linux_Attack_Flow.md`](./Linux_Attack_Flow.md)（SSH 等で使う）|
| TLS / 証明書から組織・製品・追加 FQDN を掘りたい | [`../01_Reconnaissance/TLS_Audit.md`](../01_Reconnaissance/TLS_Audit.md) |
| 急に 403 / challenge に変わる・前段が CDN/WAF らしい | [`../01_Reconnaissance/CDN_WAF_Edge_Recon.md`](../01_Reconnaissance/CDN_WAF_Edge_Recon.md) |
| 見たことのない技術・機能で脆弱性クラスが特定できない | [`01_Unknown_Tech_Research.md`](./01_Unknown_Tech_Research.md) |

> **web で cred を取ったら元の OS フローに戻る**のがこのファイルの要点。「web 入口 → cred 入手 → WinRM / SSH で侵入」の橋渡しがここで切れないようにする（Windows 標的で web を見て cred を取り、5985 で入るのは定番経路）。

---

## 関連技術

- 列挙手順の詳細 → [`../01_Reconnaissance/Web_Enumeration.md`](../01_Reconnaissance/Web_Enumeration.md)
- Web 診断スコープ（網羅洗い出し）→ [`Web_Vuln_Flow.md`](./Web_Vuln_Flow.md)
- 戻り先（OS 別の本流）→ [`Windows_AD_Attack_Flow.md`](./Windows_AD_Attack_Flow.md) / [`Linux_Attack_Flow.md`](./Linux_Attack_Flow.md)
- 個別 Web 脆弱性の攻撃（例）→ [`../02_Initial_Access/Web_Vulnerabilities/SQLi.md`](../02_Initial_Access/Web_Vulnerabilities/SQLi.md) ほか同フォルダ各ファイル
