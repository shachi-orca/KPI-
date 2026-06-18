# KPI管理システム / CRM — Claude Code 引き継ぎプロンプト

## ⚡ 次のセッションで最初にやること

着手前に必ず対象ファイルを Read して現在の行数・関数配置を確認すること。

---

### 【次優先】未対応事項（2026-06-19時点）

| ファイル | 内容 | 状態 |
|---|---|---|
| `crm_system.html` | 活動記録ページを削除 | **次セッションで実行** |
| `crm_system.html` | 案件管理：CRMの「完了」顧客から自動転記 | **次セッションで実装** |
| `crm_system.html` | KGIダッシュボード：CRMデータ連携 | **次セッションで実装** |
| `kpi_system.html` | 目標の複数月トレンドグラフ | 未着手 |
| `kpi_system.html` | ファネル下流の0偏り / 期間ナビの年範囲ハードコード | 見送り中 |
| `kpi_system.html` | パスワードの平文保存 | 管理者確認要件で見送り中 |

---

### 次セッションの実装詳細

#### ① 活動記録ページ削除（`crm_system.html`）

削除対象：
- サイドバーの「活動記録」ナビ（`{id:'activities',...}`）
- `activitiesPage()` 関数
- `exportActivityCSV()` 関数
- `actModal()` 関数
- `showActModal()` 関数
- `bindPage()` 内の活動関連バインド（`act-filter-type`, `act-filter-staff`, `act-csv-btn`, `act-add-btn`, `data-act-del`）
- state の `actFilter` 参照

#### ② 案件管理：CRMから自動転記（`crm_system.html`）

方針：
- KGI独自の顧客DBは廃止済み（顧客リストページ削除済み）
- CRM（`crmCustomerState_v1`）の顧客ステータスが「**完了**」になった対応履歴（contacts）を案件として自動転記
- 転記元：`D.contacts`（`crmCustomerState_v1`）の `status === '完了'` の対応
- 転記先：`state.deals`（`crmSystemState_v1`）
- 重複防止：contactの `id` をキーに既存チェック
- 案件のタイトル：顧客名 + 商材名（contact の pid → products から取得）
- 案件の状態：`won`（成約）として登録
- 担当者：contact の `si`（スタッフindex）を引き継ぐ

読み込み処理は `loadKpiShared()` に相当する `loadCrmData()` 関数を追加し、`render()` の先頭で呼ぶ。

```js
// 追加するデータ読み込み（crm_system.html の先頭付近に追加）
let crmCustomerState = null;
function loadCrmData() {
  try {
    const raw = localStorage.getItem('crmCustomerState_v1');
    if (!raw) return;
    crmCustomerState = JSON.parse(raw);
  } catch(e) {}
}
// 自動転記処理（render() の先頭で呼ぶ）
function syncDealsFromCrm() {
  if (!crmCustomerState) return;
  const contacts = crmCustomerState.contacts || [];
  const customers = crmCustomerState.customers || [];
  const products = crmCustomerState.products || [];
  contacts.filter(h => h.reason === '新規提案' /* または status判定 */).forEach(h => {
    // TODO: CRM側のステータス更新タイミングと連携ロジックを要設計
  });
}
```

> **設計注意**：CRM（customer_crm.html）のcontactsには「対応後ステータス」(`contactStatus`)が含まれる。顧客ステータスを「完了」に変更したcontactを転記トリガーとする。

#### ③ KGIダッシュボード：CRMデータ連携（`crm_system.html`）

方針：
- `dashboardPage()` の「今月の成約数」をKGI独自の `state.deals` だけでなく、CRMの「完了」データも合算して表示
- 「失注」情報はCRMの対応理由「クレーム対応」など特定の区分をマッピング（要確認）
- KPIの契約数（`kpiState.kpiData`）も引き続き参照

---

## 今セッション（2026-06-19）で完了した実装

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

KGIに残すページ：ダッシュボード、案件管理、管理者ビュー
KGIから削除するページ：顧客リスト（削除済み）、活動記録（次セッション）

---

## プロジェクト概要

### ファイル構成・行数

```
kpi_system.html     （KPI・シフト・勤怠管理。LocalStorageキー: kpiSystemState_v2）約3896行
crm_system.html     （KGI管理システム。LocalStorageキー: crmSystemState_v1）約810行
customer_crm.html   （顧客管理CRM。LocalStorageキー: crmCustomerState_v1）約946行
```

### KPI ↔ KGI(crm_system.html) の共有設計
- `crm_system.html` は起動時に `kpiSystemState_v2` を読み込み、`_staffList`・`staffPw`・`adminPw`・`goals`・`role` を参照（書き込みなし）
- KPIシステムで管理者ログイン中にKGIを開くと自動的に管理者セッションを同期
- スタッフがKGIにアクセスするとアクセス拒否画面を表示

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
  appointments: [{id, cid, si, at, note}],
  _cid, _hid, _aid
}
```

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
