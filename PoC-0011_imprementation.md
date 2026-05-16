# PoC-0011_Spec（実装仕様＋CSVスキーマ＋分析ルール）

本ドキュメントは、PoC-0011 の実装仕様・CSVスキーマ・分析ルールを統合したものである。  
PoC-0010 の PoC 管理ルールに準拠し、PoC-0011 の Entry / Touch / Momentum の3段階ログを収集し、  
first-touch 以降の継続動意（Momentum）を分析するための基盤を提供する。

---

# 1. 目的

PoC-0011 の目的は以下の通り。

1. Entry時点では方向（UP/DOWN）を一切見ず、「動意（Momentumの前段階）」のみでEntryする。
2. Entry後、±0.03 に到達した first-touch によって方向が確定する。
3. first-touch 以降の価格推移と、10秒足・1分足・5分足のMomentumを記録する。
4. そのデータから「どのような継続動意パターンのときに伸びるか」を統計的に抽出し、Exitロジック候補を設計する。

本PoCは Exitロジックを決めるためのデータ収集が目的であり、EA内でExitは行わない（ログのみ）。

---

# 2. スコープ

- 実売買ロジックの完成ではなく、ログ収集用EAとして実装する。
- EntryはBuy固定（方向を見ないため）。
- Exitは行わず、first-touch後のMomentumログのみを収集する。
- 評価はオフライン（Python/Excel等）で行う。

---

# 3. 実行環境

- MetaTrader 5
- MQL5
- シンボル: EURUSD
- テストモード: Tickベース
- ログ出力: CSV（Entry.csv / Touch.csv / Momentum.csv）
- CSV書き込みは OnInit で open → OnDeinit で close（中間 close 禁止）

---

# 4. パラメータ仕様

以下を input として定義する。

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

# 5. データ構造

## 5.1 EntryRecord

Entryごとに以下を保持する。

- ticket
- entry_time
- entry_price
- spread_at_entry
- touched（first-touch済みか）
- touch_direction（1=UP, -1=DOWN, 0=NONE）
- touch_time
- touch_price
- max_favor_before_touch
- max_adverse_before_touch

## 5.2 10秒足バッファ

- start_time
- open
- high
- low
- close

OnTickで秒境界を監視し、10秒ごとに新Barを生成する。

---

# 6. 全体フロー

## 6.1 OnInit

- CSVファイル（Entry.csv / Touch.csv / Momentum.csv）を open
- 10秒足バッファ初期化

## 6.2 OnTick

1. Spread チェック  
2. 時間帯フィルタ  
3. 10秒足バッファ更新  
4. Entry判定（方向を見ない）  
5. ポジション保有中なら first-touch 判定  
6. first-touch確定後は Momentum 計測（ログのみ）

## 6.3 OnDeinit

- CSVファイル close

---

# 7. Entryロジック（方向を見ない）

## 7.1 動意判定（10秒足）

以下を満たす場合に「動く準備が整っている」と判断する。

- 直近10秒の値幅 >= Param_MinRange10s
- 10秒足MACDの絶対値 >= Param_MinMACDAbs10s

## 7.2 レンジ除外（60秒）

- 直近60秒の値幅 <= Param_QuietMaxRange → Entry不可

## 7.3 Entry実行

- Buy固定でEntry
- EntryRecordを作成し、Entry.csvに記録

---

# 8. first-touch判定

## 8.1 判定条件

Buy前提で以下を計算する。

- favor = Bid - entry_price
- adverse = entry_price - Bid

以下のいずれかで first-touch 確定。

- favor >= Param_TouchDistance → UP方向
- adverse >= Param_TouchDistance → DOWN方向

タイムアウト時：

- touch_direction = 0（NONE）

## 8.2 Touch.csv の記録内容

- entry_id
- time_entry
- time_touch
- touch_direction
- touch_price
- time_to_touch_sec
- max_favor_before_touch
- max_adverse_before_touch
- macd10s_touch
- stoch10s_touch
- trend1m_touch
- trend5m_touch

---

# 9. 継続動意（Momentum）計測

## 9.1 観測ウィンドウ

- 60秒（1分）
- 180秒（3分）
- 300秒（5分）

## 9.2 計測内容

- max_favor_after_touch
- max_adverse_after_touch
- sum_favor_after_touch
- sum_adverse_after_touch

## 9.3 時間足との整合性

- macd10s_delta
- stoch10s_delta
- trend1m_align（same/opposite/neutral）
- trend5m_align（same/opposite/neutral）

## 9.4 Momentum.csv の記録内容

- entry_id
- touch_direction
- window_sec
- max_favor_after_touch
- max_adverse_after_touch
- sum_favor_after_touch
- sum_adverse_after_touch
- macd10s_delta
- stoch10s_delta
- trend1m_align
- trend5m_align

---

# 10. CSV スキーマ（json形式）

PoC-0011 用 csvschema.json の内容は以下。

{
  "Entry": {
    "description": "PoC-0011 Entry log schema (directionless entry)",
    "columns": {
      "entry_id": "string",
      "time_entry": "datetime",
      "price_entry": "double",
      "spread_entry": "double",
      "range10s_entry": "double",
      "macd10s_entry": "double",
      "stoch10s_entry": "double",
      "range60s_before_entry": "double",
      "session_label": "string"
    }
  },
  "Touch": {
    "description": "PoC-0011 first-touch log schema",
    "columns": {
      "entry_id": "string",
      "time_entry": "datetime",
      "time_touch": "datetime",
      "touch_direction": "int",
      "touch_price": "double",
      "time_to_touch_sec": "int",
      "max_favor_before_touch": "double",
      "max_adverse_before_touch": "double",
      "macd10s_touch": "double",
      "stoch10s_touch": "double",
      "trend1m_touch": "string",
      "trend5m_touch": "string"
    }
  },
  "Momentum": {
    "description": "PoC-0011 momentum log schema",
    "columns": {
      "entry_id": "string",
      "touch_direction": "int",
      "window_sec": "int",
      "max_favor_after_touch": "double",
      "max_adverse_after_touch": "double",
      "sum_favor_after_touch": "double",
      "sum_adverse_after_touch": "double",
      "macd10s_delta": "double",
      "stoch10s_delta": "double",
      "trend1m_align": "string",
      "trend5m_align": "string"
    }
  }
}

---

# 11. 分析ルール（json形式）

PoC-0011 用 preprocess_rules.json の内容は以下。

{
  "Entry": {
    "primary_key": "entry_id",
    "filters": {
      "valid_spread": "spread_entry <= 3",
      "valid_range10s": "range10s_entry >= 0.0001",
      "not_quiet_market": "range60s_before_entry > 0.0002"
    },
    "derived_fields": {
      "is_high_momentum": "abs(macd10s_entry) >= 0.00001 && range10s_entry >= 0.0001"
    }
  },
  "Touch": {
    "primary_key": "entry_id",
    "filters": {
      "valid_touch": "touch_direction != 0",
      "timeout_touch": "touch_direction == 0"
    },
    "derived_fields": {
      "is_fast_touch": "time_to_touch_sec <= 30",
      "is_slow_touch": "time_to_touch_sec > 30"
    }
  },
  "Momentum": {
    "primary_key": "entry_id",
    "group_by": ["window_sec", "touch_direction"],
    "metrics": {
      "avg_max_favor": "mean(max_favor_after_touch)",
      "avg_max_adverse": "mean(max_adverse_after_touch)",
      "avg_sum_favor": "mean(sum_favor_after_touch)",
      "avg_sum_adverse": "mean(sum_adverse_after_touch)"
    },
    "filters": {
      "valid_direction": "touch_direction != 0"
    },
    "derived_fields": {
      "is_strong_momentum": "macd10s_delta > 0 && stoch10s_delta > 0",
      "is_trend_aligned": "trend1m_align == 'same' || trend5m_align == 'same'"
    }
  },
  "AnalysisRules": {
    "exit_rule_A": {
      "description": "first-touch immediate exit (+0.03)",
      "logic": "profit = 0.0003"
    },
    "exit_rule_B": {
      "description": "extend profit when momentum strong",
      "logic": "if is_strong_momentum && is_trend_aligned then profit = 0.0005 else profit = 0.0003"
    },
    "exit_rule_C": {
      "description": "stop-loss when adverse move exceeds threshold",
      "logic": "if max_adverse_after_touch >= 0.0001 then profit = -0.0001"
    }
  }
}

---

# 12. 実装上の注意

- Entry条件に方向を一切入れない（Buy固定）
- first-touch方向は「事後ラベル」
- CSVは OnInit で open → OnDeinit で close（中間 close 禁止）
- ログはヘッダ付き、1行1レコード
- 10秒足は自前集計のため、秒境界処理を厳密に行う

---

# 13. 今後の拡張ポイント

- 継続動意のスコア化
- 上位足の環境認識を Entry に組み込む
- ±0.03 の最適化
- 動意判定の強化（Tick密度、ボラティリティなど）

---

以上が PoC-0011 の最新版仕様書である。
