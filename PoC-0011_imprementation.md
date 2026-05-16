# PoC-0011_Spec（実装仕様＋評価仕様）
方向を見ないEntry条件と、first-touch以降の継続動意（Momentum）に基づくExit条件候補を見出すためのPoC実装仕様。

---

## 1. 目的

PoC-0011 の目的は以下の通り。

1. Entry時点では方向（UP/DOWN）を一切見ず、「動意（Momentumの前段階）」のみでEntryする。
2. Entry後、±0.03 に到達した first-touch によって方向が確定する。
3. first-touch 以降の価格推移と、10秒足・1分足・5分足のMomentumを記録する。
4. そのデータから「どのような継続動意パターンのときに伸びるか」を統計的に抽出し、Exitロジック候補を設計する。

本PoCは **Exitロジックを決めるためのデータ収集** が目的であり、EA内でExitは行わない（ログのみ）。

---

## 2. スコープ

- 実売買ロジックの完成ではなく、ログ収集用EAとして実装する。
- EntryはBuy固定（方向を見ないため）。
- Exitは行わず、first-touch後の価格推移をログに記録する。
- 評価はオフライン（Python/Excel等）で行う。

---

## 3. 実行環境

- MetaTrader 5
- MQL5
- シンボル: EURUSD
- テストモード: Tickベース
- ログ出力: CSV（Entry.csv / Touch.csv / Momentum.csv）

---

## 4. パラメータ仕様

外部パラメータ（input）は以下とする。

- Param_Symbol: "EURUSD"
- Param_LotSize: 0.01
- Param_Slippage: 3
- Param_MaxSpread: 3（pips換算）
- Param_TouchDistance: 0.0003（±0.03相当）
- Param_TouchTimeoutSec: 600（10分）
- Param_EntryLookbackSec: 10（動意判定用）
- Param_MinRange10s: 0.0001（直近10秒の最小値幅）
- Param_MinMACDAbs10s: 0.00001（10秒足MACD絶対値の下限）
- Param_QuietRangeSec: 60（レンジ判定用）
- Param_QuietMaxRange: 0.0002（レンジとみなす最大値幅）
- Param_UseTimeFilter: true
- Param_LogFilePrefix: "PoC-0011"

---

## 5. データ構造

### 5.1 EntryRecord（エントリ管理）

Entryごとに以下を保持する。

- ticket: ポジションチケット
- entry_time: Entry時刻
- entry_price: Entry価格
- spread_at_entry: Entry時スプレッド
- touched: first-touch済みか
- touch_direction: 1=UP, -1=DOWN, 0=NONE
- touch_time: first-touch時刻
- touch_price: first-touch価格
- max_favor_before_touch: Entry→first-touchまでの最大順行幅
- max_adverse_before_touch: Entry→first-touchまでの最大逆行幅

### 5.2 10秒足バッファ

- start_time
- open
- high
- low
- close

OnTickで秒境界を監視し、10秒ごとに新Barを生成する。

---

## 6. 全体フロー

### 6.1 OnInit

- ログファイル（Entry.csv / Touch.csv / Momentum.csv）を開く。
- 10秒足バッファ初期化。

### 6.2 OnTick

1. スプレッドチェック  
   Spread > Param_MaxSpread → 処理スキップ。

2. 時間帯フィルタ  
   Param_UseTimeFilter = true の場合、除外時間帯（例: 22〜23, 6〜8）はスキップ。

3. 10秒足バッファ更新  
   秒境界を跨いだら新Barを追加。

4. Entry判定（方向を見ない）  
   - ポジション未保有の場合のみ実行。
   - 動意条件・レンジ除外条件を満たせば Buy でEntry。
   - EntryRecordを作成し、Entry.csvに記録。

5. ポジション保有中  
   - first-touch未確定 → first-touch判定  
   - first-touch確定済み → Momentum計測（ログのみ）

### 6.3 OnDeinit

- ログファイルを閉じる。

---

## 7. Entryロジック（方向を見ない）

### 7.1 動意判定（10秒足）

以下を満たす場合に「動く準備が整っている」と判断する。

- 直近10秒の値幅 >= Param_MinRange10s
- 10秒足MACDの絶対値 >= Param_MinMACDAbs10s

MACDは10秒足Closeを使った簡易計算でよい。

### 7.2 レンジ除外（60秒）

- 直近60秒の値幅 <= Param_QuietMaxRange → レンジとみなしEntry不可。

### 7.3 Entry実行

- Buy固定でEntry。
- EntryRecordを作成し、Entry.csvに記録。

---

## 8. first-touch判定

### 8.1 判定条件

Buy前提で以下を計算する。

- favor = Bid - entry_price
- adverse = entry_price - Bid

以下のいずれかで first-touch 確定。

- favor >= Param_TouchDistance → UP方向
- adverse >= Param_TouchDistance → DOWN方向

タイムアウト（Param_TouchTimeoutSec）を超えた場合：

- touch_direction = 0（NONE）

### 8.2 first-touchログ（Touch.csv）

記録内容：

- entry_id
- time_entry
- time_touch
- touch_direction
- touch_price
- time_to_touch_sec
- max_favor_before_touch
- max_adverse_before_touch
- 10秒足MACD/STOCH（簡易）
- 1分足・5分足のトレンド状態（up/down/flat）

---

## 9. 継続動意（Momentum）計測

### 9.1 観測ウィンドウ

first-touch後の以下の時間幅を観測する。

- 60秒（1分）
- 180秒（3分）
- 300秒（5分）

### 9.2 計測内容

観測ウィンドウ内で以下を記録。

- max_favor_after_touch（同方向最大伸び幅）
- max_adverse_after_touch（逆方向最大逆行幅）
- sum_favor_after_touch（同方向累積値幅）
- sum_adverse_after_touch（逆方向累積値幅）

### 9.3 時間足との整合性

観測ウィンドウ終了時点で以下を取得。

- 10秒足MACD/STOCHの変化量（first-touch時との差分）
- 1分足MACD/STOCHの方向整合性（same/opposite/neutral）
- 5分足トレンド整合性（same/opposite/flat）

### 9.4 Momentumログ（Momentum.csv）

記録内容：

- entry_id
- touch_direction
- window_sec
- max_favor_after_touch
- max_adverse_after_touch
- sum_favor_after_touch
- sum_adverse_after_touch
- macd_10s_delta
- stoch_10s_delta
- trend_1m_align
- trend_5m_align

---

## 10. 評価仕様（オフライン分析）

### 10.1 基本集計

- Entry数
- first-touch発生率（UP/DOWN/NONE）
- first-touch到達時間の分布
- first-touch方向別の max_favor_before_touch / max_adverse_before_touch

### 10.2 継続動意と伸びの関係

- window_secごとに max_favor_after_touch の分布を算出
- macd_10s_delta / stoch_10s_delta と伸び幅の相関を分析
- trend_1m_align / trend_5m_align 別に伸び幅を比較

### 10.3 Exitルール候補の評価

Momentum.csv を使って以下の仮想Exitルールを評価する。

- ルールA：first-touch即利確（+0.03固定）
- ルールB：trend_1m_align = same かつ macd_10s_delta > 0 のとき +0.05利確
- ルールC：逆行が -0.01 を超えたら損切り

各ルールについて：

- 勝率
- 平均損益（期待値）
- リスクリワード比
- 最大ドローダウン（簡易）

を算出する。

---

## 11. 実装上の注意

- Entry条件に方向を一切入れない（Buy固定）。
- first-touch方向は「事後ラベル」として扱う。
- ログはヘッダ付きCSVで、1行1レコードのフラット構造にする。
- 10秒足は自前集計のため、秒境界処理を厳密に行う。
- ExitロジックはEA内で実装しない（ログのみ）。

---

## 12. 今後の拡張ポイント

- 継続動意の強弱をスコア化し、Exitロジックに直接利用する。
- 上位足（1分・5分）の環境認識をEntry条件に組み込む。
- first-touch距離（±0.03）の最適化。
- 動意判定の強化（ボラティリティ、Tick密度など）。

---

以上が PoC-0011 の完全実装仕様＋評価仕様。
