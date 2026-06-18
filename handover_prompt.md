# KPI管理システム / CRM — Claude Code 引き継ぎプロンプト

## ⚡ 次のセッションで最初にやること

着手前に必ず対象ファイルを Read して現在の行数・関数配置を確認すること。

---

### 【次優先】未対応事項（2026-06-18時点）

| ファイル | 内容 | 状態 |
|---|---|---|
| `kpi_system.html` | 目標の複数月トレンドグラフ | 未着手 |
| `kpi_system.html` | ファネル下流の0偏り / 期間ナビの年範囲ハードコード | 見送り中 |
| `kpi_system.html` | パスワードの平文保存 | 管理者確認要件で見送り中 |

---

## 今セッション（2026-06-18）で完了した実装

### CRMデザイン全面刷新（`customer_crm.html`）✅
- CSS全面刷新：コントラスト強化・余白拡大・フォーカスリング・タイポグラフィ改善
- ポータルカード：季節グラデーション＋ボーダー・月数字54px
- 顧客一覧：件数バッジ追加・行高さ拡大
- 顧客詳細：ステータスをヘッダー右端に・電話番号太字強調
- 対応フォーム：保存ボタンflex幅最大化・テキストエリア4行
- タイムライン：担当者太字・種別サブテキスト分離

### 次回アポイント設定（`customer_crm.html`）✅
- 顧客詳細の左カード上部（顧客名の直下）に次回アポを表示
- 日時とメモを横並び表示
- 20分以内は⚡赤枠で強調表示
- アポ設定パネル（日時・担当者・メモ）→ 設定/更新/削除ボタン

### 20分前アラートバナー（全3ファイル共通）✅
- KPI/KGI/CRM どのページにいても画面最上部に赤バナーが固定表示（z-index:10000）
- スタッフは自分担当のアポのみ・管理者は全件表示
- 「詳細を見る」→ `customer_crm.html?cid={id}` で顧客詳細に直接ジャンプ
- ID数字部分をクリックでクリップボードにコピー（白フラッシュでフィードバック）
- 「閉じる」で sessionStorage に記録（タブ閉じると再表示）
- 30秒ごとにチェック（setInterval）

---

## プロジェクト概要

### ファイル構成・行数

```
kpi_system.html     （KPI・シフト・勤怠管理。LocalStorageキー: kpiSystemState_v2）約3896行
crm_system.html     （KGI管理システム。LocalStorageキー: crmSystemState_v1）
customer_crm.html   （顧客管理CRM。LocalStorageキー: crmCustomerState_v1）約946行
```

### KPI ↔ KGI(crm_system.html) の共有設計
- `crm_system.html` は起動時に `kpiSystemState_v2` を読み込み、`_staffList`・`staffPw`・`adminPw`・`goals`・`role` を参照（書き込みなし）
- KPIシステムで管理者ログイン中にKGIを開くと自動的に管理者セッションを同期
- スタッフがKGIにアクセスするとアクセス拒否画面を表示
- KPI topbar右：「KGI」ボタン（管理者のみ）・「CRM」ボタン（全員）

### KPI ↔ CRM(customer_crm.html) の設計
- `customer_crm.html` は `kpiSystemState_v2` を認証のみに使用（スタッフリスト・パスワード参照）
- セッション自動同期なし。KPIにログイン済みでも **CRMへの再ログインが必要**
- KPIのスタッフ/管理者パスワードをそのまま使って認証
- `?cid={id}` クエリパラメータで顧客詳細に直接遷移（起動時・ログイン後両対応）

### kpi_system.html 基本設計

- フレームワーク不使用・バニラJS + Chart.js + Tabler Icons（現在 **約3896行**）
- 全データはメモリ上の `state` オブジェクトで管理（LocalStorageに永続化）
- 個別認証：スタッフ別パスワード 初期 `kpi@1234`（初回・月初ログイン時に変更必須）／管理者 `admin@1234`

### 対象ユーザー
- スタッフ10名のコールセンター・営業チーム
- 担当者ロール / 管理者ロールの2種類

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
  appointments: [{id, cid, si, at, note}],   // 次回アポイント（顧客1件につき最大1件）
  _cid, _hid, _aid  // 採番カウンタ
}
```

### 定数
```js
ROLES    = ['代表取締役','取締役','部長','課長','係長','担当者','一般','その他']
METHODS  = ['アウトバウンド','インバウンド','メール対応','OLS','対面','その他']
REASONS  = ['折返し連絡','クレーム対応','新規提案','その他']
STATUSES = ['未対応','優先対応','対応中','完了']
```

### ステータス色
| ステータス | CSSクラス | 色 |
|---|---|---|
| 未対応 | `.bd` | 赤 |
| 優先対応 | `.bp` | オレンジ枠 |
| 対応中 | `.bw` | 黄 |
| 完了 | `.bs` | 緑 |

### 画面構成（横タブナビ）

| 画面 | pg値 | 概要 |
|---|---|---|
| CRM ポータル | `portal` | 担当割り当てを季節カードで表示。カードクリック → 顧客一覧へ |
| 顧客一覧 | `customers` | 検索・ステータスフィルタ・新規登録。ポータルカード文脈で絞り込み |
| 顧客詳細 | `detail` | 左：次回アポ→顧客情報→アポ設定パネル。右：対応入力フォーム＋履歴タイムライン |
| 設定（管理者のみ） | `settings` | 商材管理・担当割り当て・データ管理の3タブ |

### 顧客詳細ページ — 左カラム構成
```
[顧客名]
───────────────────────────
次回アポイント
  [日時]  [メモ]（横並び）
  担当: ○○
───────────────────────────
ID / 会社名 / 役職 / 電話番号 / メール / タグ / 最新対応日
───────────────────────────
アポイント設定パネル
  日時 / 担当者 / メモ
  [設定する] [削除]
```

### 重要な設計ルール（customer_crm.html）
- `render()` → `bind()` の流れで毎回全画面再描画
- 保存確認はインラインバー（`confirm()` 不使用）
- 顧客カードのステータスは **ヘッダー右端にバッジ表示**。対応履歴入力の「対応ステータス」選択で自動更新
- 郵便番号 → 住所自動補完は `zipcloud.ibsnet.co.jp` API（async/await）
- スタッフはポータルカード経由でのみ顧客一覧に到達（`ui.ctx` に文脈が入る）
- 管理者はすべての顧客を閲覧・編集可能
- アポイントは顧客1件につき最大1件（上書き方式）

### 主要関数
```js
render()          // セッション確認 → login/app HTML を描画 → bind()
portalHTML()      // 季節カードグリッド（staff: 自分のみ / admin: 全員）
customersHTML()   // 顧客一覧（ui.ctx でポータル文脈フィルタ）
detailHTML()      // 顧客詳細＋アポ表示＋対応入力フォーム＋タイムライン
settingsHTML()    // 設定（productsTab / assignTab / exportTab）
saveContact()     // cc-okボタン（確認後）から呼ばれる。contactStatus で顧客status更新
saveCust()        // 顧客新規登録・編集（modal経由）
saveApo()         // アポイント設定・更新（顧客1件につき1件・上書き）
deleteApo()       // アポイント削除
lookupZip()       // 郵便番号 → zipcloud API → 住所フィールド自動補完（async）
expJSON(name,d)   // JSONエクスポート
expCSV()          // 顧客一覧CSVエクスポート（BOM付き・Excelで開ける）
apoWatchdog IIFE  // 全3ファイル末尾に共通。30秒ごとにアポを監視しバナー表示
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

getUser() 優先順位:
  1. sessionStorage['crmCustomerSession_v1']（CRMセッション）
  2. インメモリ state.role / state.user（KPI/KGI ページ）
  3. localStorage['kpiSystemState_v2']（フォールバック）
```

---

## kpi_system.html 実装済み機能

### ページ構成（サイドバーナビ）

| ページ | id | 概要 |
|--------|-----|------|
| 打刻 | attendance | 出勤・休憩・退勤の4ボタン。残業申請（理由5択）・有給申請（理由5択）フォーム付き |
| 出勤履歴 | history | 日別打刻一覧 / 総合時間集計の2タブ。残業(未承認)バッジ・欠勤チップをクリックで申請ページへ遷移。CSV出力可 |
| KPI入力 | kpi | 活動指標10項目・率指標8項目をカレンダー軸で入力。Total選択時は「担当者別」「日別合計」2タブ |
| 時間帯集計 | timeslot | 区切り時刻を自由追加・削除。個人の「日別履歴」タブあり |
| コールセンター | cc | 着信数/応答数/通話時間を日次入力 → 応答率/放棄率/AHTを自動計算 |
| チャット報告 | report | LINE/Chatwork/Slackフォーマット切替。1クリックコピー |
| 個人ダッシュボード | dashboard | 前月比カード。全KPI達成状況（月末予測列付き）。フィードバック・1on1記録 |
| チームダッシュボード | team | アラートパネル（残業→申請ボタン・KPI未達→KPI確認ボタン）。目標vs着地予測カード |
| 業績サマリー | summary | アウトバウンド・インバウンド統合サマリー |
| 着地タイムライン | timeline | スタッフ：自分の全KPIをタイムラインバーで表示。管理者：全スタッフ一覧＋チーム合計 |
| チームメモ帳 | memo | 全スタッフ共有メモ。センシティブ情報（電話/メール/マイナンバー等）は保存ブロック |
| 管理者画面（管理者のみ） | admin | 代理打刻 / 打刻修正 / 有給欠勤 / 目標設定 / 残業承認 / シフト / 給与 / スタッフ管理 の8タブ |

### stateオブジェクト主要プロパティ

```js
state = {
  page, user, role,                    // 現在ページ・ログインユーザー・ロール
  staffPw, adminPw, pendingUser,       // 認証
  viewYear, viewMonth, _gen,           // 期間
  attStatus, attLog, workStart, breakStart, totalBreak, workEndTime,  // 打刻
  kpiData, goals, activeStaff, kpiTab, kpiTotalView,                  // KPI
  timeSlots, timeData, tsStaff, tsView, tsTab,                        // 時間帯
  reportTab, reportScope, reportKpis, reportStaff, reportPeriod, reportDay, // チャット報告
  goalStaff, adminTab, proxyStaff, correctionLog,                     // 管理者
  ccStaff, ccView, leaveData, leaveRequests, alertAck,
  otRequests, shifts, payroll, shiftPrefs, shiftChangeReqs,
  shiftRemind, shiftDeadline, shiftNotify, otNotify, leaveNotify,     // 勤怠・シフト
  dashStaff, dashKpi, dashChart, teamKpi, teamTab,
  histStaff, histTab, summaryStaff,                                   // ダッシュボード
  tlKpi, tlView, tlStaff,             // 着地タイムライン
  sharedMemo,                          // メモ帳 {text, by, byName, at}
  _prefillOtDate, _prefillLvDate,      // 一時UI状態（NO_PERSIST）
}
```

> **永続化キー**: `kpiSystemState_v2`
> **NO_PERSIST**: `dashChart`, `clockInterval`, `_dashDrawChart`, `greetingMsg`, `greetingType`, `_prefillOtDate`, `_prefillLvDate`
> **日付定数**: `REAL_Y/REAL_M/REAL_D`＝実際の今日。`Y/M/DAYS/TODAY_D/LAST_D`＝閲覧中の月（recalcPeriodで更新）

---

## 開発ルール・お作法（kpi_system.html）

- **render() を基本単位**として扱う。状態変更後は必ず `render()` を呼ぶ
- **render()を通らない連続入力（oninput）では `saveStateDebounced()` を明示的に呼ぶ**
- **KPI/目標/時間帯データは必ず `kData()`/`gData()`/`tData()` 経由で参照**
- **「現在」操作（打刻・代理打刻）は `REAL_Y/REAL_M/REAL_D` を使う**
- **confirm() / alert() / prompt() は使わない** — toast / インラインバー / `<input>` で代替
- **innerHTMLへ入れる自由テキストは必ず `esc()`**
- **スタッフはindex（si）で全データを参照**。削除は必ず `removeStaff()` 経由
- `state.activeStaff` は `'total'` になる場合があるため `parseInt()` 不可
- 率指標は必ず **Math.min(100, ...)** で100%クランプ
- **横長テーブルは `.kpi-table-wrap`（overflow:auto）で包む**

## あなたへの指示

このプロジェクトは **コールセンター・営業KPI管理システム + 顧客管理CRM** の開発です。
既存ファイルを引き継いで開発を継続してください。
