# KPI管理システム — Claude Code 引き継ぎプロンプト

## ⚡ 次のセッションで最初にやること

着手前に必ず対象ファイルを Read して現在の行数・関数配置を確認すること。

---

### 【次優先】未対応事項（2026-06-18時点）

対象ファイル：`kpi_system.html`（現在 約3825行）／`crm_system.html`

- 目標の複数月トレンドグラフ（未着手）
- ファネル下流の0偏り / 期間ナビの年範囲ハードコード（見送り中）
- パスワードの平文保存（管理者確認要件で見送り中）

---

## 今セッション（2026-06-18）で完了した実装

### チームメモ帳（`kpi_system.html`）✅
- サイドバーに「メモ帳」(id=`memo`) ページを追加（全スタッフ共有・1つのメモ帳）
- `state.sharedMemo = {text, by, byName, at}` で保存。最終更新者・日時を表示
- `SENSITIVE_RULES` / `detectSensitive(text)` で8パターンをリアルタイム検出しブロック
  - 電話番号・メールアドレス・郵便番号（〒）・クレジットカード番号
  - マイナンバー・個人番号 / 口座番号情報 / パスワード記述（「パスワード：○○」形式）/ 社会保険番号
- クリアボタン → インライン確認バー（「消去する」「キャンセル」）で即時消去を防止
- `memoPage()` / `bindMemo()` を `timelinePage()` の直前に実装

### KGI管理者セッション自動同期（`crm_system.html`）✅
- KPIシステムで管理者ログイン中にKGIボタンを開くと、CRM側の古いセッションを上書きして自動的に管理者としてログイン
- `render()` 冒頭で `kpiState.role==='admin'` を検出して `state.role='admin'` に同期

### KGIリネーム＋着地タイムライン（`kpi_system.html` / `crm_system.html`）✅
- `kpi_system.html`: topbarのCRMリンクを管理者専用・アイコン`ti-target`・ラベル「KGI」に変更（スタッフには完全非表示）
- `crm_system.html`: タイトル・topbar・ログイン画面を「KGI管理」に変更。スタッフがログインするとアクセス拒否画面を表示
- `kpi_system.html`: サイドバーに「着地確認」(id=`timeline`)ページを追加
  - `timelinePage()` / `bindTimeline()` を実装
  - スタッフ：自分の全KPI_ACTをタイムラインバーで表示（実績バー・目標ペース線・今日の縦線・→月末予測%・バッジ）
  - 管理者：KPI切替 ＋ 全スタッフ一覧ビュー（チーム合計行付き）／特定スタッフ全KPIビューを切替
  - state追加: `tlKpi`（選択KPI文字列）/ `tlView`（`'staff'`|`'self'`）/ `tlStaff`（スタッフindex）

### KPI実績参照カード（`crm_system.html`）✅
- `sumKpiCRM(si, k)` を追加（`kpiSystemState_v2` の `kpiData` から月次合計を算出）
- ダッシュボードの月末着地予測カード上部に「KPI実績参照」カードを表示
  （契約数・コール数・アプローチ数・クロージング数。KPIデータ未設定時は非表示）

### アラートボタン＋目標・着地予測（`kpi_system.html`）✅
- `alertsPanelHTML()` の `active.map` 内：残業アラート → 「申請」ボタン（`state.page='attendance'`へ遷移）
- `alertsPanelHTML()` の `active.map` 内：KPI達成率アラート → 「KPI確認」ボタン（`state.page='kpi'; state.kpiTab='rate'`へ遷移）
- `dashboardPage()` 個人ビューの「全KPI達成状況」カード：各行に「月末予測」列を追加
- `teamPage()` のアラートパネル直下：「目標 vs 着地予測」カード追加（4KPI×全スタッフ表形式）

---

## あなたへの指示

このプロジェクトは **コールセンター・営業KPI管理システム** の開発です。
既存ファイル `kpi_system.html` を引き継いで開発を継続してください。

---

## プロジェクト概要

### 構成
- **単一HTMLファイル** (`kpi_system.html`)
- フレームワーク不使用・バニラJS + Chart.js + Tabler Icons
- 全データはメモリ上の `state` オブジェクトで管理（LocalStorageに永続化）
- 個別認証（2026-06〜）：スタッフ別パスワード。初期 `kpi@1234`（初回・月初ログイン時に変更必須）／管理者 `admin@1234`

### 対象ユーザー
- スタッフ10名のコールセンター・営業チーム
- 担当者ロール / 管理者ロールの2種類

---

## 実装済み機能

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
| **着地タイムライン** | **timeline** | **スタッフ：自分の全KPIをタイムラインバーで表示。管理者：全スタッフ一覧＋チーム合計** |
| **チームメモ帳** | **memo** | **全スタッフ共有メモ。センシティブ情報（電話/メール/マイナンバー等）は保存ブロック** |
| 管理者画面（管理者のみ） | admin | 代理打刻 / 打刻修正 / 有給欠勤 / 目標設定 / 残業承認 / シフト / 給与 / スタッフ管理 の8タブ |

### KPI項目定義

**活動指標（入力値）**
```
コール数, アプローチ数, 対コールコンタクト数, 対アプローチコンタクト数,
完了数, クロージング数, 依頼数, 後確数, 契約数, 解約数
```

**率指標（自動計算・最大100%クランプ済み）**
```
対コールコンタクト率     = 対コールコンタクト数 / コール数
対アプローチコンタクト率 = 対アプローチコンタクト数 / アプローチ数
対コール完了率           = 完了数 / コール数
対アプローチ完了率       = 完了数 / アプローチ数
クロージング率           = クロージング数 / 完了数
依頼率                   = 依頼数 / クロージング数
後確率                   = 後確数 / 依頼数
契約率                   = 契約数 / 後確数
解約率                   = 解約数 / 契約数
```

### stateオブジェクト主要プロパティ

```js
state = {
  page,           // 現在表示ページID
  user,           // ログイン中スタッフのindex (null=管理者)
  role,           // 'staff' | 'admin'

  // 個別認証
  staffPw,        // {si: {pw, changedYM}}
  adminPw,        // 管理者パスワード（既定 'admin@1234'）
  pendingUser,    // PW変更待ちスタッフindex

  // 期間
  viewYear, viewMonth, _gen,

  // 打刻
  attStatus, attLog, workStart, breakStart, totalBreak, workEndTime,

  // KPI（periodKey='年_月'で管理）
  kpiData,        // {periodKey: {staffIndex: {day: {kpiName: value}}}} — kData()
  goals,          // {periodKey: {staffIndex: {kpiName: targetValue}}}   — gData()
  activeStaff,    // index | 'total'
  kpiTab,         // 'act' | 'rate'
  kpiTotalView,   // 'staff' | 'daily'

  // 時間帯集計
  timeSlots, timeData, tsStaff, tsView, tsTab,

  // チャット報告
  reportTab, reportScope, reportKpis, reportStaff, reportPeriod, reportDay,

  // 管理者・目標設定
  goalStaff, adminTab, proxyStaff, correctionLog,

  // 勤怠・シフト
  ccStaff, ccView, leaveData, leaveRequests, alertAck,
  otRequests, shifts, payroll, shiftPrefs, shiftChangeReqs,
  shiftRemind, shiftDeadline, shiftNotify, otNotify, leaveNotify,

  // ダッシュボード
  dashStaff, dashKpi, dashChart,
  teamKpi, teamTab,
  histStaff, histTab,
  summaryStaff,

  // 着地タイムライン（2026-06追加）
  tlKpi,          // 選択中KPI名（文字列）
  tlView,         // 'staff'（全スタッフ一覧）| 'self'（個人全KPI）
  tlStaff,        // 表示スタッフindex（管理者の個人ビュー時）

  // メモ帳（2026-06追加）
  sharedMemo,     // {text, by(si|null), byName, at(timestamp)}

  // 一時UI状態（NO_PERSIST）
  _prefillOtDate, _prefillLvDate,
}
```

### 主要関数

```js
render()              // 全画面再描画。冒頭で recalcPeriod()→initKpi()→ensureGoalFloor()→saveState()
sumKpi(si, kpi)       // スタッフsiのkpi「閲覧中月」合計
sumKpiAll(kpi)        // 全スタッフのkpi「閲覧中月」合計
calcRate(si, d)       // スタッフsiの日dの率指標オブジェクト
getDayLogs(si)        // スタッフsiの打刻集約
computeAlerts()       // アラート配列生成（KPI未達/欠勤/放棄率/PW/残業/休憩不足/深夜/KPI達成率80%未満）
alertsPanelHTML()     // アラートパネルHTML（スタッフは自分のみ・管理者は全員。遷移ボタン付き）
timelinePage()        // 着地タイムラインページ（スタッフ：自分全KPI / 管理者：全スタッフ一覧）
memoPage()            // チームメモ帳ページ（センシティブ検出付き）
detectSensitive(text) // センシティブ情報を検出して検出種別の配列を返す
saveState()           // LocalStorageへ保存（NO_PERSIST除外）
loadState()           // 保存stateを復元
```

> **永続化キー**: `kpiSystemState_v2`
> **NO_PERSIST**: `dashChart`, `clockInterval`, `_dashDrawChart`, `greetingMsg`, `greetingType`, `_prefillOtDate`, `_prefillLvDate`
> **日付定数**: `REAL_Y/REAL_M/REAL_D`＝実際の今日。`Y/M/DAYS/TODAY_D/LAST_D`＝閲覧中の月（recalcPeriodで更新）

---

## 既知の課題・未対応事項

- **目標の複数月トレンドグラフ**：未着手
- **ファネル下流の0偏り / 期間ナビの年範囲ハードコード**：見送り中
- **パスワードの平文保存**：管理者確認要件で見送り中

---

## 開発ルール・お作法

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

---

## ファイル構成

```
kpi_system.html    （KPI・シフト・勤怠管理。LocalStorageキー: kpiSystemState_v2）
crm_system.html    （KGI管理システム。LocalStorageキー: crmSystemState_v1）
```

### KPI ↔ KGI(CRM) の共有設計
- `crm_system.html` は起動時に `kpiSystemState_v2` を読み込み、`_staffList`・`staffPw`・`adminPw`・`goals`・`role` を参照（書き込みなし）
- **KPIシステムで管理者ログイン中にKGIを開くと自動的に管理者セッションを同期**（別ログイン不要）
- スタッフがKGIにアクセスするとアクセス拒否画面を表示（`kpi_system.html` に戻るボタン）
- KPI側 topbar右に「KGI」ボタン（**管理者のみ表示**。スタッフには非表示）
- CRM topbar右に「KPI管理」リンク
