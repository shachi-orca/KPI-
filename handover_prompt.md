# KPI管理システム / CRM — Claude Code 引き継ぎプロンプト

## ⚡ 次のセッションで最初にやること

着手前に必ず対象ファイルを Read して現在の行数・関数配置を確認すること。

---

### 【次優先】未対応事項（2026-06-20時点）

| ファイル | 内容 | 状態 |
|---|---|---|
| `customer_crm.html` | 録画ファイルの対応履歴内再生：現状は対応履歴にファイル名だけ残り、動画自体はローカルにダウンロードされるのみ。履歴から直接再生したい場合はIndexedDBに動画を保存しアプリ内再生する設計に変更が必要 | 要否はユーザー判断待ち |
| 3ファイル共通 | バックアップの復元機能（バックアップJSONを読み込んで状態を戻す導線）が未実装 | 未着手 |
| `customer_crm.html` | CTI（電話）本格連携：ソフトフォン/CTIサービス契約後、`callNumber()`をAPI呼び出しに差し替え | **契約待ち** |
| `customer_crm.html` | Web会議自動生成連携：Zoom等のAPI契約後、`openMeeting()`周りに会議自動作成処理を追加 | **契約待ち** |
| `kpi_system.html` | 目標の複数月トレンドグラフ | 未着手 |
| `kpi_system.html` | ファネル下流の0偏り / 期間ナビの年範囲ハードコード | 見送り中 |
| `kpi_system.html` | パスワードの平文保存 | 管理者確認要件で見送り中 |

---

### 解決済み：案件管理の顧客選択バグ → 手動入力機能を削除する方針に確定（2026-06-19）

前回セッションでKGI独自の「顧客リスト」ページを削除した際、`state.customers` を読み込む手段がなくなり、`dealModal`の顧客選択が常に空になる不具合があったが、ユーザー判断で「KGIはKPI・CRMからの自動転記の確認のみで良く、手動入力は不要」と方針確定。`dealModal`/`showDealModal`自体を削除して解決済み（詳細は後述）。

---

### 次セッションの実装詳細（kpi_system.html、未着手）

- 目標の複数月トレンドグラフ：具体的な仕様は次回相談

---

## 今セッション（2026-06-19・後半）で完了した実装：CTI/Web会議連携の下準備

ユーザー要望：CRMから顧客情報をもとに電話・メール発信、オンライン商談（Web会議）をできるようにしたい（昔のFileMaker+Zoiperのような使い方のイメージ）。
今回はソフトフォン・会議サービスをまだ契約していないため、契約後すぐ繋ぎ込めるよう「ベース」のみ実装。本格CTI API（Twilio等）・会議自動生成API（Zoom等）は未実装（契約後に着手）。

### `customer_crm.html` 実装内容 ✅

- `callNumber(tel)` / `composeMail(email)` / `openMeeting(url)` を追加（utilities関数群の直後）。現状は`tel:`/`mailto:`リンクと`window.open`のみだが、**将来CTI/会議APIに差し替える際はこの3関数だけ変更すればよい設計**。
- 顧客詳細ページ：電話番号・連絡先に「発信」ボタン（`data-call`）、メールに「メール作成」ボタン（`data-mail`）を追加。クリックでOS既定のアプリ（Zoiper等をtel:ハンドラに設定していればそれが起動）が立ち上がる。
- アポイント設定フォームに「Web会議URL（任意）」入力欄（`apo-mtg`）を追加。`http(s)://`形式のみ許可（簡易バリデーション）。
- アポイントデータに`mtg`フィールドを追加（既存データへの影響なし）。設定されている場合、次回アポイント表示エリアに「Web会議に参加」ボタン（`data-meeting`）を表示。
- バインド：`data-call`/`data-mail`/`data-meeting`を`bind()`内に追加。

### 既存バグ修正：アポイント編集時の日時表示がズレる（2026-06-19発見・修正済み）

`detailHTML()`の`apoVal`が`new Date(apo.at).toISOString().slice(0,16)`（UTC基準）を`datetime-local`入力欄にそのまま入れていたため、JST環境では実際の予定時刻より9時間早い日時が表示されていた。既存の有効なアポイントを編集して保存すると、日時を変更していなくても「過去の日時は設定できません」エラーで保存できない、または意図せず時刻がズレる不具合があった。
`toLocalDT(ts)`関数を新設（ローカル時刻基準でフォーマット）し、`apoVal`の計算をこれに置き換えて修正。**今回のWeb会議URL機能のテスト中に発覚**——既存アポイントにURLを追記しようとすると保存できない事象から特定。

**動作確認**：headless Chrome（CDP経由）で、(1)発信/メール作成/会議参加ボタンの表示とクリック時の関数呼び出し、(2)Web会議URLの不正値拒否・正常値保存・空値クリア、(3)修正後は既存アクティブなアポイントの編集保存が正常に通ることを確認済み。

### 追加要望：対応記録の保存を強制するナビゲーションロック ✅

要望：発信/メール作成/Web会議参加ボタンを押したら、新規対応入力を保存するまで他ページに移動できないようにする。

実装：
- `pendingLog`（グローバル変数。`{type:'call'|'mail'|'meeting'}`）を追加。`data-call`/`data-mail`/`data-meeting`クリック時にセットし`render()`。
- `blockNav()`：`pendingLog`があれば`toast`でエラー表示し`true`を返すガード関数。`logout-btn`/`data-nav`/`back-btn`の各クリックハンドラの先頭で呼び出し、`true`なら処理を中断（ページ遷移しない）。
- 顧客詳細ページ上部に警告バナーを表示（`pendingLog`がある間）。誤クリック時の救済として「対応記録をキャンセル」リンク（`pl-cancel`）を設置——確認なしの完全ブロックだと誤操作時に詰むため追加。不要なら削除可。
- `saveContact()`成功時に`pendingLog=null`でロック解除。
- `window.beforeunload`でタブを閉じる/リロードする際もブラウザ標準の確認ダイアログを表示（`pendingLog`がある場合のみ）。

**動作確認**：headless ChromeでCDP経由で、(1)発信後に設定/案件管理タブ・戻るボタン・ログアウトが全てブロックされること、(2)対応記録を保存すると解除され遷移できること、(3)「キャンセル」リンクでも解除できることを確認済み。

---

## 今セッション（2026-06-19・後半その2）で完了した実装：① 案件追加モーダル削除 ／ ③ データバックアップ

### ① crm_system.html：案件管理の手動入力を完全削除 ✅

方針確定：KGIには「KPI・CRMからの自動転記の確認」だけが残ればよく、手動での案件入力・編集は不要（ユーザー指示）。

削除したもの：
- `dealModal()` / `showDealModal()` 関数（「案件追加」「編集」両方）
- 「案件追加」ボタン（`deal-add-btn`）、編集ボタン（`data-deal-edit`）とそのバインド
- `dealRow`の「顧客」列・「確度」列（`state.customers`参照は常に空配列で動作していなかったため。タイトルに顧客名が既に含まれているので実害なし）
- `adminPage()`の「顧客数」列
- `state`から`customers`・`selCustId`・`custFilter`（前回の顧客リスト削除時の残骸、完全に未使用だった）

結果：案件は完全に「CRMの完了顧客→自動転記」のみで作られ、表示専用になった。

### ③ 3ファイル共通：自動バックアップ機能 ✅

要望：データが大きくなってきたので壊れる前にバックアップしたい。外部に漏れない方法で、夜間・朝などの「決まった時刻」自動実行を希望されたが、静的HTML（サーバーなし）では「タブが閉じていても決まった時刻に実行」は技術的に不可能なため、下記の方式に決定：

- **ログアウトボタンを押した瞬間に自動バックアップ**（実質「閉じる前にバックアップ」と同等。`beforeunload`中のダウンロードはブラウザにブロックされるため不採用）
- **ログイン後、前回バックアップから12時間以上経過していたら自動バックアップ**（ログアウトせず閉じた場合の保険）
- 保存先は**完全にローカルのみ**（ブラウザの自動ダウンロード。外部送信なし）

実装（3ファイル共通パターン、关数名統一）：
- `BACKUP_KEY`／`BACKUP_INTERVAL_MS`（12時間）／`_backupChecked`（1ページロードに1回だけチェック）
- `lastBackupLabel()`：最終バックアップ日時を表示用に整形
- `exportFullBackup(silent)`：全データをJSONダウンロード＋`BACKUP_KEY`に時刻保存。`silent=false`なら保存完了トースト表示
- `checkAutoBackup()`：`BACKUP_INTERVAL_MS`超過していれば`exportFullBackup(true)`
- `render()`内で認証済み描画の直前に1回だけ`checkAutoBackup()`を呼ぶ
- ログアウトボタンのonclick先頭で`exportFullBackup(true)`を呼んでからセッションクリア

ファイル別の詳細：
| ファイル | 手動バックアップボタンの場所 | バックアップ対象 |
|---|---|---|
| `kpi_system.html` | 管理者画面 → スタッフ管理タブ末尾 | `localStorage['kpiSystemState_v2']`全体 |
| `crm_system.html` | 管理者ビュー（ページ末尾に新規カード） | `localStorage['crmSystemState_v1']`全体 |
| `customer_crm.html` | 設定 → データ管理タブ（既存の「すべてのデータをまとめて保存」ボタンを`exportFullBackup`経由に変更） | `D`（顧客・履歴・商材・割り当て）全体 |

**動作確認**：headless Chrome（CDP経由）で3ファイルそれぞれ、(1)初回ログイン時に自動バックアップが走ること、(2)同セッション内の再renderで重複実行されないこと、(3)手動バックアップボタンが動作すること、(4)ログアウト時に自動バックアップ後セッションがクリアされることを確認済み。

---

## 今セッション（2026-06-20）で完了した実装：通話・Web会議の録画機能

要望：応対履歴で電話やオンライン商談を録画して、後から対応履歴から確認できるようにしたい。
電話の録画はブラウザからは技術的に不可能（OS/電話アプリ側の機能が必要）なため対象外。Web会議の録画は「有料の会議サービスの録画機能」と「無料のブラウザ標準画面録画」の二択を提示し、**無料の方（ブラウザ標準）を採用**。

### `customer_crm.html` 実装内容 ✅

- `navigator.mediaDevices.getDisplayMedia()` + `MediaRecorder`でブラウザ標準の画面録画を実装（契約・追加コストなし）
- 顧客詳細ページの「新規対応入力」パネル内に録画開始/停止ボタンを追加
- 録画停止時：`.webm`ファイルを自動ダウンロード（ファイル名: `録画_顧客名_YYYYMMDD_HHMM.webm`）し、対応入力欄に`[録画ファイル: ファイル名]`を自動追記
- 録画中・対応記録未保存時はナビゲーションロック（`blockNav()`）が発動し、他ページへ移動できないようにした（既存の`pendingLog`機構を拡張、`type:'record'`を追加）
- `beforeunload`にも`isRecording`チェックを追加（録画中のタブを閉じようとすると確認ダイアログ）

**動作確認済み（2026-06-20、headless Chrome／CDP、`getDisplayMedia`/`MediaRecorder`をモック）**：(1)録画開始でUI(録画中表示・停止ボタン)が切り替わること、(2)録画中は`blockNav()`が全ページ遷移をブロックすること、(3)停止で`.webm`ダウンロード・対応入力欄への追記・`pendingLog`セットが正しく行われること、(4)保存前は`pendingLog`によりナビゲーションロックが継続すること、(5)`saveContact()`で対応履歴に記録され、ロックが解除されること、を一連の流れで確認済み。

**既知の制約**：
- 動画ファイルはローカルダウンロードのみで、対応履歴上では再生できない（ファイル名の記録のみ）。履歴から直接再生したい場合はIndexedDB保存＋アプリ内プレイヤーへの設計変更が必要（要否はユーザー判断待ち）
- Mac環境では`getDisplayMedia`の音声キャプチャがタブ共有以外（画面全体・他アプリウィンドウ）だとOS側の制約で音声が含まれない場合がある（ブラウザの仕様上の制約）

---

## 前回セッション（2026-06-19・前半）で完了した実装

### ① crm_system.html 活動記録ページ削除 ✅

削除したもの：
- サイドバーの「活動記録」ナビ
- `ACT_TYPES`定数 / `actChip()`関数
- `exportActivityCSV()` / `activitiesPage()` / `actModal()` / `showActModal()` 関数
- `bindPage()`内の活動関連バインド（`act-filter-type`,`act-filter-staff`,`act-csv-btn`,`act-add-btn`,`data-act-del`）
- `state.activities` / `state.actFilter`
- `dashboardPage()`の「最近の活動」カード
- `adminPage()`の「今月活動数」列

### ② crm_system.html CRM自動転記 ✅

- `loadCrmData()`：`crmCustomerState_v1`（customer_crm.html）を読み込み
- `syncDealsFromCrm()`：顧客`status==='完了'`の顧客を`state.deals`に`status:'won'`で自動追加
  - 重複防止：`d.crmSourceId === 'crm_'+cust.id`
  - 商材名：該当顧客の最新contactの`pid`→`products`
  - 担当者：最新contactの`si`
  - `render()`先頭で`loadCrmData()`→`syncDealsFromCrm()`を呼び出し

### ③ crm_system.html KGIダッシュボード連携 ✅

②で自動転記された案件は`state.deals`に乗るため、既存の`monthPrediction()`が自動集計。
ダッシュボードの月末着地予測カードに「（うちCRM自動転記 n件）」の注記を追加。

結果：810行 → 698行（活動記録機能の削減分が転記機能の追加分を上回る）

**動作確認**：headless Chrome（CDP経由）でCRM顧客データを模擬投入し、①サイドバーから活動記録が消えていること②自動転記でstate.dealsに1件追加されること③再renderしても重複しないこと④ダッシュボード・案件管理（フィルタ「全て」時）・管理者ビューに正しく反映されることを確認済み。

---

## 前々回セッション（2026-06-18）で完了した実装

### KGI（crm_system.html）顧客リストページ削除 ✅

削除したもの：
- サイドバーの「顧客リスト」ナビ
- `customersPage()` 関数（48行）
- `customerDetailPage()` 関数（65行）
- `custModal()` 関数（34行）
- `showCustModal()` 関数（23行）
- 関連イベントバインド（顧客リンク・フィルタ・追加・編集）
- 案件・活動ページの顧客名リンク → プレーンテキスト化

結果：996行 → 810行

---

## 3ファイルの役割（確定）

| ファイル | 役割 | 管理する内容 |
|---|---|---|
| **kpi_system.html** | 日々の活動量の記録・管理 | コール数・勤怠・シフト・KPI数字 |
| **customer_crm.html** | 顧客情報マスタ＋対応履歴 | 誰に・何をしたか。成約・失注の記録 |
| **crm_system.html（KGI）** | 結果の確認のみ | 月末着地・契約結果（KPI＋CRMデータを参照） |

KGIに残すページ：ダッシュボード、案件管理、管理者ビュー（3ページのみ）
KGIから削除したページ：顧客リスト（2026-06-18）、活動記録（2026-06-19）

---

## プロジェクト概要

### ファイル構成・行数

```
kpi_system.html     （KPI・シフト・勤怠管理。LocalStorageキー: kpiSystemState_v2）約3934行
crm_system.html     （KGI管理システム。LocalStorageキー: crmSystemState_v1）約650行
customer_crm.html   （顧客管理CRM。LocalStorageキー: crmCustomerState_v1）約1048行
```

### KPI ↔ KGI(crm_system.html) の共有設計
- `crm_system.html` は起動時に `kpiSystemState_v2` を読み込み、`_staffList`・`staffPw`・`adminPw`・`goals`・`role` を参照（書き込みなし）
- KPIシステムで管理者ログイン中にKGIを開くと自動的に管理者セッションを同期
- スタッフがKGIにアクセスするとアクセス拒否画面を表示

### KGI ↔ CRM(customer_crm.html) のデータ連携（2026-06-19追加）
- `crm_system.html` は起動時に `crmCustomerState_v1` を読み込み（`loadCrmData()`）
- 顧客ステータスが「完了」の顧客を`state.deals`に成約として自動転記（`syncDealsFromCrm()`、書き込みは`crmSystemState_v1`側のみ。CRM側は読み取り専用）

### KPI ↔ CRM(customer_crm.html) の設計
- `customer_crm.html` は `kpiSystemState_v2` を認証のみに使用（スタッフリスト・パスワード参照）
- セッション自動同期なし。KPIにログイン済みでも **CRMへの再ログインが必要**
- `?cid={id}` クエリパラメータで顧客詳細に直接遷移

---

## customer_crm.html 設計詳細

### データ構造
```js
D = {
  products:     [{id, name}],
  assignments:  [{id, si, year, month, pid}],
  customers:    [{id, name, company, role, tel, tel2, email,
                  postalCode, address, status, tags, ctx[], createdAt}],
  contacts:     [{id, cid, si, pid, year, month, at,
                  method, reason, manager, content}],
  appointments: [{id, cid, si, at, note, mtg}],  // mtg: Web会議URL（任意、2026-06-19追加）
  _cid, _hid, _aid
}
```
※`contacts`に「対応後ステータス」フィールドは無い。`saveContact()`で`contactStatus`を選ぶと顧客側の`status`だけが更新される（contact自体には残らない）。CRM自動転記はこの顧客`status`を見ている。

### 定数
```js
ROLES    = ['代表取締役','取締役','部長','課長','係長','担当者','一般','その他']
METHODS  = ['アウトバウンド','インバウンド','メール対応','OLS','対面','その他']
REASONS  = ['折返し連絡','クレーム対応','新規提案','その他']
STATUSES = ['未対応','優先対応','対応中','完了']
```

### 顧客ステータス色
| ステータス | CSSクラス | 色 |
|---|---|---|
| 未対応 | `.bd` | 赤 |
| 優先対応 | `.bp` | オレンジ枠 |
| 対応中 | `.bw` | 黄 |
| 完了 | `.bs` | 緑 |

### 画面構成（横タブナビ）

| 画面 | pg値 | 概要 |
|---|---|---|
| CRM ポータル | `portal` | 担当割り当てを季節カードで表示 |
| 顧客一覧 | `customers` | 検索・ステータスフィルタ・新規登録 |
| 顧客詳細 | `detail` | 左：次回アポ→顧客情報→アポ設定。右：対応入力フォーム＋履歴タイムライン |
| 設定（管理者のみ） | `settings` | 商材管理・担当割り当て・データ管理の3タブ |

### 主要関数
```js
render()          // セッション確認 → login/app HTML を描画 → bind()
portalHTML()      // 季節カードグリッド
customersHTML()   // 顧客一覧
detailHTML()      // 顧客詳細＋アポ表示＋対応入力フォーム＋タイムライン
settingsHTML()    // 設定
saveContact()     // 対応履歴保存（contactStatus で顧客status更新）
saveCust()        // 顧客新規登録・編集
saveApo()         // アポイント設定・更新
deleteApo()       // アポイント削除
lookupZip()       // 郵便番号 → zipcloud API → 住所自動補完
apoWatchdog IIFE  // 全3ファイル末尾に共通。30秒ごとにアポ監視
```

---

## アポイントアラート仕様（全3ファイル共通）

```
定数:
  CRM_KEY = 'crmCustomerState_v1'
  DIS_KEY = '_apoDismissed'（sessionStorage）
  BID     = '_apoBanner'（DOM要素ID）
  TWENTY  = 20分（ms）

表示条件:
  - アポ時刻の20分前 〜 1分経過後まで表示
  - sessionStorage[DIS_KEY] に含まれるIDは除外
  - staff: a.si === sess.user のもののみ
  - admin: 全件
```

---

## kpi_system.html 実装済み機能

### ページ構成（サイドバーナビ）

| ページ | id | 概要 |
|--------|-----|------|
| 打刻 | attendance | 出勤・休憩・退勤の4ボタン。残業申請・有給申請フォーム付き |
| 出勤履歴 | history | 日別打刻一覧 / 総合時間集計の2タブ。CSV出力可 |
| KPI入力 | kpi | 活動指標10項目・率指標8項目をカレンダー軸で入力 |
| 時間帯集計 | timeslot | 区切り時刻を自由追加・削除 |
| コールセンター | cc | 着信数/応答数/通話時間を日次入力 → 応答率/放棄率/AHT自動計算 |
| チャット報告 | report | LINE/Chatwork/Slackフォーマット切替 |
| 個人ダッシュボード | dashboard | 前月比カード。全KPI達成状況（月末予測列付き）|
| チームダッシュボード | team | アラートパネル。目標vs着地予測カード |
| 業績サマリー | summary | アウトバウンド・インバウンド統合サマリー |
| 着地タイムライン | timeline | KPIタイムラインバー表示 |
| チームメモ帳 | memo | 全スタッフ共有メモ |
| 管理者画面 | admin | 代理打刻 / 打刻修正 / 有給欠勤 / 目標設定 / 残業承認 / シフト / 給与 / スタッフ管理 |

---

## 開発ルール・お作法（kpi_system.html）

- **render() を基本単位**として扱う。状態変更後は必ず `render()` を呼ぶ
- **confirm() / alert() / prompt() は使わない** — toast / インラインバー で代替
- **innerHTMLへ入れる自由テキストは必ず `esc()`**
- **スタッフはindex（si）で全データを参照**
- 率指標は必ず **Math.min(100, ...)** で100%クランプ

## あなたへの指示

このプロジェクトは **コールセンター・営業KPI管理システム + 顧客管理CRM** の開発です。
既存ファイルを引き継いで開発を継続してください。
