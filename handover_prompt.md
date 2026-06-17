# KPI管理システム — Claude Code 引き継ぎプロンプト

## ⚡ 次のセッションで最初にやること

着手前に必ず `kpi_system.html` を Read して現在の行数・関数配置を確認すること。

---

### 提案E（最優先）：シフト希望 一括反映ボタン

管理者が `state.shiftPrefs[nextMonthKey()]` の提出内容を `state.shifts` に一括転記するボタンを実装する。

**実装方針**

- `adminShift()` のシフト希望概覧テーブルの上部（または下部）に「**希望を一括シフト反映**」ボタンを追加
- クリック時にインライン2段階確認 → `shiftPrefs[nextMonthKey()]` を走査し、各スタッフ・各日について `start` が設定されている場合は `state.shifts[si][dateStr]` に `start` を書き込む（`null` = 絶対休み = `'休'` を書き込む）
- 未提出スタッフはスキップ（既存シフトを上書きしない）
- 完了後 `render()` して toast「シフト希望を反映しました（N名・M日分）」

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
| 打刻 | 出勤・休憩開始・休憩終了・退勤の4ボタン。退勤はインライン確認バーで実行。グリーティング表示。下部に**本日のシフト**＋**残業申請**フォーム（申請履歴/状態付き） |
| 出勤履歴 | 日別打刻一覧 / 総合時間集計の2タブ。管理者はスタッフ切替＋Total（全員）表示可。日別に**労務バッジ**（残業/長時間/休憩不足/深夜）。CSV出力可 |
| KPI入力 | 活動指標10項目・率指標8項目をカレンダー軸で入力。Total選択時は「担当者別」「日別合計」2タブ。値変更で即時保存 |
| 時間帯集計 | 区切り時刻を自由追加・削除。Total選択時は「時間帯別」「日別履歴」2タブ。各行右端にTotal列（時間帯入力値の横合計） |
| コールセンター | 着信数/応答数/通話時間(分)を日次入力 → 応答率/放棄率/AHTを自動計算。月次カード＋日次表。Total=担当者別 |
| チャット報告 | 個人 / チーム全体の切替。プレーン（LINE/Chatwork）/ Slackフォーマット切替。1クリックコピー |
| 個人ダッシュボード | Total選択時はチーム合算グラフ＋スタッフ別達成状況。個人選択時は本日入力テーブル＋**前月比**カード。下部に**フィードバックコメント**（双方向）＋**1on1記録**（管理者が記録） |
| チームダッシュボード | 上部に**アラートパネル**（未達/欠勤/放棄率/PW未変更/残業/休憩不足/深夜）。KPI別**前月比**。ランキング / 全員比較表の2タブ |
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
  workStart,      // {si: Date.now()} 出勤時刻ms（★個人別。共有端末でのタイマー混線を修正）
  breakStart,     // {si: Date.now()} 休憩開始ms
  totalBreak,     // {si: 累計休憩ms}
  workEndTime,    // {si: 退勤確定時の実働ms}

  // KPI ★v2: 最上位が期間キー「年_月」（例 '2026_5'）になった。各ページは必ずアクセサ経由で参照
  kpiData,        // {periodKey: {staffIndex: {day: {kpiName: value}}}} — kData() で閲覧中月分を取得
  goals,          // {periodKey: {staffIndex: {kpiName: targetValue}}}   — gData()
  activeStaff,    // KPI入力・時間帯で選択中のスタッフ index | 'total'
  kpiTab,         // 'act' | 'rate'
  kpiTotalView,   // 'staff' | 'daily'

  // 時間帯集計
  timeSlots,      // ['09:00','12:00',...] 区切り時刻配列
  timeData,       // {periodKey: {'si_kpi_slotIndex': value}} — tData()
  tsStaff,        // 時間帯選択スタッフ index | 'total'
  tsView,         // 'slot' | 'daily'

  // チャット報告
  reportTab,      // 'plain' | 'slack'
  reportScope,    // 'individual' | 'team'
  reportKpis,     // 表示するKPI名の配列
  reportStaff,    // 報告対象スタッフ index（activeStaffとは独立）

  // チャット報告（2026-06追加）
  reportPeriod,   // 'day'(日次/指定日) | 'month'(当月累計)
  reportDay,      // 日次レポートの対象日（1-31）

  // 管理者・目標設定（2026-06追加）
  goalStaff,      // 目標設定タブ専用のスタッフindex（activeStaffの'total'流入クラッシュ回避）

  // コールセンター・勤怠（2026-06追加）
  ccStaff,        // コールセンター指標ページの対象 index | 'total'（着信/応答/通話時間はkData内に格納）
  ccView,         // 'today'|'monthly'|'total' — コールセンター指標の表示モード（初期値 'today'）
  leaveData,      // {si:{granted:付与日数, taken:['YYYY-MM-DD',...]}} 有給。欠勤は打刻から自動算出
  leaveRequests,  // [{id,si,date,reason,status:'pending'|'approved'|'rejected',by}] スタッフ有給申請
  alertAck,       // {key:true} アラート確認済みフラグ（key = `pKey()_si_msg`）
  shiftNotify,    // {si:[{ds,code}]} シフト変更通知キュー（ログイン時にtoast → クリア）
  _noSample,      // true = 稼働開始後（clearOperationData()実行済み）。genDayData/ensureGoalFloor/backfillを抑止
  payroll,        // {si:{type:'monthly'|'hourly', base}} 給与（出勤実績連動・管理者「給与」タブ）
  comments,       // {si:[{by,role,t,text}]} フィードバック（双方向）
  oneOnOne,       // {si:[{date,note,action}]} 1on1記録（追加は管理者のみ）
  otRequests,     // [{id,si,date,hours,reason,status,by}] 残業申請（status: pending|approved|rejected）
  shifts,         // {si:{'YYYY-MM-DD':code}} シフト（code: 早|A|B|遅|夜|休。SHIFT_TYPES={c,label,bg,tx,start,end,nextDay} 参照）
  otNotify,       // {si:[{date,hours,status:'approved'|'rejected'}]} 残業承認通知キュー（ログイン時toast→クリア）
  leaveNotify,    // {si:[{date,status:'approved'|'rejected'}]} 有給承認通知キュー（ログイン時toast→クリア）
  shiftPrefs,     // {periodKey:{si:{submitted,submittedAt,days:{ds:{start,end}|null}}}} 翌月シフト希望
  shiftChangeReqs,// [{id,si,date,start,end,reason,status:'pending'|'approved'|'rejected',by,createdAt}] シフト修正依頼
  shiftRemind,    // {si:true} 未提出リマインド通知（管理者ボタン→次回ログイン時toast）
  shiftDeadline,  // 提出期限日（既定20。管理者「シフト」タブで変更可。clearOperationDataではリセットしない）

  // ダッシュボード
  dashStaff,      // index | 'total'
  dashKpi,        // 選択中KPI名
  dashChart,      // Chart.jsインスタンス（render前にdestroy必要）

  // チーム
  teamKpi, teamTab,

  // 管理者
  adminTab,       // 'proxy'|'correction'|'leave'|'goals'
  proxyStaff,     // 代理打刻対象スタッフ index
  proxyStatus,    // {si: status}
  proxyLog,       // 代理打刻ログ配列
  correctionLog,  // 打刻修正ログ配列

  // 出勤履歴
  histStaff,      // index | 'total'
  histTab,        // 'daily' | 'total'
}
```

### 主要関数

```js
render()              // 全画面再描画。冒頭で recalcPeriod()→initKpi()→ensureGoalFloor()→saveState() を実行
initKpi()             // 閲覧中月のサンプルデータ生成（冪等。未来月は空。初回のみ打刻backfill）
sumKpi(si, kpi)       // スタッフsiのkpi「閲覧中月」合計
sumKpiAll(kpi)        // 全スタッフのkpi「閲覧中月」合計
calcRate(si, d)       // スタッフsiの日dの率指標オブジェクトを返す
getDayLogs(si)        // スタッフsiの打刻を集約（当月=今日まで/過去月=月末まで）
calcWorkMin(inT,outT,brkS,brkE)  // 実働分数計算
fmtWorkMin(min)       // 分数→表示文字列 "Xh00m (X:00)"
fmt30(ms)             // ms→30分丸め表示文字列
rnd(min,max)          // ランダム整数
rndBelow(upper,pctMax)// ファネル用ランダム整数
ensureGoalFloor()     // 目標値の下限保証（閲覧中月の実績を下回る目標を引き上げ）
saveState()           // stateをLocalStorageへ保存（render()冒頭で毎回実行）
saveStateDebounced()  // 遅延保存（連続入力oninput用・400ms）
loadState()           // 保存stateを復元（実行時フィールド初期化＋表示月を当月へリセット）
resetData()           // 保存データ全消去→リロード
// ★v2 期間（年月切替）
recalcPeriod()        // state.viewYear/viewMonth から Y/M/DAYS/TODAY_D/LAST_D/IS_CURRENT/IS_FUTURE を再計算
pKey()                // 期間キー文字列 "年_月"
kData()/gData()/tData() // 閲覧中月の kpiData/goals/timeData 部分集合を返す（なければ生成）。全ページ必ず経由
periodNavHTML()       // 年＋月セレクタ（content上部の表示月バー）
bindPeriodNav()       // 期間ナビのイベント束縛（未来月は当月へクランプ）
// ★スタッフ管理
addStaff(name)        // 末尾に追加（既存indexは不変。_genクリアで新スタッフにもサンプル生成）。名前はsanitizeName適用
removeStaff(r)        // index r を削除し、index参照の全データを-1シフト再キー（最低1名は残す）
adminStaff()          // 管理者「スタッフ管理」タブのUI
// ★セキュリティ/UI共通（2026-06の検証修正で追加）
esc(s)                // HTMLエスケープ。innerHTMLへ入れる自由テキスト（レポート・修正理由等）に適用
sanitizeName(s)       // スタッフ名の危険文字除去＋40字制限。入力(addStaff/改名)・復元(loadState)で適用
toast(msg,type)       // 非ブロッキング通知（type: success|error|info）。alert/prompt全廃の代替
// ★CSVエクスポート（2026-06）
downloadCSV(name,rows)// 2次元配列→CSVダウンロード（Excel用UTF-8 BOM付与・toast通知）
csvCell(v)            // CSVセルのエスケープ（,"改行を""で囲む）
exportKpiCSV()        // KPI入力ページの現ビューを出力（個人=日×KPI / Total担当者別=担当者×KPI / Total日別=日×KPI）
exportAttCSV()        // 出勤履歴ページの現ビューを出力（個人=日別打刻+実働 / Total=全員サマリ）
// ★レスポンシブ（2026-06）
scrollKpiToToday()    // 横長KPI表で今日の列が画面外に隠れないよう自動スクロール（当月のみ・bindKpi末尾）
// ★個別認証（2026-06）
initAuth()            // 全スタッフPW・管理者PWを用意（render内で実行）
validatePw(pw)        // ポリシー検証（8文字以上・英/数/記号 各1文字以上）。OK=null/NG=メッセージ
mustChangePw(si)      // 今月パスワード変更が必要か（changedYM!==今月 or 未変更）
curYM()/fmtChangedYM()// 実際の今月キー / changedYM表示整形
pwchangeHTML()/bindPwChange() // 月初強制パスワード変更画面（state.page='pwchange'）
// ★コールセンター・勤怠・アラート（2026-06）
calcCC(si,d)/ccMonthly(si) // CC日次/月次（応答率・放棄率・AHT）。入力は KPI_CC=['着信数','応答数','通話時間(分)']
ccPage()/bindCc()     // コールセンター指標ページ
initLeave()/leaveOf(si)    // 有給データ用意 / 取得
absenceDays(si)       // 閲覧中月の欠勤（平日・打刻なし・有給でない）。leaveTakenThisMonth(si)も
computeAlerts()       // 全スタッフ走査でアラート配列。alertsPanelHTML()でチームDB上部に表示。シフト早退2日以上も含む
shiftDeviation(si,dateStr) // シフト終業時刻と実退勤打刻の乖離（±30分以内はnull。戻値:{diff,label,level:'warn'|'danger',type:'early'|'late'}）
// ★コメント・1on1・労務（2026-06）
staffNotesHTML(si)    // 個人ダッシュ下部のコメント＋1on1カード（bindDashboardでcmt-post/o1-add束縛）
laborIssues(si)       // 労務判定 {overtimeDays,longDays,shortBreakDays,lateNightDays,overtimeMin,perDay}
fmtTs(ms)             // コメント時刻の表示整形
// ★残業承認・シフト・前月比・CSV拡張（2026-06）
genDayData(v)         // 1日分のサンプル（ファネル＋CC）を冪等生成。initKpiとensurePeriodDataが共用
ensurePeriodData(yy,mm)/periodSum(yy,mm,si,kpi)/prevYM() // 前月比トレンド用（前月データ生成＋集計）
trendHTML(cur,prev)   // 前月比の▲▼表示
attExtraHTML(si)      // 打刻ページ下部：本日シフト＋残業申請フォーム
adminOt()/adminShift()// 管理者の残業承認タブ／シフトグリッド（クリックで循環）
exportCcCSV()/exportLeaveCSV() // CC・有給のCSV出力
exportShiftCSV()              // シフト表（担当者×日）CSV出力
// ★稼働開始・アラートACK・通知（2026-06-16追加）
clearOperationData()          // 全運用データ消去（_noSample=true設定 → リロード）。サンプル再生成を防ぐ
adminAttendanceOverview()     // 管理者の出勤状況ページ：全スタッフ状態サマリー＋本日打刻一覧
bindLeaveAck()                // adminLeave()の有給申請承認/却下ハンドラ（残日数チェック付き）
// ★役割別閲覧制限（2026-06-16追加）
// kpiPage/timeslotPage/ccPage/reportPage/dashboardPage：isAdmin分岐でstaffSelectと各状態を制御
// bindKpi/bindTimeslot/bindCc/bindDashboard：セレクタのnullガード追加。ハンドラのsi参照をロール分岐で統一
// exportKpiCSV/exportCcCSV：スタッフはstate.userのみ出力（Total不可）
// timeslotPage：時間帯スロットの追加・削除ボタンはisAdmin時のみ表示
```

> **永続化キー**: `kpiSystemState_v2`（v2でデータを期間キー化）/ **除外フィールド**: `NO_PERSIST`（dashChart・clockInterval・_dashDrawChart・greetingMsg・greetingType）
> **日付定数**: `REAL_Y/REAL_M/REAL_D`＝実際の今日（**打刻・代理打刻・修正フォーム既定は常にこちら**）。`Y/M/DAYS/TODAY_D`＝**閲覧中の月**（recalcPeriodで更新）。`LAST_D`＝実績のある最終日、`TODAY_D`＝今日ハイライト用（当月のみ、他月は-1）

---

## 既知の課題・未対応事項

1. ~~**データ永続化なし**~~ → **✅ 対応済み（2026-06）**：LocalStorageへ自動保存（キー `kpiSystemState_v1`）。`render()` 冒頭で `saveState()`、render()を通らない連続入力は `saveStateDebounced()`（400ms）。起動時 `loadState()` で復元し、ログイン状態・表示ページも含めて再開する。ログイン画面に「保存データをリセット」（インライン2段階確認）を設置。実行時オブジェクト（dashChart等）は `NO_PERSIST` で除外。プライベートモード等でlocalStorage不可でもtry/catchで動作継続
2. ~~**管理者目標値の下限なし**~~ → **✅ 対応済み（2026-06）**：`ensureGoalFloor()` が当月実績を下回る目標を「実績＋3〜18%」へ引き上げ。**初回生成時のみ実行**（`initKpi`内・firstGen）で、管理者の手動設定は上書きしない。解約数は対象外
3. ~~**スタッフ名・人数がハードコード**~~ → **✅ 対応済み（2026-06）**：管理者画面「スタッフ管理」タブで追加・改名・削除が可能。`STAFF_LIST` は `let` 化し `_staffList` キーで永続化。**削除時は `removeStaff()` がindex参照の全データ（kpiData/goals/timeData/attLog/proxyLog/各statusオブジェクト/選択中index）を再キーする**（r削除→r超は-1シフト）。最低1名は残す
4. **グリーティングメッセージ** — 管理者代理打刻時には表示されない（仕様通り）
5. **チーム比較・時間帯集計のTotal** — `timeData` のサンプルデータはランダム生成のため、実際の入力値とKPIデータとは連動していない
6. ~~**月次切替なし**~~ → **✅ 対応済み（2026-06）**：content上部の「表示月」バーで年＋月を切替（過去2年分＋当年）。データは期間キー化し各月独立。閲覧月へ移動すると不足分はその場でサンプル生成（未来月は閲覧不可＝当月へクランプ）。打刻/代理打刻は表示月に関係なく常に実際の今日へ記録。過去月のダッシュボードは「本日入力」カードを非表示
7. **ファネルが下流ほど0に偏る** — サンプルデータ生成上、契約数など下流KPIが0近くになりやすい（既存仕様。実データ運用なら問題なし）
8. **年範囲はハードコード** — 期間ナビの年は `REAL_Y-2`〜`REAL_Y`。それ以前を見るには `periodNavHTML()` の範囲を広げる

### 3エージェント検証で対応済み（2026-06）
- ✅ **クラッシュ修正**：`activeStaff='total'` の管理者目標画面流入 → `state.goalStaff`（数値専用）で根治
- ✅ **XSS対策**：スタッフ名 `sanitizeName()`（入力・復元）＋自由テキスト `esc()`（レポート・修正理由）
- ✅ **目標自動上書きの抑止**：`ensureGoalFloor()` を初回生成時のみに（手動設定を尊重）
- ✅ **日次/月次レポート**：`reportPeriod`（day/month）＋日付ピッカー。日次は指定日の値＋日割り目標比。表示KPIも10指標全選択可に
- ✅ **prompt/alert 全廃**：`toast()`＋インライン入力（時間帯追加は `<input type=time>`）
- ✅ **CSVエクスポート**（バックログ⑤）：KPI入力・出勤履歴の「CSV出力」ボタン。閲覧中の月・選択中ビューに連動。Excel互換（UTF-8 BOM）
- ✅ **スマホ対応**（バックログ⑥）：`@media(max-width:820px/480px)`。サイドバーはハンバーガー式ドロワー（背景オーバーレイ・`#nav-toggle`/`#nav-backdrop`）。grid-4/3→2列・grid-2→1列。トップバー時計非表示。KPI表は今日列へ自動スクロール。全ページで横はみ出しなしを確認
- ✅ **個別認証**（責任者指摘の最重要）：スタッフ別パスワード＋月初の強制変更＋ポリシー（8文字/英数記号）。管理者「スタッフ管理」タブでPWの確認・リセット・変更要求・管理者PW変更。**注意：確認欄の要件によりPWは平文でlocalStorage保存（実運用ではハッシュ化＋セキュアなリセット導線に置換推奨）**
- ✅ **打刻修正の実反映**：修正フォーム保存時に `correctionLog` だけでなく **`attLog` 本体を更新/追加**（出勤履歴・実働へ即反映）
- ✅ **有給・欠勤の実体化**：`state.leaveData={si:{granted,taken[]}}`。管理者「有給・欠勤」タブで付与日数・取得日（追加/削除）を管理。**欠勤は打刻実績から自動算出**（平日・打刻なし・有給でない日）。`removeStaff`で再キー
- ✅ **コールセンター指標**：ナビ「コールセンター」ページ。入力=着信数/応答数/通話時間(分)（`KPI_CC`・kData内）、自動計算=応答率/放棄率/AHT(分)。個人（日次表＋月次カード）/Total（担当者別）
- ✅ **アラート**：チームダッシュボード上部にパネル（`computeAlerts`）。コール大幅未達/欠勤3日以上/放棄率15%超/今月PW未変更 を要対応・注意で表示

- ✅ **フィードバック・コメント**：個人ダッシュ下部に双方向コメント（`state.comments={si:[{by,role,t,text}]}`）。誰でも投稿可
- ✅ **1on1記録**：個人ダッシュ下部に記録（`state.oneOnOne={si:[{date,note,action}]}`）。追加は管理者のみ・閲覧は全員
- ✅ **労務チェック**：`laborIssues(si)` が残業(>8h)/長時間(>10h)/休憩不足(労基45-60分)/深夜(22時以降)を判定。出勤履歴の日別にバッジ表示＋アラート連携（残業4日以上/休憩不足2日以上/深夜1日以上）

- ✅ **残業申請・承認フロー**：打刻ページから申請（`state.otRequests`）→ 管理者「残業承認」タブで承認/却下
- ✅ **シフト管理**：管理者「シフト」タブの担当者×日グリッド（セルクリックで 早→遅→夜→休→未設定 を循環・`state.shifts`）。打刻ページに本日のシフト表示
- ✅ **前月比トレンド**：個人ダッシュ（サマリーに前月比カード）＋チームダッシュ（KPI別前月比）。前月データは `ensurePeriodData` で自動用意
- ✅ **CSV対象拡大**：コールセンター（個人日次/担当者別）・有給欠勤 に「CSV出力」ボタン

### 1日運用テストで発見・修正（2026-06-12）
- ✅ **[重大] 実働タイマーのスタッフ間混線**：`workStart`/`breakStart`/`totalBreak`/`workEndTime` をグローバル値から**個人別 `{si:値}`** へ変更（共有端末で未出勤者に他人の「計測中…」が出る／実働時間が誤記録される問題を根治）。`removeStaff`で再キー
- ✅ **打刻修正の片側警告**：出勤/退勤の片側のみになる修正時に注意トーストを表示
- ✅ **承認残業の可視化**：出勤履歴の残業バッジを**残業(承認)/(未承認)**で色分け（承認済み残業申請と連動。`approvedOTH()`）
- ✅ **退勤打刻漏れアラート**：過去の完了日で出勤あり・退勤なしの日数をアラート表示（`missingOutDays()`）

### 4視点レビューで発見・修正（2026-06-12）
プロのシニアエンジニア / デザイナー / コールセンター運営責任者 / シフトアシスタントマネージャーの4視点でレビューし、再現確認の上で修正。
- ✅ **[重大] スタッフ削除でCC選択indexの再キー漏れ→クラッシュ**：`removeStaff()` が `state.ccStaff` を再キーしておらず、コールセンター指標ページで末尾スタッフを選択中に別スタッフを削除すると、次にCCページを開いた際に `kData()[si]` が undefined となり `TypeError`（画面が固まる）。他の全選択index（activeStaff/dashStaff/histStaff/goalStaff/tsStaff/reportStaff/proxyStaff）と同様に `reSel()` で再キーするよう1行追加して根治。
- ✅ **[中] 欠勤集計の定義が画面間で不一致**：出勤履歴（個人・Total）と出勤CSVが「平日／有給」を考慮しない素の式（`!d.inT && d.d<=…`）で欠勤を数えており、**土日や有給取得日まで欠勤としてカウント**。有給・欠勤タブやアラートが使う `absenceDays()`（平日・打刻なし・有給でない日）と数値が食い違っていた（サンプルデータでも10名中5名が不一致／有給取得日が欠勤表示）。履歴・CSVの該当3箇所を `absenceDays()` に統一し、アプリ全体で欠勤の定義を一本化。
- ✅ **[低] 前日の実働時間が「本日」として残存表示**：退勤後に翌日ログインして未出勤（status=off・本日の退勤打刻なし）の状態のとき、打刻ページの「実働時間（本日）」に前日の記録値（`workEndTime`）が表示されていた。**本日の退勤打刻がある場合のみ**表示する条件（`log.some(l=>l.type==='out')`）を追加して修正。
- 検証：4視点で約40のページ／タブ描画（staff・Total・admin・1名残し）をスモークテストしコンソールエラー0を確認。`removeStaff`の`ccStaff`再キー（'total'保持含む）・履歴の欠勤数（DOM）・打刻ページの実働表示を実機で再確認済み。

### 給与機能の追加＋残課題の修正（2026-06-12）
**管理者画面に「給与」タブを新設**（出勤実績にリアルタイム連動）。あわせて4視点レビューの残課題も対応。
- ✅ **給与ページ（月給制／時給制）**：`state.payroll={si:{type:'monthly'|'hourly',base}}`。管理者「給与」タブで雇用形態と基本給/時給を設定 → 打刻実績から給与を自動計算。
  - 時給制（アルバイト）：所定内（8h以内）×時給 ＋ **残業1.25倍** ＋ **深夜割増+0.25**（22時〜翌5時）。欠勤控除なし。
  - 月給制（正社員）：基本給固定 − **欠勤の日割り控除**（基本給÷当月平日×`absenceDays`）＋ 残業/深夜手当（時間単価＝基本給÷(当月平日×8h)）。
  - 雇用形態select・金額inputの変更は**即時に支給額・支給総額へ反映**（`recalcPayRow`でrenderを通さずDOM更新）。CSV出力対応（`exportPayrollCSV`）。`removeStaff`で`payroll`再キー。初期サンプルは約2/3を月給制・約1/3を時給制。
  - 主要関数：`calcPay(si)`/`initPayroll()`/`payOf(si)`/`workdaysInMonth(y,m)`/`nightMinutesOf(inT,outT)`/`adminPayroll()`/`exportPayrollCSV()`。render内で`initPayroll()`を実行。
- ✅ **[残課題] 夜勤の日跨ぎ打刻に対応**：`calcWorkMin` を「退勤＜出勤＝翌日」とみなすよう変更（17:00→翌2:00＝9h）。退勤打刻時、本日未出勤かつ前日に未退勤の出勤があれば**退勤を前日キーに記録**（`punch`内）。深夜判定は新規`nightMinutesOf`（22時〜翌5時の労働分）に置換し、`laborIssues`の深夜/残業・給与の深夜手当が日跨ぎでも正しく算出。
- ✅ **[残課題] CCにサービスレベル(SL)を追加**：入力に`20秒内応答数`を追加（`KPI_CC`拡張・`genDayData`が既存日にも遡って補完）、SL＝20秒内応答/着信 を自動計算（`calcCC`/`ccMonthly.slRate`）。冗長だった放棄率カードをSLカードに置換（放棄率はサブ表示＋表に保持）。日次行・Total列・CSVにSL追加。
- ✅ **[残課題] `workStart`の過大表示を抑止**：稼働中の実働計算が20時間超（前日からの退勤打刻漏れ）の場合は「退勤打刻が必要です」と表示し、誤った実働時間を本日分として出さない。
- ✅ **[残課題] コントラスト改善**：`--text-tertiary` を `#999`→`#737373`（WCAG AAを満たす濃さ）。
- 検証：日跨ぎ実働/深夜の算出、給与の独立再計算一致（時給・月給）、給与の即時再反映（DOM）、約40ページ/タブのスモーク、`removeStaff`の`payroll`再キーを実機確認。コンソールエラー0。
- 注：パスワードは管理者の平文確認要件のため**平文保存を維持**（ハッシュ化は依頼により見送り）。

### シフトと実打刻の乖離チェック（2026-06-15）
- ✅ **シフト乖離チェック**：`SHIFT_TYPES` に `start`/`end`/`nextDay` フィールドを追加。`shiftDeviation(si, dateStr)` が終業時刻との差分を返す（±30分以内は誤差として null。差が正=定時超、負=早退。夜勤 `nextDay:true` は終業を+1440分・深夜0〜5時台の退勤を翌日扱いで正規化）。
  - 出勤履歴の日別打刻行に**乖離バッジ**を追加（早退=danger/warn チップ、定時超=warn チップ）。labor flags (残業/休憩不足/深夜) が空でも表示する。
  - `computeAlerts()` に**早退アラート**を追加：早退60分以上が月2日以上の場合 `danger` アラート。
  - 既存の残業アラートは `laborIssues()` ベースで継続。シフト乖離バッジは `shiftDeviation()` で独立判定なので残業承認状態と競合しない。

### コールセンター管理者テスト→フィードバック修正（2026-06-16）
運用開始テストを兼ねてコールセンター管理者が11項目のフィードバックを提出。重大→低の順で全対応。

**稼働開始リセット**
- ✅ `clearOperationData()` を新設（`resetData()` の傍に配置）：kpiData/timeData/_gen/attLog/attStatus/workStart〜/proxyLog〜/otRequests/comments/oneOnOne/**leaveRequests/alertAck/shiftNotify** を全消去 → リロード。
- ✅ `state._noSample=true` フラグ：消去後にサンプルデータが再生成されないよう `genDayData()`・`ensureGoalFloor()`・`initKpi()` のバックフィルループに guard を追加。
- ✅ 管理者画面「データ管理」タブに「稼働開始リセット（サンプルデータ全消去）」ボタンを設置（インライン2段階確認）。

**目標設定 — 全体枠（チーム合計）の追加**
- ✅ `state.goalStaff` に `-1` を追加（番兵値 = 「全体」）。`adminGoals()` でスタッフリストの末尾に「【全体】チーム合計」オプションを配置。
- ✅ 全体表示時は各KPIに個人目標の合計を自動表示（読み取り専用・変更不可）。実績も `sumKpiAll(k)` で全員集計。達成率・保存ボタンは全体選択時は非表示。
- ✅ `removeStaff()` の goalStaff 再キー：`state.goalStaff===-1?-1:remap(goalStaff)===null?0:remap(goalStaff)` で番兵を保護。

**出勤状況管理（管理者用）**
- ✅ `adminAttendanceOverview()` を追加：管理者が打刻ページを開いたとき（`state.role==='admin'`）に呼び出す個別ビュー。全スタッフの状態（稼働中/休憩中/未出勤）サマリーカード3枚 + 本日打刻一覧テーブル（出勤・退勤・休憩・復帰時刻・実働・シフト）。
- ✅ `attendancePage()` を `role==='admin'` 分岐に変更（スタッフは従来通り個人ビュー）。

**打刻忘れ案内バナー（#7）**
- ✅ `attendancePage()` スタッフビューの上部に `missingOutDays(si)>=1` のとき赤バナーを表示。「管理者に打刻修正を依頼してください」の案内文付き。

**コールセンター指標 — 今日ビュー**
- ✅ `state.ccView:'today'|'monthly'|'total'` を追加（初期値 `'today'`）。ツールバーにタブトグル `[data-ccview]` を追加。
- ✅ 今日ビュー（`ccView==='today'`）：本日の入力フォーム + 本日のCC指標カード（応答率/放棄率/AHT/SL）を表示。
- ✅ `bindCc()` に `[data-ccview]` クリックハンドラを追加。

**アラート 確認済みフロー**
- ✅ `state.alertAck:{}` を追加（キー = `pKey()_si_msg`）。
- ✅ `alertsPanelHTML()` でアクティブアラートに「確認済みにする」ボタン `[data-ack-key]`、確認済み件数の折りたたみ表示、「全て確認済みにリセット」ボタン `#alert-clear-all` を追加。
- ✅ `bindTeam()` にACKハンドラとリセットハンドラを追加。

**シフト変更通知**
- ✅ `state.shiftNotify:{si:[{ds,code}]}` を追加。管理者がシフトセルをクリックして変更した際に対象スタッフの通知キューへ追加（同日重複排除）。
- ✅ `bindLogin()` のスタッフログイン成功時に `shiftNotify[si]` があれば `toast()` で通知 → キューをクリア。

**有給申請フロー（スタッフ側）（#8）**
- ✅ `attExtraHTML(si)` に有給申請カードを追加：申請日ピッカー・理由テキスト・「申請」ボタン + 直近6件の申請履歴テーブル（pending/approved/rejected 色分け）。残日数表示。
- ✅ `bindAttendance()` に `#lv-submit` ハンドラを追加：残日数チェック・同日重複チェック → `state.leaveRequests.push(...)` → toast。
- ✅ `adminLeave()` の上部に保留中の有給申請パネルを追加（承認→`leaveData.taken`追加 / 却下）。`bindLeaveAck()` 関数で承認/却下処理。`bindAdmin()` から `bindLeaveAck()` を呼び出し。

**打刻修正 — 定型文プリセット（#9）**
- ✅ `adminCorrection()` の修正理由欄にプリセット `<select id="cor-reason-preset">` を追加（操作ミス・打刻忘れ・システム障害・端末トラブル・在宅代理入力）。選択でテキスト入力へ自動転写。`bindAdmin()` に `onchange` ハンドラを追加。

**シフト表CSVエクスポート（#10）**
- ✅ `exportShiftCSV()` を追加：担当者×日グリッドのシフトコードをCSV化（`シフト表_YYYY年M月.csv`）。`adminShift()` に「シフトCSV出力」ボタン `#shift-csv` を追加。`bindAdmin()` からハンドラを追加。

**1on1記録 — 閲覧可能を明示（#11）**
- ✅ `staffNotesHTML()` のセクションタイトルに「管理者・本人が閲覧可能」バッジを追加。
- ✅ スタッフビュー（フォームなし）の tip 文を「1on1の記録は閲覧のみ可能です（追記は管理者が行います）」に変更。

**保存インジケーター**
- ✅ `saveStateDebounced()` が保存完了後に固定ポジション（右下）に「✓ 保存済み」バッジを1.2秒間表示するよう更新（`#_save-ind`要素を動的生成）。

### 役割別の閲覧制限の厳密化（2026-06-16）
- ✅ **KPI入力・時間帯集計・CC指標・チャット報告・個人ダッシュボード**：スタッフロールはスタッフ切替セレクタ・Total ビューを非表示。自分のデータのみ表示・編集。
- ✅ **bindXxx()** 関数のセレクタ null ガード追加（スタッフ時に要素が存在しない場合のエラー防止）。
- ✅ **exportKpiCSV() / exportCcCSV()**：スタッフが CSV を出力する際も自分のデータのみ出力されるよう制限。
- ✅ **bindDashboard()** のコメント投稿・1on1記録・drawChart の si 参照をロール分岐で統一。

### 6月テスト（CC責任者・未経験者想定）→バグ修正（2026-06-16）
CC責任者（田中 一郎）・未経験者（佐藤 花子）を含む30日間の全業務シナリオを追跡。

- ✅ **[中] 代理打刻が管理者出勤状況ビューに反映されない**：`proxyPunch()` が `state.proxyStatus[si]` のみ更新し `state.attStatus[si]` を更新しない → `adminAttendanceOverview()` が後者を参照するため代理打刻後も「未出勤」と表示されていた。両方を同時更新するよう修正。
- ✅ **[低] 有給申請を管理者が残0でも承認できる**：`bindLeaveAck()` に残日数チェックを追加。残0の場合は承認ブロック（エラートースト表示）。
- ✅ **[低] スタッフが時間帯スロット（全員共通）を追加・削除できる**：`state.timeSlots` は全スタッフ共有の設定値だが、スタッフロールでも追加ボタン・削除ボタンが表示されていた。管理者のみに制限（`isAdmin&&!isTotal` ガード）。
- ✅ **[軽微] CCページのスタッフビューに担当者名表示なし**：スタッフがコールセンター指標ページを開いたとき、誰のデータを見ているか表示されていなかった。ツールバーに自分の名前バッジを表示するよう追加。

### 時間帯集計とKPIデータの連動（2026-06-16追加）
- ✅ **サンプルデータを kpiData 月次合計から比例配分**：個人ビューの初期値がランダム(0-19)だった問題を修正。初回表示時に `sumKpi(si,k)` をランダム重みで時間帯数に分割し、最終スロットに端数を割り当てることでスロット合計 = KPI月計が保証される。
- ✅ **`_noSample=true` 時は 0 スタート**：稼働開始リセット後はランダム埋めを行わず全スロット 0 で初期化（実データ入力を促す）。
- ✅ **「KPI月計」参照列を追加**：個人ビュー・Total時間帯別ビューの右端に `sumKpi(si,k)` / `sumKpiAll(k)` を薄灰背景で表示。
- ✅ **「合計」列のカラー化**：スロット合計 = KPI月計なら緑（`#2E7D32`）、不一致なら赤（`#C62828`）で色分け。
- ✅ **入力時にリアルタイム合計更新**：`bindTimeslot()` の `oninput` ハンドラで、`closest('tr')` から同行の inputs を集計し `.total` セルを即時更新・再カラー化（`data-kmonthly` 属性から参照）。renderを通さず軽量に反映。
- ✅ **率指標タブを追加**：`state.tsTab='act'|'rate'`。ツールバーに「活動指標 / 率指標（自動計算）」タブを追加（日別履歴では非表示）。`calcSlotRate(v)` が活動値オブジェクトから8種の率を算出。個人ビュー・Total時間帯別ビューの両方でスロット別率指標を read-only 表示。月次列は `sumKpi(si,k)` / `sumKpiAll(k)` から月次総計の率を表示。

### 複数機能追加（2026-06-17）
- ✅ **残業申請に時間帯フィールド追加**：`attExtraHTML` のフォームに `ot-start`/`ot-end`（type=time）を追加。`otRequests` の各オブジェクトに `startTime`/`endTime` フィールドを保存。スタッフ申請履歴テーブル・`adminOt()` 管理者テーブルの両方に「時間帯」列を追加。
- ✅ **出勤履歴に有給チップ表示**：`histDailyContent()` で `ds`（日付文字列）と `isLeave`（`leaveOf(si).taken` に含まれるか）を計算。無打刻日が有給の場合は緑チップ「休暇 / 有給」を表示、欠勤と区別。
- ✅ **解約率を KPI_RATE に追加**：`KPI_RATE` 配列の末尾に `'解約率'` を追加。`calcRate()` と `calcSlotRate()` に `解約率 = 解約数/契約数` を追加。KPI入力の率指標タブと時間帯集計の率指標タブの両方に反映。
- ✅ **CC今日入力にアウトバウンド KPI を追加**：今日ビューの下部に「アウトバウンド KPI（本日・KPI入力より）」セクションを追加。クロージング率（クロージング数/完了数）・依頼数・契約数・解約率（解約数/契約数）を kpiData[si][REAL_D] から read-only カード表示。解約率が10%超で赤色。

### CRMシステム新規作成（2026-06-18 最終）
- ✅ **`crm_system.html` を新規作成**：顧客・案件・活動記録・ダッシュボード・管理者ビューを単一HTMLで実装（バニラJS・LocalStorage永続化）
- ✅ **ログイン**：`kpiSystemState_v2` からスタッフ一覧・パスワード・管理者パスワードを読み込み、KPI システムと同一認証
- ✅ **ダッシュボード**：月末着地予測（今月成約件数＋パイプライン確度加重件数）、予測コメント自動生成（達成見込み/不足/大幅不足）、パイプラインフェーズ別集計、最近の活動タイムライン
- ✅ **顧客リスト**：CRUD（追加・編集・削除）、ステータスフィルター（リード〜失注）、担当者フィルター（管理者のみ）、フリーワード検索
- ✅ **顧客詳細**：アクティビティタイムライン、紐づき案件一覧、累計成約件数・金額、関連案件/活動を顧客詳細から直接追加
- ✅ **案件管理**：フェーズ（リード→クロージング）、件数・金額・確度・クロージング予定日、進行中/成約/失注の状態管理、フェーズ/担当者/状態フィルター
- ✅ **活動記録**：電話/訪問/メール/その他の4種別、顧客紐づけ、担当者フィルター（管理者のみ）
- ✅ **管理者ビュー**：スタッフ別サマリー（顧客数・進行中案件・今月成約件数・成約金額・目標・達成率・差分・活動数）、チーム合計行付き
- ✅ **表示制限**：スタッフは件数のみ表示（金額列は管理者のみ）
- ✅ **KPI↔CRM相互ナビ**：KPI topbar に「CRM」ボタン追加（`crm_system.html` へ）、CRM topbar に「KPI管理」ボタン（`kpi_system.html` へ）

### 提案F・シフト希望UI改善・管理者タスクアラート（2026-06-18 後半）
- ✅ **提案F — シフト提出期限を管理者が変更可能に**：`state.shiftDeadline`（既定20）を追加。管理者「シフト」タブの希望一覧ヘッダーに「提出期限：毎月 \_\_ 日」入力欄（1〜28）を配置。変更時 `saveState()`＋`render()` で即時反映。`computeAlerts()`・`shiftPrefCardHTML()` のハードコード`>=20`を `state.shiftDeadline` に統一。`clearOperationData()` ではリセットしない（設定値）。後方互換ガードを `loadState()` に追加。
- ✅ **シフト希望カレンダー：出退勤時間を個別プルダウン化**：従来の PREF_SLOTS 固定1セレクト → 出勤時間（06:00〜23:30）＋退勤時間（06:00〜翌05:30）の2セレクト。出勤時間選択で退勤プルダウンが出現。「— 休み扱い」と「🔴 絶対休み」は出勤プルダウン先頭。提出済み表示では翌日越えを「翌HH:MM」表示。`timeOpts(sel,startH,endH)` ヘルパーを追加。
- ✅ **シフト修正依頼フォーム：出退勤プルダウン化**：`shiftchg-slot`（PREF_SLOTS固定）→ `shiftchg-start`＋`shiftchg-end` に分離。「🔴 休み希望」選択時は退勤欄を非表示。デフォルト09:00〜18:00。
- ✅ **管理者タスクアラートパネル**：`adminTaskAlerts()`＋`adminTaskPanel()` を追加。管理者画面トップに「対応が必要なタスク N件」パネルを常時表示。残業承認待ち（赤）・有給承認待ち（赤）・シフト修正依頼（黄）・PW未変更スタッフ（黄）・KPI/時間帯未入力（青、稼働開始後のみ）を色分けリスト。クリックで対象タブ/ページへ移動。全タスク解消時は自動消滅。

### 複数機能追加（2026-06-18）
- ✅ **承認通知（残業・有給）**：`state.otNotify`/`state.leaveNotify`（`{si:[{date,hours,status}]}`）を追加。管理者が承認/却下時にキューへプッシュ → スタッフ次回ログイン時にトースト表示 → クリア。`clearOperationData()`・`removeStaff()` の対象に追加（`reIntObj()` で再キー）。
- ✅ **ログイン画面🔔未読通知インジケーター**：`loginHTML()` のスタッフ選択 `<option>` に `otNotify[si]`/`leaveNotify[si]`/`shiftRemind[si]` のいずれかがあれば `🔔` を付与。
- ✅ **業績サマリー Total ビューにコール達成率バー列追加**：`summaryPage()` の actRows/actTotRow に `コール達成率` 列（プログレスバー＋%）を追加。100%以上=緑、70%以上=オレンジ、未達=赤。
- ✅ **シフト希望提出フロー**：`state.shiftPrefs`（`{periodKey:{si:{submitted,submittedAt,days:{ds:{start,end}|null}}}}`）を追加。`PREF_SLOTS`（5スロット）定数で時間帯選択。`shiftPrefCardHTML(si)` が翌月カレンダーUIを生成。前月20日締切・絶対休み指定・バリデーション（日+時間のセット必須）。`bindAttendance()` で `#shiftpref-submit` ハンドラ。
- ✅ **シフト修正依頼フロー**：`state.shiftChangeReqs`（`[{id,si,date,start,end,reason,status,by,createdAt}]`）。スタッフが `attExtraHTML` から申請 → 管理者 `adminShift()` で承認/却下 → 承認時に `state.shifts[si][date]` へ PREF_SLOTS の `code` を自動反映。管理者は手動でシフトグリッドも変更可。
- ✅ **シフト未提出アラート**：`computeAlerts()` に翌月シフト希望未提出アラートを追加（`REAL_D >= 20` かつ未提出の場合）。
- ✅ **提案D — 未提出者へ再通知**：管理者 `adminShift()` に「未提出者に通知」ボタンを追加。クリックで未提出スタッフの `state.shiftRemind[si]=true` をセット → 次回ログイン時にトースト通知（`bindLogin()`・1500ms遅延）。
- ✅ **SHIFT_TYPESにA番(10-19)/B番(11-20)追加**：`start`/`end`フィールドを持つ新シフト区分。
- ✅ **セキュリティ設定**：`.claude/settings.json` にdenyルール（rm/curl/wget/nc/ssh/scp/find /Users）を追加。`find`は相対パス（`find .`）のみ使用すること。
- ✅ **GitHub連携**：https://github.com/shachi-orca/KPI- にプッシュ済み。

### 未対応（次以降の候補）
- **【次セッション最優先】提案E**：シフト希望を管理者がワンクリックで一括シフト反映（`state.shiftPrefs` → `state.shifts` 転記）
- **CRM拡張**（`crm_system.html`）：提案・商談フェーズの詳細メモ、ファイル添付相当のリンク欄、活動レポートCSV出力
- 目標の複数月トレンドグラフ
- PWは平文保存（ハッシュ化推奨・管理者平文確認要件で見送り中）
- ファネル下流の0偏り／期間ナビの年範囲ハードコード

---

## ファイル構成

```
kpi_system.html    （KPI・シフト・勤怠管理。LocalStorageキー: kpiSystemState_v2）
crm_system.html    （顧客・案件・活動管理 CRM。LocalStorageキー: crmSystemState_v1）
```

### KPI ↔ CRM の共有設計
- `crm_system.html` は起動時に `kpiSystemState_v2` を読み込み、`_staffList`・`staffPw`・`adminPw`・`goals` を参照する（書き込みは行わない）
- ログイン認証は KPI システムと同一（スタッフ名・パスワードが共通）
- 月末着地予測の「目標」は `goals[pKey()][si]['契約数']` を参照（`state.goalOverride` で CRM 側が独自上書き可能）
- KPI 側 topbar 右に「CRM」リンク、CRM 側 topbar 右に「KPI管理」リンクで相互遷移

---

## 開発ルール・お作法

- **render() を基本単位**として扱う。状態変更後は必ず `render()` を呼ぶ（`render()` 冒頭で `saveState()` が走り自動保存される）
- **render()を通らない連続入力（oninput）では `saveStateDebounced()` を明示的に呼ぶ** — KPI入力・時間帯入力・ダッシュボード当日入力・目標設定の各oninputが該当
- **KPI/目標/時間帯データは必ず `kData()`/`gData()`/`tData()` 経由で参照**（`state.kpiData[...]` 直接アクセス禁止＝期間キーを飛ばしてしまう）
- **「現在」操作（打刻・代理打刻・修正フォーム既定値）は `REAL_Y/REAL_M/REAL_D` を使う**。`Y/M/TODAY_D` は閲覧中の月なので打刻に使わない
- **新しい日付ループは `DAYS`（月の日数）と `LAST_D`（実績のある最終日）を使い分ける**。今日ハイライトは `IS_CURRENT && day===TODAY_D`
- **スタッフはindex（si）で全データを参照している**。スタッフ追加は末尾のみ（index不変）。削除は必ず `removeStaff()` 経由で全データを再キーすること（生のspliceは厳禁＝データがずれる）。`STAFF_LIST` は `let`・`_staffList` で永続化
- **ページ関数は `xxxPage()` + `bindXxx()` のペア**で実装する
- `state.dashChart` は `render()` 冒頭で必ず `destroy()` してから再描画
- **confirm() / alert() / prompt() は使わない** — 成功/失敗通知は `toast(msg,type)`、確認はインラインバー、入力は `<input>` で代替（2026-06に全廃済み）
- **innerHTMLへ入れる自由テキストは必ず `esc()`**（レポート出力・修正理由など）。**スタッフ名は入力・復元時に `sanitizeName()`** 済みなので各表示箇所での再エスケープは不要
- **目標設定は `state.goalStaff`（常に数値）を使う**。`activeStaff` は `'total'` になり得るため目標・管理画面で直接使わない
- **横長テーブルは必ず `.kpi-table-wrap`（overflow:auto）で包む**。直接置くとモバイルで横はみ出しする。ドロワー開閉はDOMのclassトグル（renderを通さない・`bindNav`）／ナビ選択時はrenderで閉じる
- **認証情報は `state.staffPw`（index si）/`state.adminPw`**。スタッフ削除時は `removeStaff()` が `staffPw` も再キー。新スタッフは `initAuth()` が初期PWを付与。月初判定は `REAL` の年月（`mustChangePw`）。PWは平文保存（管理者確認欄の要件）
- `state.activeStaff` は `'total'` (文字列) になる場合があるため、`parseInt()` ではなく値チェックが必要
- `state.reportStaff` はチャット報告専用。`activeStaff` とは独立して管理する
- 率指標は必ず **Math.min(100, ...)** で100%クランプ
- サンプルデータはファネル構造で生成（コール数 >= 完了数 >= 契約数 の大小関係を保証）

---

## 次にやること候補（優先度順）

1. ~~目標値の下限保証~~ ✅ **完了**（`ensureGoalFloor()`）
2. ~~LocalStorageへのデータ永続化~~ ✅ **完了**（`saveState()` / `loadState()`）
3. ~~スタッフ管理画面（名前・人数の編集）~~ ✅ **完了**（管理者「スタッフ管理」タブ／`addStaff`・`removeStaff`）
4. ~~月次切替（過去月データ参照）~~ ✅ **完了**（年＋月セレクタ／期間キー化）
5. ~~CSVエクスポート機能~~ ✅ **完了**（KPI入力・出勤履歴に「CSV出力」ボタン／`exportKpiCSV`・`exportAttCSV`）
6. ~~スマホ対応（レスポンシブ）~~ ✅ **完了**（ハンバーガー式ドロワー＋`@media`／グリッド流動化／KPI表は今日列へ自動スクロール）

**→ 当初バックログ①〜⑥はすべて完了。** 以降は下記「未対応の大型課題」（認証・勤怠実体化・CC指標等）が候補。

---

## Claude Codeへの作業指示

`kpi_system.html` を作業ディレクトリに配置した上で、上記の情報をもとに開発を継続してください。

修正・追加を行う際は：
1. 既存の `render()` / `bindXxx()` パターンを踏襲する
2. `state` の追加プロパティは初期値を必ず宣言する
3. `confirm()` / `alert()` を使わずインラインUIで実装する
4. `str_replace` で差分修正するか、必要なら全体書き直し（ファイルは単一なので全書き出しも可）
