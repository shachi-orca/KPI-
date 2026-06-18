# KPI管理システム — Claude Code 引き継ぎプロンプト

## ⚡ 次のセッションで最初にやること

着手前に必ず対象ファイルを Read して現在の行数・関数配置を確認すること。

---

### 【最優先】アラートからのページ遷移ボタン ＋ 目標・着地の目視確認

対象ファイル：`kpi_system.html`（現在 約3530行）

#### ① 残業アラートに「申請する」ボタン追加

`alertsPanelHTML()`（関数内の `active.map(...)` 部分）で、アラートの `msg` に「残業」が含まれる場合、アラート行の右端に「申請する」ボタンを追加。

- クリック → `state.page='attendance'; render();`
- スタッフが自分の残業申請フォーム（打刻ページ下部 `attExtraHTML`）へ直接遷移

```javascript
// alertsPanelHTML() の active.map 内、ボタン部分に追加
${a.msg.includes('残業')?`<button class="btn btn-sm btn-primary" onclick="state.page='attendance';render();" style="font-size:11px;flex-shrink:0"><i class="ti ti-send"></i> 申請</button>`:''}
```

#### ② KPI未達成アラートに「KPIを確認」ボタン追加

アラートの `msg` に「KPI達成率」が含まれる場合、「KPIを確認」ボタンを追加。

- クリック → `state.page='kpi'; state.kpiTab='rate'; render();`
- 率指標タブ（`kpiTab='rate'`）を開いて達成状況を直接確認できる

```javascript
${a.msg.includes('KPI達成率')?`<button class="btn btn-sm" onclick="state.page='kpi';state.kpiTab='rate';render();" style="font-size:11px;flex-shrink:0"><i class="ti ti-chart-bar"></i> KPI確認</button>`:''}
```

#### ③ 目標 vs 着地の目視確認カードを追加

`dashboardPage()` または `teamPage()` に、KPI目標と現在実績を並べたサマリーカードを追加。

**表示内容**（各スタッフまたは自分について）：
- KPI項目名
- 目標値
- 現在実績
- 達成率（%）＋プログレスバー（100%=緑、80%以上=黄、未満=赤）
- 月末着地予測（現実績 ÷ 経過日数 × 月日数）

**実装方針**：
- 個人ダッシュボード（`dashboardPage()`）のスタッフ自身ビューに追加
- 管理者はチームダッシュボード（`teamPage()`）に全スタッフ分サマリー表示
- 既存の `sumKpi(si, k)` / `gData()[si][k]` / `LAST_D` / `DAYS` を使用
- `elapsed = DAYS ? Math.round(LAST_D/DAYS*100) : 0` で月経過割合取得済み
- 予測 = `Math.round(sumKpi(si,k) / LAST_D * DAYS)`（LAST_D > 0 の場合）

---

## 今セッション（2026-06-18）で完了した実装

### CRM拡張（`crm_system.html`）✅
- `deal.memo`（商談メモ）：dealModal に追加、案件一覧に1行プレビュー表示
- `deal.link`（関連リンク URL）：dealModal に追加、一覧にリンクアイコン（https?://のみリンク化）
- `exportActivityCSV()`：活動記録ページに「CSV出力」ボタン（担当者・日付・種別・顧客名・メモ、Excel互換UTF-8 BOM）

### 残業・有給申請の改善（`kpi_system.html`）✅
- 残業申請の理由を5択プルダウン＋「その他（直接入力）」に変更
- 有給申請の理由を5択プルダウン＋「その他（直接入力）」に変更（任意）
- 出勤履歴の「残業(未承認)」チップをクリッカブルに：`showOtApplyModal(ds)` でポップアップ → 打刻ページへ遷移・日付自動セット（`state._prefillOtDate`）
- 出勤履歴の欠勤日（スタッフのみ）を「欠勤/有給申請」チップに：クリックで打刻ページへ遷移・日付自動セット（`state._prefillLvDate`）
- `NO_PERSIST` に `_prefillOtDate`・`_prefillLvDate` を追加

### アラート・時間帯集計の修正（`kpi_system.html`）✅
- 時間帯集計：個人の「日別履歴」タブを実装（`if(!isTotal && tsView==='daily')` ブランチを `timeslotPage()` に追加）
- アラート：スタッフは自分のアラートのみ表示（`alertsPanelHTML()` でロール分岐）
- KPI達成率80%未満アラートを `computeAlerts()` に追加（月経過30%以上・目標設定済みKPIのみ・1行にまとめて表示）

---

## あなたへの指示

このプロジェクトは **コールセンター・営業KPI管理システム** の開発です。
既存ファイル `kpi_system.html` を引き継いで開発を継続してください。

---

## プロジェクト概要

### 構成
- **単一HTMLファイル** (`kpi_system.html`)
- フレームワーク不使用・バニラJS + Chart.js + Tabler Icons
- 全データはメモリ上の `state` オブジェクトで管理（永続化未実装）
- 個別認証（2026-06〜）：スタッフ別パスワード。初期 `kpi@1234`（初回・月初ログイン時に変更必須）／管理者 `admin@1234`。旧共通PW`1234`は廃止

### 対象ユーザー
- スタッフ10名のコールセンター・営業チーム
- 担当者ロール / 管理者ロールの2種類

---

## 実装済み機能

### ページ構成（サイドバーナビ）

| ページ | 概要 |
|--------|------|
| 打刻 | 出勤・休憩開始・休憩終了・退勤の4ボタン。退勤はインライン確認バーで実行。グリーティング表示。下部に**本日のシフト**＋**残業申請**（理由5択プルダウン）＋**有給申請**（理由5択プルダウン）フォーム（申請履歴/状態付き） |
| 出勤履歴 | 日別打刻一覧 / 総合時間集計の2タブ。管理者はスタッフ切替＋Total（全員）表示可。日別に**労務バッジ**（残業/長時間/休憩不足/深夜）。**残業(未承認)バッジはクリックでポップアップ→打刻ページへ遷移**。**欠勤日はクリックで有給申請ページへ遷移**（スタッフのみ）。CSV出力可 |
| KPI入力 | 活動指標10項目・率指標8項目をカレンダー軸で入力。Total選択時は「担当者別」「日別合計」2タブ。値変更で即時保存 |
| 時間帯集計 | 区切り時刻を自由追加・削除。**個人の「日別履歴」タブを実装**（日×KPIテーブル）。Total選択時は「時間帯別」「日別履歴」2タブ。各行右端にTotal列 |
| コールセンター | 着信数/応答数/通話時間(分)を日次入力 → 応答率/放棄率/AHTを自動計算。月次カード＋日次表。Total=担当者別 |
| チャット報告 | 個人 / チーム全体の切替。プレーン（LINE/Chatwork）/ Slackフォーマット切替。1クリックコピー |
| 個人ダッシュボード | Total選択時はチーム合算グラフ＋スタッフ別達成状況。個人選択時は本日入力テーブル＋**前月比**カード。下部に**フィードバックコメント**（双方向）＋**1on1記録**（管理者が記録） |
| チームダッシュボード | 上部に**アラートパネル**（未達/欠勤/放棄率/PW未変更/残業/休憩不足/深夜/**KPI達成率80%未満**）。**スタッフは自分のアラートのみ表示**。KPI別**前月比**。ランキング / 全員比較表の2タブ |
| 管理者画面（管理者のみ） | 代理打刻 / 打刻修正(attLog反映) / 有給欠勤(実データ・CSV) / 目標設定 / **残業承認** / **シフト** / **給与(月給/時給・出勤実績連動)** / スタッフ管理(PW含む) の8タブ |

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
  page,           // 現在表示ページID（'login'|'pwchange'|各ページ）
  user,           // ログイン中スタッフのindex (null=管理者)
  role,           // 'staff' | 'admin'

  // 個別認証（2026-06追加）
  staffPw,        // {si: {pw:文字列, changedYM:'年_月'|null}} スタッフ別パスワード
  adminPw,        // 管理者パスワード（既定 'admin@1234'）
  pendingUser,    // パスワード変更画面で変更待ちのスタッフindex

  // 期間（年月切替）★v2で追加
  viewYear,       // 閲覧中の年（既定=実際の今年。リロード時は当月へリセット）
  viewMonth,      // 閲覧中の月 0-11（既定=実際の今月）
  _gen,           // {periodKey: true} 期間ごとのサンプル生成済みフラグ

  // 打刻
  attStatus,      // {si: 'off'|'working'|'break'}
  attLog,         // [{si, type:'in'|'out'|'brk'|'ret2', time, dateKey, dateD}]
  workStart,      // {si: Date.now()} 出勤時刻ms（個人別）
  breakStart,     // {si: Date.now()} 休憩開始ms
  totalBreak,     // {si: 累計休憩ms}
  workEndTime,    // {si: 退勤確定時の実働ms}

  // KPI ★v2: 最上位が期間キー「年_月」（例 '2026_5'）
  kpiData,        // {periodKey: {staffIndex: {day: {kpiName: value}}}} — kData()
  goals,          // {periodKey: {staffIndex: {kpiName: targetValue}}}   — gData()
  activeStaff,    // KPI入力・時間帯で選択中のスタッフ index | 'total'
  kpiTab,         // 'act' | 'rate'
  kpiTotalView,   // 'staff' | 'daily'

  // 時間帯集計
  timeSlots,      // ['09:00','12:00',...] 区切り時刻配列
  timeData,       // {periodKey: {'si_kpi_slotIndex': value}} — tData()
  tsStaff,        // 時間帯選択スタッフ index | 'total'
  tsView,         // 'slot' | 'daily'
  tsTab,          // 'act' | 'rate'

  // チャット報告
  reportTab,      // 'plain' | 'slack'
  reportScope,    // 'individual' | 'team'
  reportKpis,     // 表示するKPI名の配列
  reportStaff,    // 報告対象スタッフ index
  reportPeriod,   // 'day' | 'month'
  reportDay,      // 日次レポートの対象日（1-31）

  // 管理者・目標設定
  goalStaff,      // 目標設定タブ専用のスタッフindex（-1=全体）

  // コールセンター・勤怠
  ccStaff,        // index | 'total'
  ccView,         // 'today'|'monthly'|'total'
  leaveData,      // {si:{granted:付与日数, taken:['YYYY-MM-DD',...]}}
  leaveRequests,  // [{id,si,date,reason,status,by}]
  alertAck,       // {key:true} アラート確認済みフラグ
  shiftNotify,    // {si:[{ds,code}]} シフト変更通知キュー
  _noSample,      // true = 稼働開始後（clearOperationData()実行済み）
  payroll,        // {si:{type:'monthly'|'hourly', base}}
  comments,       // {si:[{by,role,t,text}]}
  oneOnOne,       // {si:[{date,note,action}]}
  otRequests,     // [{id,si,date,hours,startTime,endTime,reason,status,by}]
  shifts,         // {si:{'YYYY-MM-DD':code}}
  otNotify,       // {si:[{date,hours,status}]}
  leaveNotify,    // {si:[{date,status}]}
  shiftPrefs,     // {periodKey:{si:{submitted,submittedAt,days:{ds:{start,end}|null}}}}
  shiftChangeReqs,// [{id,si,date,start,end,reason,status,by,createdAt}]
  shiftRemind,    // {si:true}
  shiftDeadline,  // 提出期限日（既定20）

  // ダッシュボード
  dashStaff,      // index | 'total'
  dashKpi,        // 選択中KPI名
  dashChart,      // Chart.jsインスタンス（render前にdestroy必要）

  // チーム
  teamKpi, teamTab,

  // 管理者
  adminTab,
  proxyStaff,
  correctionLog,

  // 出勤履歴
  histStaff,      // index | 'total'
  histTab,        // 'daily' | 'total'

  // 一時UI状態（NO_PERSIST）
  _prefillOtDate, // 残業申請フォームの日付プリセット（履歴チップクリック時）
  _prefillLvDate, // 有給申請フォームの日付プリセット（履歴チップクリック時）
}
```

### 主要関数

```js
render()              // 全画面再描画。冒頭で recalcPeriod()→initKpi()→ensureGoalFloor()→saveState()
sumKpi(si, kpi)       // スタッフsiのkpi「閲覧中月」合計
sumKpiAll(kpi)        // 全スタッフのkpi「閲覧中月」合計
calcRate(si, d)       // スタッフsiの日dの率指標オブジェクト
getDayLogs(si)        // スタッフsiの打刻集約
computeAlerts()       // 全スタッフ走査でアラート配列（KPI未達/欠勤/放棄率/PW/残業/休憩不足/深夜/KPI達成率80%未満）
alertsPanelHTML()     // アラートパネルHTML（スタッフは自分のみ・管理者は全員）
showOtApplyModal(ds)  // 残業未申請ポップアップ（出勤履歴チップクリック時）
attExtraHTML(si)      // 打刻ページ下部：シフト＋残業/有給申請フォーム（_prefillOtDate/_prefillLvDate 参照）
timeslotPage()        // 時間帯集計（個人の日別履歴ビューを含む）
saveState()           // LocalStorageへ保存（NO_PERSIST除外）
loadState()           // 保存stateを復元
exportKpiCSV()        // KPI入力ページのCSV出力
exportAttCSV()        // 出勤履歴のCSV出力
```

> **永続化キー**: `kpiSystemState_v2` / **NO_PERSIST**: `dashChart`, `clockInterval`, `_dashDrawChart`, `greetingMsg`, `greetingType`, `_prefillOtDate`, `_prefillLvDate`
> **日付定数**: `REAL_Y/REAL_M/REAL_D`＝実際の今日（打刻系）。`Y/M/DAYS/TODAY_D/LAST_D`＝閲覧中の月（recalcPeriodで更新）

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
crm_system.html    （顧客・案件・活動管理 CRM。LocalStorageキー: crmSystemState_v1）
```

### KPI ↔ CRM の共有設計
- `crm_system.html` は起動時に `kpiSystemState_v2` を読み込み、`_staffList`・`staffPw`・`adminPw`・`goals` を参照（書き込みなし）
- ログイン認証は KPI システムと同一
- KPI 側 topbar 右に「CRM」リンク、CRM 側 topbar 右に「KPI管理」リンク
