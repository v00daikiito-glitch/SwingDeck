# 共通参照パック 1/4　なおし

## 共通言語・価格3語・共通キー

---

## 1. 価格の英語表記（統一定義）

| 日本語                                 | 英語              | 物理データの取り先                              | Catalog           | 関数                        |
| -------------------------------------- | ----------------- | ----------------------------------------------- | ----------------- | --------------------------- |
| 確定足の終値（何本前可）               | `bar_close`       | 主に `daily_bars`（計算は多く `adj_close`）     | `"bar_close"`     | なし（テーブル列を直接読む） |
| 気配・当日スナップ                     | `snapshot`        | `latest_snapshots.last_price`                   | `"snapshot"`      | なし（テーブル列を直接読む） |
| 評価用のいま値（気配優先、なければ最新終値） | `current_price` | `snapshot` → 無ければ最新の `bar_close`         | `"current_price"` | `calculate_current_price`   |


※「最新日足」「最新価格」という曖昧語は上表のどれかに置き換える。混ぜない。

---



## 2. どこでも使う共通キー


| キー                  | 意味            | 取りうる値                                       | 置く場所                                                             | paramsに入れる？                 |
| ------------------- | ------------- | ------------------------------------------- | ---------------------------------------------------------------- | --------------------------- |
| `timeframe`         | 足             | `"daily"` / `"weekly"` / `"monthly"`        | 辺の indicator、metric_json、baseline indicator、`calculate_all` のトップ | **入れない**                    |
| `lookback`          | 何本前の確定足か      | 整数（初期 0）                                    | 工房の indicator 辺。metric にある場合も外側                                  | **入れない**                    |
| `kind` | 指標の種類 | IndicatorCatalog の文字列のみ | indicator／metric の外側キー | 入れない |
| `params` | その指標の設定をまとめたオブジェクト（期間など） | オブジェクト。例: `{ "period": 25, "source": "close" }`。中に何を書くかは kind ごと（一覧はパック2） | 工房の indicator、`metric_json`、baseline の indicator、`need_` のオブジェクト | —（params 自身。timeframe / lookback は入れない） |
| `velocity_period` | MACD速度で、何本前のヒストグラムと比べるか | 整数 | `"macd_velocity"` の params 内 | 入れる（辺の `lookback` とは別） |
| `source`            | 価格列           | `"close"` / `"open"` / `"high"` / `"low"` 等 | params内（MA系など）                                                   | 入れる                         |
| `period`            | 期間・本数         | 整数                                          | params内                                                          | 入れる                         |


※ `source` を省略した場合は `"close"` で補う（`ma` / `ema` / `rsi` / `macd_line` 系で共通）。`atr` / `volume_surge` / `volume_ma` には `source` 自体が無い（出来高や高安終だけを使うため、選ぶ必要がない）。

---



## 3. 分野ごとの呼び方は残す／揃えるのは取りに行く先だけ

---

### 工房の辺（`type`）

| 呼び方 | 意味 | 取りに行く先 |
| --- | --- | --- |
| `type:"number"` + `value` | その場の固定数 | 外部参照なし。JSON内の`value`をそのまま使う |
| `type:"bar_close"` + `timeframe` + `lookback` | 確定終値 | `daily_bars`。`lookback` の初期値は0 |
| `type:"current_price"` | 評価用いま値 | Catalog `"current_price"`（`calculate_current_price`） |
| `type:"indicator"` | 指標 | Catalog → 列を読む or `calculate_*` |
| `type:"baseline"` + `role` | 役割つき線 | `symbol_baselines`（roleごとに基準値1個） |
| `type:"drawing"` + `drawing_type` + `id` | 役割なし描画 | `chart_drawings` で `drawing_type` と `id` を指定して1件特定し、種類に応じた列を返す（いまは `drawing_type:"horizontal_line"` のみ対応 → `price` を返す） |

### 基準ラインのスロット（`type`）

| 呼び方 | 意味 | 取りに行く先 |
| --- | --- | --- |
| `type:"fixed_price"` + `value` | 固定価格（正本はこのスロット） | 外部参照なし。JSON内の`value`をそのまま使う（工房`number`とは名前を揃えない） |
| `type:"indicator"` | 指標 | Catalog → 列を読む or `calculate_*` |
| `type:"drawing"` + `drawing_type` + `id` | 役割なし描画 | `chart_drawings` で種類と `id` から特定し、種類に応じて数値にする（例： `drawing_type:"horizontal_line"` と `id`を指定　→ `price`を返す） |

### 描画の正本（参照先）

| 置き場 | 意味 |
| --- | --- |
| `chart_drawings.horizontal_lines[]`（`id` + `price`） | 役割なし水平線の正本。工房の `drawing`（`drawing_type:"horizontal_line"`）はここを見に来る。将来の種類は `drawings_json` 内に別配列などで足す。役割つきは `symbol_baselines` に置く（二重正本にしない） |









# 共通参照パック 2/4



## IndicatorCatalog ＋ `calculate_*` 詳細

---



## 1. IndicatorCatalog（全文）


| kind                      | 種別                 | 対応                                          | need_ キー                   |
| ------------------------- | -------------------- | --------------------------------------------- | ---------------------------- |
| `"bar_close"`             | テーブル列を直接読む | `close`（確定終値）。関数なし                 | 付けない                     |
| `"snapshot"`              | テーブル列を直接読む | `latest_snapshots.last_price`。関数なし       | 付けない                     |
| `"current_price"`         | 規則                 | `calculate_current_price`                     | 付けない                     |
| `"volume"`                | テーブル列を直接読む | `volume`。関数なし                            | 付けない                     |
| `"change_rate"`           | 加工                 | `calculate_change_rate`                       | `need_change_rate`           |
| `"high_low_bounds"`       | 加工                 | `calculate_high_low_bounds`                   | `need_high_low_bounds`       |
| `"ma"`                    | 加工                 | `calculate_ma`                                | `need_ma`                    |
| `"ema"`                   | 加工                 | `calculate_ema`                               | `need_ema`                   |
| `"rsi"`                   | 加工                 | `calculate_rsi`                               | `need_rsi`                   |
| `"deviation"`             | 加工                 | `calculate_deviation`                         | `need_deviation`             |
| `"macd_line"`             | 加工                 | `calculate_macd_line`                         | `need_macd_line`             |
| `"macd_signal"`           | 加工                 | `calculate_macd_signal`                       | `need_macd_signal`           |
| `"macd_hist"`             | 加工                 | `calculate_macd_hist`                         | `need_macd_hist`             |
| `"macd_velocity"`         | 加工                 | `calculate_macd_velocity`                     | `need_macd_velocity`         |
| `"tr"`                    | 加工                 | `calculate_tr`                                | `need_tr`                    |
| `"atr"`                   | 加工                 | `calculate_atr`                               | `need_atr`                   |
| `"volume_surge"`          | 加工                 | `calculate_volume_surge`                      | `need_volume_surge`          |
| `"volume_ma"`             | 加工                 | `calculate_volume_ma`                         | `need_volume_ma`             |
| `"bollinger_bands"`       | 加工                 | `calculate_bollinger_bands`                   | `need_bollinger_bands`       |
| `"bollinger_bandwidth"`   | 加工                 | `calculate_bollinger_bandwidth`               | `need_bollinger_bandwidth`   |


ルール:

- JSON の `kind` は上表のいずれかのみ。勝手なキー禁止
- need_ ＝ `"need_"` ＋ kind を1文字も変えず連結
- Catalog と「列読取／`calculate_*`」は両方使う（どちらか一方だけ、ではない）

禁止例: Catalog に `"SMA"` という kind を作らない。単純移動平均の kind は `"ma"`。
ボリンジャーの params の `ma_type` も `"ma"` / `"ema"` とし、Catalog と同じ辞書で関数を引く。

---



## 2. 関数一覧（引数・params・need_）



### 2-0. 全体呼び出し関数


| 関数 | 引数 | 役割 |
| --- | --- | --- |
| `calculate_all` | `(生データ, config_json)` | `config_json` の指示に従い、必要な加工結果をまとめて返す |


`config_json` 共通:


| キー | 形 | 意味 |
| --- | --- | --- |
| `timeframe` | `"daily"` / `"weekly"` / `"monthly"` | 足 |
| `need_<kind>` | 下表「need_の形」 | その kind を計算する／しない、および注文内容 |


---



### 2-1. 変化率・高安・MA系・RSI・TR・ATR・出来高急増・出来高MA


| 関数 | 引数 | params（工房/metric/baselineでオブジェクトとして書くとき） | need_ の短い形 | params を省略したときの補い |
| --- | --- | --- | --- | --- |
| `calculate_change_rate` | `(価格系列, period)` | `{ "period" }` | `[7, 10]`＝period列挙 | `period` は省略しない |
| `calculate_high_low_bounds` | `(価格系列)` | `{ "bound": "52WH"|"52WL"|"ATH"|"ATL" }` | `["52WH","52WL","ATH","ATL"]` | `bound` は省略しない。工房・metric では1つずつ |
| `calculate_ma` | `(価格系列, period, source)` | `{ "period", "source" }` | `[5,25,75,200]` | `period` は省略しない（`source` の省略時ルールはパック1 §2） |
| `calculate_ema` | `(価格系列, period, source)` | `{ "period", "source" }` | `[9,21]` | `period` は省略しない（`source` の省略時ルールはパック1 §2） |
| `calculate_rsi` | `(価格系列, period, source)` | `{ "period", "source" }` | `[14,7]` | `period` は省略しない（`source` の省略時ルールはパック1 §2） |
| `calculate_tr` | `(価格系列)` | params は無し | `true` | 設定できるものが無い（そのまま計算する） |
| `calculate_atr` | `(価格系列, period)` | `{ "period" }` | `[14,20]` | `period` は省略しない |
| `calculate_volume_surge` | `(出来高系列, period)` | `{ "period" }` | `[20,10]` | `period` は省略しない |
| `calculate_volume_ma` | `(出来高系列, period)` | `{ "period" }` | `[20,50]` | `period` は省略しない |


※「価格系列」は確定足 OHLCV（主に `daily_bars`）。`current_price`／`snapshot` そのものではない。
※ `calculate_change_rate` は常に終値比（params に source はない）。lookback は辺の外側。
※ `calculate_tr` は、その日1本ぶんの値幅（`高値-安値`／`|高値-前日終値|`／`|安値-前日終値|`の中で一番大きいもの）を返す。平滑化しない生の値なので、期間などの設定は無い。`calculate_atr` と同じく、調整後の高安終（`adj_high` / `adj_low` / `adj_close`）を使う
※ `calculate_atr` は内部で高安終を使う（params に source はない）。調整後の高安終（`adj_high` / `adj_low` / `adj_close`）を使う。株式分割時に値が跳ねないようにするため
※ `calculate_volume_ma` は出来高列だけを使う（`source` は無い）。

---



### 2-2. Bollinger


| 関数                              | 引数                                            | params                                                | need_               |
| ------------------------------- | --------------------------------------------- | ----------------------------------------------------- | ------------------- |
| `calculate_bollinger_bands`     | `(価格系列, period, multiplier, ma_type, source)` | `{ period, multiplier, ma_type, source, band }`       | オブジェクト1つ（または複数なら配列） |
| `calculate_bollinger_bandwidth` | `(価格系列, period, multiplier, ma_type, source)` | `{ period, multiplier, ma_type, source }`（**bandなし**） | オブジェクト1つ（または複数なら配列） |


`band`（関数の引数にはしない。params に書く）: `"both"` / `"upper"` / `"lower"`

- チャートで上下両方出したいとき: 通常 `"both"`
- 工房や baseline で線を1本だけ使うとき: `"upper"` または `"lower"`

`ma_type`: `"ma"` / `"ema"`
- 中心線を単純移動平均にするか、指数移動平均にするかの指定
- 値は Catalog の kind 名と同じ（`"ma"` → `calculate_ma`、`"ema"` → `calculate_ema`）

---




### 2-3. MACD（関数を順に渡す。関数名は略さない）

calculate_macd_line  
　→ calculate_macd_signal  
　　→ calculate_macd_hist  
　　　→ calculate_macd_velocity  

| 関数 | 関数が実際に受け取る引数 | ユーザーが注文する params | need_ |
| --- | --- | --- | --- |
| `calculate_macd_line` | `(価格系列, fast_ema_period, slow_ema_period, source)` | `{ fast_ema_period, slow_ema_period, source }` | オブジェクト1つ（または複数なら配列） |
| `calculate_macd_signal` | `(macd_line, signal_ema_period)` | `{ fast_ema_period, slow_ema_period, signal_ema_period, source }` | オブジェクト1つ（または複数なら配列） |
| `calculate_macd_hist` | `(macd_line, macd_signal)` | `{ fast_ema_period, slow_ema_period, signal_ema_period, source }` | オブジェクト1つ（または複数なら配列） |
| `calculate_macd_velocity` | `(macd_hist, smooth_ema_period, velocity_period)` | `{ fast_ema_period, slow_ema_period, signal_ema_period, smooth_ema_period, velocity_period, source }` | オブジェクト1つ（または複数なら配列） |

※ `calculate_macd_signal` / `calculate_macd_hist` / `calculate_macd_velocity` の工場引数にある途中結果（macd_line など）は、params には書かない。内側で前の関数から受け取る。

`need_macd_velocity` だけのとき: 内側で  
`calculate_macd_line` → `calculate_macd_signal` → `calculate_macd_hist` → `calculate_macd_velocity`  
の順に実行する。戻り値に途中結果も要るときだけ、それぞれの `need_macd_line` / `need_macd_signal` / `need_macd_hist` を追加する。

`velocity_period`: MACD速度で、平滑化したヒストグラムを何本前と比べるか。辺の `lookback`（評価の現在地を何本前にするか）とは別。

need の例:

    "need_macd_velocity": {
      "fast_ema_period": 12,
      "slow_ema_period": 26,
      "signal_ema_period": 9,
      "smooth_ema_period": 5,
      "velocity_period": 1,
      "source": "close"
    }



### 2-4. Deviation

関数:

`calculate_deviation(left, right)`

- 引数の `left` / `right` は数値
- 式: `(left − right) / right × 100`

params:

```json
{
  "left": { ... },
  "right": { ... }
}
```

- `left` / `right` にはオブジェクトを入れる
- そのオブジェクトは、下表のいずれかの形をとる
- どの形かは `type` で区別する。`type` ごとに必須キーが異なる
- 各オブジェクトを評価して数値にした結果が、関数引数の `left` / `right` になる

例:

```json
{
  "left": { "type": "current_price" },
  "right": {
    "type": "indicator",
    "kind": "ma",
    "timeframe": "daily",
    "params": { "period": 25, "source": "close" },
    "lookback": 0
  }
}
```

`need_deviation`: 上と同じ形のオブジェクトの配列。

---

オブジェクトの形（`type` で区別）:

| type | 必須キー | 備考 |
| --- | --- | --- |
| `"number"` | `type`, `value` | 固定の数値 |
| `"bar_close"` | `type`, `timeframe`, `lookback` | 確定終値。`lookback` の初期値は 0 |
| `"current_price"` | `type` | 評価用いま値。気配があればそれ、なければ最新の日足終値。`timeframe` は付けない |
| `"indicator"` | `type`, `kind`, `timeframe`, `params`, `lookback` | 工房の indicator 辺と同じ |
| `"baseline"` | `type`, `role` | 工房の baseline 辺と同じ。`role` は `entry` / `take_profit` / `stop_loss`。値は `symbol_baselines` の該当role1件 |
| `"drawing"` | `type`, `drawing_type`, `id` | 工房の drawing 辺と同じ |

※ `snapshot` はここ（ユーザーが選ぶ `type`）には置かない。`current_price` の内側で見に行く。






### 2-5. current_price

関数:

`calculate_current_price(銘柄)`

- Catalog kind: `"current_price"`（`need_` は付けない）
- 手順: `snapshot`（`latest_snapshots.last_price`）があればそれ。なければ最新の日足 `bar_close`
- `timeframe` / `lookback` / `params` は付けない（`params` は空 `{}`）
- ユーザーが指定するのは `"current_price"`。`"snapshot"` はユーザー向けの選択肢にしない（この関数の中で読む）

---



## 3. kind → params キー早見（オブジェクト形）


| kind | params 必須キー |
| --- | --- |
| `bar_close` / `volume` | なし（`{}`） |
| `snapshot` / `current_price` | なし（`{}`） |
| `change_rate` | `period` |
| `high_low_bounds` | `bound` |
| `ma` / `ema` / `rsi` | `period`, `source` |
| `atr` / `volume_surge` / `volume_ma` | `period` |
| `bollinger_bands` | `period`, `multiplier`, `ma_type`, `source`, `band` |
| `bollinger_bandwidth` | `period`, `multiplier`, `ma_type`, `source` |
| `macd_line` | `fast_ema_period`, `slow_ema_period`, `source` |
| `macd_signal` / `macd_hist` | `fast_ema_period`, `slow_ema_period`, `signal_ema_period`, `source` |
| `macd_velocity` | 上＋ `smooth_ema_period`, `velocity_period` |
| `deviation` | `left`, `right`（中身の形は 2-4） |

`timeframe` / `lookback` は params に入れない。付けるときは外側。`snapshot` / `current_price` には付けない。

---




## 4. `need_` の注文内容

`config_json` では、次のように書く。

```json
"need_ma": [5, 25]
```

- 左（`need_ma`）… どの kind を計算するか
- 右（`[5, 25]`）… その kind の**注文内容**（2-0 の「注文内容」と同じ）

`params` とは別物。`params` は工房の indicator などの**中のキー**。ここでは `need_<kind>` というキーの**値**として書く。オブジェクト形で書くとき、中身の形（`period`, `source` など）は params と同じ。

| kind | 注文内容 | 省略形 |
| --- | --- | --- |
| `ma`, `ema`, `rsi`, `atr`, `change_rate`, `volume_surge`, `volume_ma` | オブジェクト（複数なら配列） | 整数の列 |
| `high_low_bounds` | オブジェクト | 文字列の列 |
| `macd_line`, `macd_signal`, `macd_hist`, `macd_velocity` | オブジェクト（複数なら配列） | なし |
| `bollinger_bands`, `bollinger_bandwidth` | オブジェクト（複数なら配列） | なし |
| `deviation` | オブジェクト（複数なら配列） | なし |

省略形があるのは上2行だけ。

---

### 省略形がある kind（`ma` など）

同じ注文を、次のどちらでも書ける。

オブジェクトで全部書くとき:

```json
"need_ma": [
  { "period": 5, "source": "close" },
  { "period": 25, "source": "close" }
]
```

整数の列だけ書くとき（省略形）:

```json
"need_ma": [5, 25]
```

省略形の意味:

- `{ "period": 5, "source": "close" }` と `{ "period": 25, "source": "close" }` を、period だけにした形
- `source` を書いていないので `"close"` とみなす
- `source` を `"high"` などにしたいときは、省略形は使わずオブジェクトで書く

---

### 省略形がある kind（`high_low_bounds`）

文字列の列（省略形）:

```json
"need_high_low_bounds": ["52WH", "52WL"]
```

オブジェクト:

```json
"need_high_low_bounds": { "bound": "52WH" }
```

工房や metric で1つだけ使うときは、オブジェクト `{ "bound": "52WH" }` で書く（2-1 どおり）。

---

### 省略形がない kind（MACD・Bollinger・Deviation）

オブジェクト（またはオブジェクトの配列）だけで書く。

```json
"need_macd_velocity": {
  "fast_ema_period": 12,
  "slow_ema_period": 26,
  "signal_ema_period": 9,
  "smooth_ema_period": 5,
  "velocity_period": 1,
  "source": "close"
}
```

---





## 5. 内部呼び出し（関数→関数）

各 `calculate_*` が、内部でどの関数を呼び、何を受け取り、何を返すかの早見。

### MACD 系

| kind | 関数 | 受け取るもの | 内部で何をするか | 返すもの |
| --- | --- | --- | --- | --- |
| `macd_line` | `calculate_macd_line` | 価格系列、`fast_ema_period`、`slow_ema_period`、`source` | `calculate_ema` を2回呼ぶ（fast 用と slow 用）。2本の EMA の差を出す | `macd_line` |
| `macd_signal` | `calculate_macd_signal` | `macd_line`、`signal_ema_period` | `macd_line` に対して `calculate_ema` を1回呼び、`signal_ema_period` を基に平滑化する（signal 用） | `macd_signal` |
| `macd_hist` | `calculate_macd_hist` | `macd_line`、`macd_signal` | 他の `calculate_*` は呼ばない。`macd_line` − `macd_signal` を計算する | `macd_hist` |
| `macd_velocity` | `calculate_macd_velocity` | `macd_hist`、`smooth_ema_period`、`velocity_period` | `macd_hist` に対して `smooth_ema_period` を基に `calculate_ema` で平滑化する。平滑化後のいまの値と `velocity_period` 本前の値の差を、`velocity_period` で割って傾きを出す | `macd_velocity` |

`need_macd_velocity` だけ書いてあるとき: 内側で `calculate_macd_line` → `calculate_macd_signal` → `calculate_macd_hist` → `calculate_macd_velocity` の順に全部呼ぶ。途中の結果（line / signal / hist）を戻り値にも入れたいときだけ、それぞれの `need_macd_line` / `need_macd_signal` / `need_macd_hist` を追加する（2-3 どおり）。

### 全体

| 関数 | 内部で何をするか |
| --- | --- |
| `calculate_all` | `need_<kind>` に書かれた kind ごとに、Catalog（§1）の対応する `calculate_*` を呼ぶ。MACD は上表のとおり |

---



# 共通参照パック 3/4



## 全テーブル一覧 ＋ JSON分岐ツリー

正本: Master＝アプリJSON／一覧⑥〜⑧。Data Layer＝価格・銘柄系。

---



## 1. テーブル総覧


| #   | テーブル                      | 正本ファイル | 役割            | JSON欄             |
| --- | ------------------------- | ------ | ------------- | ----------------- |
| ①   | `chart_settings`          | Master | チャート表示設定      | `settings_json`   |
| ②   | `user_filters`            | Master | 工房エレメント／完成品   | `conditions_json` |
| ③   | `symbol_signals`          | Master | 水色シグナル（1銘柄1本） | なし（`filter_id`）   |
| ④   | `chart_drawings`          | Master | 役割なし描画        | `drawings_json`   |
| —   | `symbol_baselines`        | Master | 役割つき線の正本      | `baseline_json`   |
| —   | `user_metrics`            | Master | 列の計算定義（output_form: single/deviation） | `metric_json`     |
| ⑤   | `app_ui_settings`         | Master | 列・ソート・オレンジ    | `columns_json`    |
| ⑥   | `playlists`               | Master | 箱（user/special ＋ special_key）  | なし                |
| ⑦   | `playlist_symbols`        | Master | 箱の中身          | なし                |
| ⑧   | `user_watchlist`          | Master | 監視＝一括更新の対象一覧  | なし                |
| —   | `system_playlists`        | Master | デモ箱           | なし                |
| —   | `system_playlist_symbols` | Master | デモ中身          | なし                |
| —   | `symbols`                 | Data   | 銘柄マスタ         | なし                |
| —   | `symbol_last_viewed`      | Data   | 最終閲覧          | なし                |
| —   | `daily_bars`              | Data   | 確定日足          | なし                |
| —   | `latest_snapshots`        | Data   | 当日気配スナップ      | なし                |
| —   | `search_cache`            | Data   | 検索用語キャッシュ     | なし                |


IndicatorCatalog は **DBテーブルではない**（コード定数）。

---









## 2. 列定義（正規化テーブル）

各テーブルは **カラム｜型｜必須｜備考** の4列で統一する。  
主キー・UNIQUE・取りうる値・ルールはすべて **備考** に書く。

---

### Master

#### `chart_settings`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| id | INTEGER | はい | 主キー・自動連番 |
| user_id | TEXT | はい | ユーザー識別ID |
| target_type | TEXT | はい | `global`（全体設定）または `symbol`（銘柄個別） |
| symbol_id | TEXT | いいえ | 銘柄コード。`target_type` が `global` のときは NULL |
| country | TEXT | はい | `JP` または `US` |
| timeframe | TEXT | はい | `daily` / `weekly` / `monthly` |
| settings_json | TEXT | はい | 表示設定の JSON。中身は §3-A |

※ `target_type` が `symbol` の行は、`global` の設定との**差分だけ**を持つ（全部をコピーしない）。例：globalでMAの色を全部設定していても、この銘柄だけ25日線の色を変えたい場合、`symbol` の行にはその変えた部分だけが入る
※ 読み込み方（2段階マージ）：まず `target_type: global` の行を読み、その上に `target_type: symbol` の行の差分を重ねて上書きする。これで実際にチャートに使う設定が1つに決まる
※ 上のルールは `app_ui_settings.chart_settings_scope` の値によって動作が変わる：
　- `per_symbol` のとき：編集すると `symbol` の行（差分）だけを書き換える
　- `global_only` のとき：編集すると `global` の行と `symbol` の行（差分）の**両方**を書き換える。ただしチャートに表示する設定は常に `global` を使う





#### `user_filters`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| id | INTEGER | はい | 主キー・自動連番 |
| user_id | TEXT | はい | ユーザー識別ID |
| filter_type | TEXT | はい | `element`（部品）または `condition`（完成品） |
| filter_name | TEXT | はい | 表示用の名前 |
| conditions_json | TEXT | はい | 条件の JSON。中身は §3-B |

#### `symbol_signals`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| id | INTEGER | はい | 主キー・自動連番 |
| user_id | TEXT | はい | ユーザー識別ID |
| symbol_id | TEXT | はい | 銘柄コード |
| filter_id | INTEGER | はい | 適用する `user_filters.id`。UNIQUE(`user_id`, `symbol_id`) … 1銘柄1シグナル |

#### `chart_drawings`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| id | INTEGER | はい | 主キー・自動連番 |
| user_id | TEXT | はい | ユーザー識別ID |
| symbol_id | TEXT | はい | 銘柄コード |
| drawings_json | TEXT | はい | 役割なし描画の JSON。中身は §3-C |

#### `symbol_baselines`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| id | INTEGER | はい | 主キー・自動連番 |
| user_id | TEXT | はい | ユーザー識別ID |
| symbol_id | TEXT | はい | 銘柄コード |
| role | TEXT | はい | `entry` / `take_profit` / `stop_loss`。UNIQUE(`user_id`, `symbol_id`, `role`) |
| baseline_json | TEXT | はい | 基準値1個ぶんのJSON。中身は §3-D |
| updated_at | TEXT | いいえ | 更新日時（任意） |

#### `user_metrics`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| id | INTEGER | はい | 主キー・自動連番。他からは `metric_id` として参照 |
| user_id | TEXT | はい | ユーザー識別ID |
| name | TEXT | いいえ | 表示用の名前（任意） |
| metric_json | TEXT | はい | 指標定義の JSON。中身は §3-E |

#### `app_ui_settings`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| id | INTEGER | はい | 主キー・自動連番 |
| user_id | TEXT | はい | ユーザー識別ID |
| last_playlist_id | INTEGER | いいえ | 最後に開いていた `playlists.id`。NULL 可。`last_playlist_id` が空、もしくは参照先のプレイリストが存在しないときは、そのユーザーの中で一番最後に作られたプレイリストを開く |
| preset_filter_id | INTEGER | いいえ | オレンジ用 `user_filters.id`。NULL 可 |
| columns_json | TEXT | はい | 列・ソートの JSON。中身は §3-F |
| chart_settings_scope | TEXT | はい | チャート表示設定（settings_json）を、画面にどちらの形で出すかの切り替え。`per_symbol`（画面には銘柄ごとの設定を出す。初期値）または `global_only`（画面には全銘柄共通のglobal設定を出す）。どちらのときも、銘柄ごとの変更点はglobalとは別に記憶され続ける。詳しい動作は `chart_settings` の備考を参照 |

※ 画面側に、①globalの設定を直接編集する画面、②`chart_settings_scope`（`per_symbol`／`global_only`）を切り替える画面、の2つが必要



#### `playlists`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| id | INTEGER | はい | 主キー・自動連番 |
| user_id | TEXT | はい | ユーザー識別ID |
| playlist_name | TEXT | はい | リスト名。同一ユーザー内で重複不可 |
| kind | TEXT | はい | `user`（普通のリスト）または `special`（特別プレイリスト） |
| special_key | TEXT | いいえ | `kind` が `special` のときだけ使う値。`hold` / `setup` / `harvest` など。種類はDBの列ではなく、IndicatorCatalog と同じくアプリのコード側の一覧（カタログ）で持つ。`kind` が `user` のときは NULL |
| created_at | TEXT | はい | 作成日時。`last_playlist_id` が使えないときのフォールバックに使う |
| updated_at | TEXT | はい | 更新日時 |

※ `special_key` ごとに1ユーザー1つだけ。改名・削除不可（無ければ自動作成）。UNIQUE(`user_id`, `special_key`)（`kind` が `special` のときのみ適用）
※ カタログに新しい特別プレイリストを増やすときは、カタログに1行足すだけでよい（DBの構造は変えない）
※ インストール時は、カタログにある特別プレイリスト（hold / setup / harvest など）を全部作ってから、デモのコピーを作る（`last_playlist_id` が使えないときのフォールバックが、この順に依存するため）
※ 特別プレイリストへのチェックは、どの画面から行っても、対象銘柄が監視銘柄（user_watchlist）に未登録なら自動で追加する（画面による差はなくす）

#### `playlist_symbols`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| id | INTEGER | はい | 主キー・自動連番 |
| playlist_id | INTEGER | はい | 紐づく `playlists.id` |
| symbol_id | TEXT | はい | 銘柄コード（例: `7203.T` / `AAPL`）。UNIQUE(`playlist_id`, `symbol_id`) |
| is_pinned | INTEGER | はい | `1`＝ピン留め、`0`＝通常（初期値 `0`） |
| added_at | TEXT | はい | 追加日時 |

※ 手動並び替え用の列は持たない。プレイリスト削除時は中身も削除（CASCADE）

#### `user_watchlist`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| id | INTEGER | はい | 主キー・自動連番 |
| user_id | TEXT | はい | ユーザー識別ID |
| symbol_id | TEXT | はい | 銘柄コード。UNIQUE(`user_id`, `symbol_id`) |
| is_active | INTEGER | はい | `1`＝更新対象、`0`＝一時停止（初期値 `1`） |
| first_added_at | TEXT | はい | 初めて監視に入れた日時 |
| updated_at | TEXT | はい | 更新日時 |

※ 日足・気配値の一括更新対象はこのテーブル（`is_active = 1`）

#### `system_playlists`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| id | INTEGER | はい | 主キー |
| name | TEXT | はい | 表示名（例: おすすめ銘柄） |
| kind | TEXT | はい | 用途区分（例: `demo`） |
| updated_at | TEXT | はい | 更新日時 |

#### `system_playlist_symbols`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| system_playlist_id | INTEGER | はい | `system_playlists.id` |
| symbol_id | TEXT | はい | 銘柄コード |
| sort_order | INTEGER | はい | 表示順 |

※ 主キーは (`system_playlist_id`, `symbol_id`)。同じ組み合わせは2行不可

---

### Data Layer

#### `symbols`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| symbol_id | TEXT | はい | 主キー。内部銘柄コード（例: `7203.T` / `AAPL`） |
| country | TEXT | はい | `JP` / `US` |
| name | TEXT | はい | 銘柄名 |
| data_source | TEXT | はい | 取得元（`yahoo_jp` / `yahoo_us` / `eodhd` / `jquants` など） |
| exchange | TEXT | いいえ | 取引所 |
| is_active | INTEGER | はい | `1`＝有効、`0`＝上場廃止等（初期値 `1`） |
| delisted_at | TEXT | いいえ | 上場廃止を検知した日（`YYYY-MM-DD`） |
| updated_at | TEXT | いいえ | マスタ最終更新日時 |

#### `symbol_last_viewed`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| symbol_id | TEXT | はい | 主キー。`symbols.symbol_id` と対応 |
| last_viewed_at | TEXT | はい | 個別チャート等で最後に表示した日時 |

#### `daily_bars`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| symbol_id | TEXT | はい | 銘柄コード。主キーは (`symbol_id`, `trade_date`) |
| trade_date | TEXT | はい | 取引日（`YYYY-MM-DD`）。確定日足のみ |
| open | REAL | はい | 素の始値 |
| high | REAL | はい | 素の高値 |
| low | REAL | はい | 素の安値 |
| close | REAL | はい | 素の終値 |
| volume | REAL | はい | 出来高 |
| adj_open | REAL | いいえ | 調整後始値 |
| adj_high | REAL | いいえ | 調整後高値 |
| adj_low | REAL | いいえ | 調整後安値 |
| adj_close | REAL | はい | 調整後終値。**指標計算はこちらを使う** |
| source | TEXT | はい | 取得元 |
| ingested_at | TEXT | いいえ | 取り込み／更新日時 |

※ パック1の確定終値（Catalog `"bar_close"`）はこのテーブルから読む。接続の詳細は §3

#### `latest_snapshots`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| symbol_id | TEXT | はい | 銘柄コード。主キーは (`symbol_id`, `trade_date`) |
| trade_date | TEXT | はい | 当日の日付（`YYYY-MM-DD`） |
| open | REAL | いいえ | 今日の始値 |
| high | REAL | はい | 日中最高値 |
| low | REAL | はい | 日中最低値 |
| last_price | REAL | はい | 現在の価格（気配）。パック1の `snapshot` はこの列 |
| volume | REAL | はい | 当日累計出来高 |
| updated_at | TEXT | はい | 取得・更新日時（ISO8601） |
| source | TEXT | はい | 取得元 |

※ `latest_snapshots`の削除は一番最後：`daily_bars`確定→`weekly_bars`更新・`monthly_bars`更新（両方`daily_bars`から作る。順不同）→両方エラーなく完了後に、対象日の`latest_snapshots`の行を削除する（途中で失敗した場合の手がかりを残すため）

#### `weekly_bars`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| symbol_id | TEXT | はい | 銘柄コード。主キーは(`symbol_id`, `week_start_date`) |
| week_start_date | TEXT | はい | その週の月曜日（`YYYY-MM-DD`）。休場日は`daily_bars`に無いので自動的に集計対象から外れる |
| open | REAL | はい | 週内、最初に確定した日の`adj_open` |
| high | REAL | はい | 週内、確定済み日の`adj_high`のMAX |
| low | REAL | はい | 週内、確定済み日の`adj_low`のMIN |
| close | REAL | はい | 週内、最後に確定した日の`adj_close` |
| volume | REAL | はい | 週内、確定済み日の`volume`の合計 |
| is_closed | INTEGER | はい | `1`＝週が終わって確定済み。`0`＝今週・進行中（日足確定ごとに更新） |
| updated_at | TEXT | いいえ | 最終更新日時 |

#### `monthly_bars`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| symbol_id | TEXT | はい | 銘柄コード。主キーは(`symbol_id`, `month_start_date`) |
| month_start_date | TEXT | はい | その月の1日（`YYYY-MM-DD`） |
| open | REAL | はい | 月内、最初に確定した日の`adj_open` |
| high | REAL | はい | 月内、確定済み日の`adj_high`のMAX |
| low | REAL | はい | 月内、確定済み日の`adj_low`のMIN |
| close | REAL | はい | 月内、最後に確定した日の`adj_close` |
| volume | REAL | はい | 月内、確定済み日の`volume`の合計 |
| is_closed | INTEGER | はい | `1`＝月が終わって確定済み。`0`＝今月・進行中（日足確定ごとに更新） |
| updated_at | TEXT | いいえ | 最終更新日時 |

#### `daily_bars_all`

全ての`calculate_*`関数、およびチャート表示は、必ずこのテーブルを見に行く。生の`daily_bars`／`latest_snapshots`を直接参照しない。

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| symbol_id | TEXT | はい | 銘柄コード。主キーは(`symbol_id`, `trade_date`) |
| trade_date | TEXT | はい | 取引日（`YYYY-MM-DD`） |
| open | REAL | はい | 最新日足の始値 |
| high | REAL | はい | 最新日足の高値 |
| low | REAL | はい | 最新日足の安値 |
| close | REAL | はい | 最新日足の終値。ただし未確定の間は`current_price` |
| volume | REAL | はい | 最新日足の出来高 |
| updated_at | TEXT | いいえ | 最終更新日時 |

※ 中身は`daily_bars`（確定分）と`latest_snapshots`（今日ぶん、未確定の間だけ）を合成したもの。合成の具体的な手順は「価格更新サービス」の設計で別途書く
※ 今日、`daily_bars`にも`latest_snapshots`にも行が無いとき（市場が開く前など）：今日の行は作らない。直前の確定済みの日が、そのまま「最新日足」として残る（矛盾ではない）

#### `weekly_bars_all`

全ての`calculate_*`関数、およびチャート表示は、必ずこのテーブルを見に行く。生の`weekly_bars`／`latest_snapshots`を直接参照しない。

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| symbol_id | TEXT | はい | 銘柄コード。主キーは(`symbol_id`, `week_start_date`) |
| week_start_date | TEXT | はい | その週の月曜日（`YYYY-MM-DD`） |
| open | REAL | はい | 最新週足の始値 |
| high | REAL | はい | 最新週足の高値 |
| low | REAL | はい | 最新週足の安値 |
| close | REAL | はい | 最新週足の終値。ただし未確定の間は`current_price` |
| volume | REAL | はい | 最新週足の出来高 |
| updated_at | TEXT | いいえ | 最終更新日時 |

※ 中身は`weekly_bars`（確定分）と`latest_snapshots`（今週ぶん、未確定の間だけ）を合成したもの。合成の具体的な手順は「価格更新サービス」の設計で別途書く
※ 新しい週になっても、その週の確定日も`latest_snapshots`も無いとき（市場が開く前など）：前の週の行が、そのまま「最新週足」として残る（矛盾ではない）

#### `monthly_bars_all`

全ての`calculate_*`関数、およびチャート表示は、必ずこのテーブルを見に行く。生の`monthly_bars`／`latest_snapshots`を直接参照しない。

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| symbol_id | TEXT | はい | 銘柄コード。主キーは(`symbol_id`, `month_start_date`) |
| month_start_date | TEXT | はい | その月の1日（`YYYY-MM-DD`） |
| open | REAL | はい | 最新月足の始値 |
| high | REAL | はい | 最新月足の高値 |
| low | REAL | はい | 最新月足の安値 |
| close | REAL | はい | 最新月足の終値。ただし未確定の間は`current_price` |
| volume | REAL | はい | 最新月足の出来高 |
| updated_at | TEXT | いいえ | 最終更新日時 |

※ 中身は`monthly_bars`（確定分）と`latest_snapshots`（今月ぶん、未確定の間だけ）を合成したもの。合成の具体的な手順は「価格更新サービス」の設計で別途書く
※ 新しい月になっても、その月の確定日も`latest_snapshots`も無いとき（市場が開く前など）：前の月の行が、そのまま「最新月足」として残る（矛盾ではない）

#### `search_cache`

| カラム | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| id | INTEGER | はい | 主キー・自動連番 |
| keyword | TEXT | はい | 正規化済み検索語。UNIQUE(`keyword`, `symbol_id`, `match_type`) |
| symbol_id | TEXT | はい | `symbols.symbol_id` と紐づけ |
| display_name | TEXT | はい | 画面表示用の銘柄名 |
| furigana | TEXT | いいえ | ふりがな（JP銘柄。USは NULL 可） |
| match_type | TEXT | はい | `symbol` / `name` / `furigana` / `romaji` など |
| match_score | INTEGER | はい | 一致度（初期値 `100`） |
| search_count | INTEGER | はい | 検索回数（初期値 `1`） |
| last_searched_at | TEXT | いいえ | 最終検索日時 |
| created_at | TEXT | はい | 初回キャッシュ作成日時 |
| source | TEXT | はい | 取得元 Adapter |

※ ユーザーが検索窓に打った文字列そのものは保存しない


---


## 3. JSONツリー



### 3-A. `settings_json`（chart_settings）

`box_kind`は、指標の設定画面（箱）1つずつに付けた名前です。ほとんどはパック2のIndicatorCatalogの`kind`と同じ名前ですが、`macd`と`atr`だけ複数の`kind`を1つの箱にまとめます。

| box_kind | kind（パック2のIndicatorCatalog） |
| --- | --- |
| `"ma"` | `"ma"` |
| `"ema"` | `"ema"` |
| `"rsi"` | `"rsi"` |
| `"macd"` | `"macd_line"` |
| `"macd"` | `"macd_signal"` |
| `"macd"` | `"macd_hist"` |
| `"macd"` | `"macd_velocity"` |
| `"atr"` | `"atr"` |
| `"atr"` | `"tr"` |
| `"volume_ma"` | `"volume_ma"` |
| `"volume_surge"` | `"volume_surge"` |
| `"bollinger_bands"` | `"bollinger_bands"` |
| `"bollinger_bandwidth"` | `"bollinger_bandwidth"` |

**全体の構造**

```text
settings_json の一番外側:
main_chart_indicators[]   … box_kind: ma / ema / bollinger_bands が入る
sub_chart_indicators[]    … box_kind: rsi / macd / volume_ma / volume_surge / bollinger_bandwidth / atr が入る

main_chart_indicators[] と sub_chart_indicators[] には、上の対応表にあるbox_kindが最初から全部入っている（新しいbox_kindを対応表に追加すれば、そのぶん自動的に増える）。
使っていないものも is_favorite: false, visible: false のまま、初期値で入っている。

各配列の要素（共通の3つ）:
{ box_kind, is_favorite, visible }

box_kind: "ma" / "ema" の中身:
lines[]（必ず10個）
  { line_number: 1〜10, period, color, visible, legend_on }

box_kind: "rsi" の中身:
lines[]（必ず3個）
  { line_number: 1〜3, period, color, visible, legend_on }
reference_lines（必ず3つ）
  upper:  { value, visible, legend_on }
  middle: { value, visible, legend_on }
  lower:  { value, visible, legend_on }

box_kind: "volume_ma" の中身:
lines[]（必ず3個）
  { line_number: 1〜3, period, color, visible, legend_on }

box_kind: "volume_surge" の中身（単一。配列にしない）:
{ period, color, visible, legend_on }

box_kind: "bollinger_bandwidth" の中身（単一。配列にしない）:
{ period, multiplier, ma_type, source, color, visible, legend_on }

box_kind: "bollinger_bands" の中身:
period, ma_type, source   … 共通パラメータ（箱の直下）
mid: { color, visible, legend_on }
bands[]（必ず3個。σ1・σ2・σ3）
  { multiplier: 1〜3, upper_color, upper_visible, upper_legend_on, lower_color, lower_visible, lower_legend_on }

box_kind: "macd" の中身（1箱固定。4つのkindをまとめる。上の対応表参照）:
fast_ema_period, slow_ema_period, signal_ema_period, source   … 共通パラメータ（箱の直下）
macd_line:     { color, visible, legend_on }
macd_signal:   { color, visible, legend_on }
macd_hist:     { color, visible, legend_on }
macd_velocity: { color, visible, legend_on, smooth_ema_period, velocity_period }

box_kind: "atr" の中身（単一×2。TRとATR。上の対応表参照）:
tr:  { color, visible, legend_on }
atr: { period, color, visible, legend_on }
```

**実際に書くとこうなる（具体例）**

```text
{
  "main_chart_indicators": [
    {
      "box_kind": "ma",
      "is_favorite": true,
      "visible": true,
      "lines": [
        { "line_number": 1, "period": 5,   "color": "#2196F3", "visible": true,  "legend_on": true },
        { "line_number": 2, "period": 25,  "color": "#FF9800", "visible": true,  "legend_on": true },
        { "line_number": 3, "period": 75,  "color": "#4CAF50", "visible": false, "legend_on": true },
        // 4番目〜10番目も同じ形で続く（値は行ごとに違う）
        { "line_number": 10, "period": 250, "color": "#9E9E9E", "visible": false, "legend_on": true }
      ]
    },
    {
      "box_kind": "ema",
      "is_favorite": false,
      "visible": false,
      "lines": [
        { "line_number": 1, "period": 9,  "color": "#9C27B0", "visible": false, "legend_on": true },
        { "line_number": 2, "period": 21, "color": "#E91E63", "visible": false, "legend_on": true },
        { "line_number": 3, "period": 55, "color": "#00BCD4", "visible": false, "legend_on": true },
        // 4番目〜10番目も同じ形で続く（値は行ごとに違う）
        { "line_number": 10, "period": 200, "color": "#9E9E9E", "visible": false, "legend_on": true }
      ]
    },
    {
      "box_kind": "bollinger_bands",
      "is_favorite": false,
      "visible": false,
      "period": 20,
      "ma_type": "ma",
      "source": "close",
      "mid": { "color": "#607D8B", "visible": false, "legend_on": true },
      "bands": [
        { "multiplier": 1, "upper_color": "#F44336", "upper_visible": false, "upper_legend_on": true, "lower_color": "#2196F3", "lower_visible": false, "lower_legend_on": true },
        { "multiplier": 2, "upper_color": "#F44336", "upper_visible": false, "upper_legend_on": true, "lower_color": "#2196F3", "lower_visible": false, "lower_legend_on": true },
        { "multiplier": 3, "upper_color": "#F44336", "upper_visible": false, "upper_legend_on": true, "lower_color": "#2196F3", "lower_visible": false, "lower_legend_on": true }
      ]
    }
  ],
  "sub_chart_indicators": [
    {
      "box_kind": "rsi",
      "is_favorite": true,
      "visible": true,
      "lines": [
        { "line_number": 1, "period": 6,  "color": "#F44336", "visible": true,  "legend_on": true },
        { "line_number": 2, "period": 14, "color": "#3F51B5", "visible": true,  "legend_on": true },
        { "line_number": 3, "period": 24, "color": "#795548", "visible": false, "legend_on": true }
      ],
      "reference_lines": {
        "upper":  { "value": 70, "visible": true, "legend_on": true },
        "middle": { "value": 50, "visible": true, "legend_on": true },
        "lower":  { "value": 30, "visible": true, "legend_on": true }
      }
    },
    {
      "box_kind": "macd",
      "is_favorite": false,
      "visible": false,
      "fast_ema_period": 12,
      "slow_ema_period": 26,
      "signal_ema_period": 9,
      "source": "close",
      "macd_line":     { "color": "#2196F3", "visible": true,  "legend_on": true },
      "macd_signal":   { "color": "#FF9800", "visible": true,  "legend_on": true },
      "macd_hist":     { "color": "#9E9E9E", "visible": true,  "legend_on": true },
      "macd_velocity": { "color": "#4CAF50", "visible": false, "legend_on": true, "smooth_ema_period": 5, "velocity_period": 1 }
    },
    {
      "box_kind": "volume_ma",
      "is_favorite": false,
      "visible": false,
      "lines": [
        { "line_number": 1, "period": 5,  "color": "#FFC107", "visible": true,  "legend_on": true },
        { "line_number": 2, "period": 10, "color": "#FF9800", "visible": false, "legend_on": true },
        { "line_number": 3, "period": 20, "color": "#795548", "visible": false, "legend_on": true }
      ]
    },
    {
      "box_kind": "volume_surge",
      "is_favorite": false,
      "visible": false,
      "period": 14,
      "color": "#FF5722",
      "legend_on": true
    },
    {
      "box_kind": "bollinger_bandwidth",
      "is_favorite": false,
      "visible": false,
      "period": 20,
      "multiplier": 2.0,
      "ma_type": "ma",
      "source": "close",
      "color": "#009688",
      "legend_on": true
    },
    {
      "box_kind": "atr",
      "is_favorite": false,
      "visible": false,
      "tr":  { "color": "#B0BEC5", "visible": false, "legend_on": true },
      "atr": { "period": 14, "color": "#E91E63", "visible": true, "legend_on": true }
    }
  ]
}
```

※ settings_json は表示専用。baseline の期間などの値を、計算のたびにここから取り直すことはしない（baseline の正本は symbol_baselines）
※ 銘柄ごとに設定を記憶する仕組み（global と差分のマージ）は §2 を参照


### 3-B. `conditions_json`（user_filters）

conditions_json は、条件を表す木構造です。一番外側には、次の3種類のどれか1つが入ります。

```text
conditions_json （一番外側は次の3種類のどれか1つ。分岐キーは node_type）
│
├─ "group"（複数の条件をAND/ORでまとめる箱）
│   ├─ logical_operator: "AND" または "OR"
│   ├─ is_active: true または false（このグループ自体を有効にするか）
│   └─ rules[]（2つ以上）… 中身は、また group／filter_reference／inline_rule のどれかを入れられる（グループの中にグループを入れることもできる）
│       ※ rules[] の各要素は、必ず条件式（true/falseを返す形）でなければならない。「MA25」のような値だけを直接置くことはできない
│
├─ "filter_reference"（保存済みのelement（部品）を1個、そのまま呼び出す）
│   ├─ filter_id → 呼び出す先の user_filters.id（elementのみ）
│   └─ is_active: true または false
│
└─ "inline_rule"（1つの比較条件。例：「現在値 ＞ MA(25)」）
    ├─ left: <辺>（比較の左側）
    ├─ operator: "<" ／ ">" ／ "<=" ／ ">=" ／ "=" ／ "GC" ／ "DC"    … GC＝ゴールデンクロス（下から上に抜ける）／DC＝デッドクロス（上から下に抜ける）
    ├─ right: <辺>（比較の右側）
    └─ is_active: true または false

<辺>（left・rightに入れる値。分岐キーは type）は、次のいずれかです:
├─ "number"        → value（固定の数値そのもの。外部参照なし）
├─ "bar_close"     → timeframe, lookback
│                     確定終値。daily_barsから取得。lookbackの初期値は0
├─ "current_price" → （キーは type のみ）
│                     評価用いま値。Catalog "current_price"（calculate_current_price）
├─ "indicator"     → kind, timeframe, params, lookback
│                     kind … IndicatorCatalogにある文字列のどれか
│                     timeframe … "daily" ／ "weekly" ／ "monthly" のどれか
│                     params … kind固有（パック2）
│                     lookback … 整数（初期0）
├─ "baseline"      → role（"entry" ／ "take_profit" ／ "stop_loss" のどれか）
│                     （銘柄はJSONに書かない。symbol_baselinesにroleごとの基準値1個が入っている）
└─ "drawing"       → drawing_type, id
                      chart_drawings から、drawing_type と id を指定して描画を1つ特定し、種類に応じた列を返す（いまは drawing_type:"horizontal_line" のみ対応 → `price` 列を返す）。斜めの線（トレンドライン等）のように、時間軸によって値が変わる描画は、まだ仕組みが決まっていない（3-C「将来枠」参照）
                  
```

| filter_type | 取り得る node_type（一番最初のキー） |
| --- | --- |
| element（部品） | inline_rule だけ |
| condition（完成品） | group、または inline_rule、または filter_reference |

**保存できないエラーになる例:**
- leftとrightが両方 `"number"`（固定の数値どうしの比較は、常に同じ結果になり意味が無いため）
- operatorが `"GC"` か `"DC"` なのに、leftかrightのどちらかが `"number"`（クロス判定は、線と線の間でしか意味を持たないため）
- groupの `rules[]` が1つしかない（groupはAND/ORでまとめる箱なので、最低2つ必要）
- 必須の項目が抜けている

▼ conditions_json の保存データ例（完成品。上の定義に対応）
{
  "node_type": "group",
  "logical_operator": "AND",
  "is_active": true,
  "rules": [
    {
      "node_type": "filter_reference",
      "filter_id": 10,
      "is_active": true
    },
    {
      "node_type": "group",
      "logical_operator": "OR",
      "is_active": true,
      "rules": [
        {
          "node_type": "inline_rule",
          "left": {
            "type": "indicator",
            "kind": "ma",
            "timeframe": "daily",
            "params": { "period": 25 },
            "lookback": 0
          },
          "operator": ">",
          "right": {
            "type": "indicator",
            "kind": "ma",
            "timeframe": "daily",
            "params": { "period": 75 },
            "lookback": 0
          },
          "is_active": false
        },
        {
          "node_type": "inline_rule",
          "left": {
            "type": "current_price"
          },
          "operator": "<",
          "right": {
            "type": "number",
            "value": 1000
          },
          "is_active": true
        }
      ]
    }
  ]
}

▼ conditions_json の保存データ例（エレメント。比較は1本だけ）
{
  "node_type": "inline_rule",
  "left": {
    "type": "indicator",
    "kind": "rsi",
    "timeframe": "daily",
    "params": { "period": 14 },
    "lookback": 0
  },
  "operator": "<",
  "right": {
    "type": "number",
    "value": 30
  },
  "is_active": true
}

---



### 3-C. `drawings_json`（chart_drawings）

```text
drawings_json
└─ horizontal_lines[]
    └─ { id, price, color, label, line_style, line_width }
```

将来枠（置き場のみ・中身未設計）: trendline[] / fibonacci[] 等もここ。  
役割つきは入れない（正本は baselines）。

---



### 3-D. `baseline_json`（symbol_baselines）

```text
baseline_json
└─ <スロット>   … このrole（entry / take_profit / stop_loss）の基準値そのもの

<スロット>（このroleの基準値。分岐キーは type）は、次のいずれかです:
├─ "fixed_price" → value
├─ "indicator"   → kind, timeframe, params
│                   ※現状 lookback キーなし（工房と差＝L1）
│                   Bollinger_Bands 時は params に band 含む
└─ "drawing"     → drawing_type, id
```

チャートには、このスロット1個の値を1本の線として引く。  
今の値段と比べる計算（乖離率など）は §3-F（columns_json）側で行う。baseline_json自体には比較先を書かない。

---



### 3-E. `metric_json`（user_metrics）

```text
metric_json
├─ output_form "single" のとき    → { value: <辺> }
└─ output_form "deviation" のとき → { left: <辺>, right: <辺> }
```

`<辺>` はパック1・3-Bと同じ形（分岐キーは `type`。number / bar_close / current_price / indicator / baseline / drawing）。

▼ metric_json の保存データ例（output_form "single"。エントリーラインをそのまま出す）
{
  "output_form": "single",
  "value": { "type": "baseline", "role": "entry" }
}

▼ metric_json の保存データ例（output_form "deviation"。いま値とMA25の乖離率）
{
  "output_form": "deviation",
  "left": { "type": "current_price" },
  "right": {
    "type": "indicator",
    "kind": "ma",
    "timeframe": "daily",
    "params": { "period": 25, "source": "close" },
    "lookback": 0
  }
}

---



### 3-F. `columns_json`（app_ui_settings）

```text
columns_json
├─ columns[]
│   ├─ 共通: column_id, slot, metric_id, label
│   ├─ 任意: extract_chips[], active_chip_id
│   │   extract_chips[] → { chip_id, operator, value }
│   └─ overrides[]（任意。例外がある銘柄だけ。無ければ空）
│       └─ { symbol_id, args }
│           symbol_id … この銘柄だけ例外にする
│           args      … metric_id の output_form に合わせた形
│                       output_form "single"    → { value }
│                       output_form "deviation" → { left, right }
└─ sort
    ├─ column_id
    └─ order: "asc" | "desc"
```

列の中身は `metric_id` が指す `user_metrics` の定義そのもの。columns には埋め込まない。

overrides[]の args は、参照している metric_id の output_form と必ず同じ形にする（output_form 自体は変えられない。列全体の意味・ソートを壊さないため）。

overrides[]の例（entryとの乖離率の列で、ある銘柄だけMA50とMA200を直接比べたい場合）：

{
  "column_id": "col_entry_dev",
  "metric_id": 42,
  "overrides": [
    {
      "symbol_id": "8267.T",
      "args": {
        "left":  { "type": "indicator", "kind": "ma", "timeframe": "daily", "params": { "period": 50 }, "lookback": 0 },
        "right": { "type": "indicator", "kind": "ma", "timeframe": "daily", "params": { "period": 200 }, "lookback": 0 }
      }
    }
  ]
}

52WH日時（52週高値からの経過日数など、日付計算が絡む列）は、現状の `metric_json`（single/deviation）では表現できない。保留・未実装として扱う。



---



### 3-G. `calculate_all` の `config_json`（テーブル欄ではないがツリー）

```text
config_json
├─ timeframe
├─ need_change_rate: int[] または object[]
├─ need_high_low_bounds: ("52WH"|"52WL"|"ATH"|"ATL")[] または object[]
├─ need_ma / need_ema / need_rsi / need_atr / need_volume_surge / need_volume_ma: int[] または object[]
├─ need_tr: true
├─ need_deviation: object[]          … { left, right }（パック2 2-4）
├─ need_macd_line / need_macd_signal / need_macd_hist / need_macd_velocity: object（または配列）
├─ need_bollinger_bands: object（+band。または配列）
└─ need_bollinger_bandwidth: object（または配列）
```

`"bar_close"` / `"snapshot"` / `"current_price"` / `"volume"` に need_ は付けない。
`[]` は列（並べる）。`{ }`は設定の一塊 （=object）。`int` は数字。
`int[]` は数字の列（例: `[5, 25]`）。`object[]` は設定の一塊（=object）の列。形の詳細はパック2。

外側の `timeframe` は、足を自分で持たない注文（`need_ma` / `need_tr` など）にだけかかる。`need_deviation` にはかけない。左右の中に書いた `timeframe` を見る。

▼ config_json の例
{
  "timeframe": "daily",
  "need_ma": [
    { "period": 5, "source": "close" },
    { "period": 25, "source": "close" }
  ],
  "need_tr": true
}
{
  "timeframe": "monthly",
  "need_ma": [
    { "period": 5, "source": "close" },
    { "period": 25, "source": "close" }
  ],
  "need_tr": true
}
{
  "need_deviation": [
    {
      "left": { "type": "current_price" },
      "right": {
        "type": "indicator",
        "kind": "ma",
        "timeframe": "daily",
        "params": { "period": 25, "source": "close" },
        "lookback": 0
      }
    },
    {
      "left": { "type": "current_price" },
      "right": {
        "type": "indicator",
        "kind": "ma",
        "timeframe": "weekly",
        "params": { "period": 25, "source": "close" },
        "lookback": 0
      }
    }
  ]
}

---



## 4. 概念の分離（混ぜない）


| 概念     | 実体                           |
| ------ | ---------------------------- |
| プレイリスト | playlists + playlist_symbols |
| 監視     | user_watchlist               |
| Hold / Setup / Harvest | playlists.kind=`special` ＋ special_key（例: `hold`）の箱。種類ごとに1つ |
| 役割つき線  | symbol_baselines             |
| 役割なし描画 | chart_drawings               |
| 表示設定   | chart_settings（計算の正本ではない）    |


---



# 共通参照パック 4/4



## 正本／参照元・食い違い対照・索引

---



## 1. 正本と参照元


| 要素             | 正本（ここが正しい置き場）                       | 参照元（見に行くだけ）                                  | 固有名／キー                                |
| -------------- | ----------------------------------- | -------------------------------------------- | ------------------------------------- |
| 役割つき線          | `symbol_baselines` 1行（user×銘柄×role） | 辺 `type:"baseline"`（工房・metric共通）、チャートON      | role: entry / take_profit / stop_loss |
| 役割なし水平線        | `chart_drawings.horizontal_lines[]` | 描画UI。コピー操作で baselines へ書込可                   | `id` + `price`                        |
| 指標の種類          | IndicatorCatalog（DB外）               | 工房 indicator、metric、baseline indicator、need_ | `kind`                                |
| 指標インスタンス（比較なし） | `user_metrics`                      | 列（columns_json の metric_id）                 | `metric_id`                           |
| 比較つき条件         | `user_filters`                      | オレンジ、水色、完成品内の filter_reference               | `filter_id`                           |
| 列の並び・ソート       | `app_ui_settings.columns_json`      | プレイリスト画面                                     | `column_id`                           |
| チャート表示ON/期間枠   | `chart_settings.settings_json`      | 描画時の visible 抽出 → need_ 組立                   | 表示用。計算の正本ではない                         |
| 確定足            | `daily_bars`                        | Price、多くの calculate_*                        | adj_close が計算の正                       |
| 気配             | `latest_snapshots`                  | Snapshot、EvalPrice（案）                        | `last_price`                          |
| 評価用いま値         | 規則（単一テーブルではない）                      | `quote_price` スロット、案 Catalog `"EvalPrice"`   | 英語名 **EvalPrice**（案）                  |
| 監視一覧           | `user_watchlist`                    | DailyBarService 一括更新                         | is_active=1                           |
| プレイリスト箱        | `playlists` + `playlist_symbols`    | 見せ方・特別プレイリスト（Hold/Setup/Harvest）        | kind: user/special（special時はspecial_keyで種類を区別） |
| 銘柄マスタ          | `symbols`                           | 検索確定後など                                      | `symbol_id`                           |
| 検索用語           | `search_cache`                      | SearchService                                | keyword（ユーザーが打った生の文字列は保存しない）          |


---

## 5. パック1〜3 索引


| パック     | 中身                                                                  | 主に見る表                                  |
| ------- | ------------------------------------------------------------------- | -------------------------------------- |
| **1/4** | 共通言語・価格3語・共通キー・参照してよい最小要素                                           | timeframe/lookback/kind/params、呼び方と取り先 |
| **2/4** | IndicatorCatalog全文、calculate_* 引数・params・need_、MACDの順、Deviation現状/案 | Catalog表、関数表、params早見                  |
| **3/4** | 全テーブル総覧、列定義、JSONツリー、参照関係、概念分離                                       | §1総覧、§2列、§3ツリー                         |
| **4/4** | 正本／参照元、食い違い、索引                                                      | 本メッセージ                                 |


---


直した語の代表: 「片肺」「暗黙」「隠れ」「着地」「入口名」「潰す」「生読取」「ショートハンド／完全形」「バケツリレー」「昇格」「破棄方向」「従属」「旧関数」など → 普通の日本語に置換。表の項目自体は変えていない。