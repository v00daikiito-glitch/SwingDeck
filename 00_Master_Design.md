=================================================================
アプリケーション・データおよび関数設計書（完全ローカル・JSON連動型）
=================================================================



【データ取得方針】
将来的にYahoo(US)、Yahoo(JP)、EODHD(US)の3つのデータソースが混在するため、データ取得部（Data Fetcher）は拡張しやすい設計にすること。どのソースからデータを取得しても、最終的には『OHLCV（始値・高値・安値・終値・出来高）』の共通フォーマットに変換してから indicators.py（計算工場）に渡すこと。

「銘柄マスタ（symbols）と日足データ（daily_bars）の詳細なスキーマ・Adapter設計は 01_Data_Layer.md を正とする。
本Master Designでは、それらを前提としてアプリケーション側のテーブルを定義する。」



1. 扱うデータの定義
-----------------------------------------------------------------
【生データ】
・各銘柄の基本データ（銘柄名, ticker, 国識別[JP/US], ジャンル）
・各銘柄の時系列データ（日足・週足・月足の OHLCV：始値, 高値, 安値, 終値, 出来高）

【加工データ】
・変化率（period で本数を指定） [指定本数前の終値と、基準とする足の終値の比率をパーセントで表したもの（例：7本前比＝7日変化率）]
・52WH（52週高値） / 52WL（52週安値） [直近52週間における最高値および最安値の価格]
・ATH（過去最高値） / ATL（過去最安値） [上場来の過去最高値および過去最安値の価格]
・MA（単純移動平均線：SMA） [指定した期間の終値の単純平均価格を結んだ線]
・EMA（指数平滑移動平均線） [直近の価格に重きを置いて算出された移動平均線]
・RSI（相対力指数） [価格の上昇圧力と下降圧力の強さをパーセントで表したオシレーター]
・乖離率 [ターゲットとなる任意の移動平均線、水平線、ボリンジャーバンド等から現在値が何パーセント離れているかを示す数値]
・MACDライン [高速EMA期間と低速EMA期間の移動平均の差を表すトレンドライン]
・MACDシグナル線 [MACDラインを指定した期間でさらにEMA（平滑化）した追従線]
・MACDヒストグラム [MACDラインとMACDシグナル線の価格差（数値データ）]
・MACDヒストグラム速度 [平滑化されたヒストグラム自体の傾き（＝その時点のモメンタムの方向と強さ、つまり速度）を早期検出する指標（※加速度は原則非採用、将来の拡張オプション枠とする）]
・ATR（アベレージ・トゥルー・レンジ） [指定した期間の価格の変動幅（ボラティリティ）の平均値を示す指標]
・Volume Surge（出来高急増率） [本日の出来高が、指定した過去の平均出来高に対して何倍に急増しているかを示す倍率指標]
・ボリンジャーバンド（上バンド＋σ, 下バンド－σ） [移動平均線の上下に標準偏差を加減した、価格が収まりやすい価格帯のライン]
・ボリンジャーバンド幅（Bandwidth） [ボリンジャーバンドの広がり度合い（収縮状態）を表すオシレーター数値]


2. 関数構成（計算工場：indicators.py）
-----------------------------------------------------------------
📁 indicators.py (計算工場)
 ├── function calculate_change_rate(価格データ, period) ─── [指定本数前の終値との変化率を計算する部品]
 ├── function calculate_high_low_bounds(価格データ) ─ [52週高値/安値(52WH/L)、過去最高値/安値(ATH/L)を割り出す部品]
 ├── function calculate_ma(価格データ, period, source) ───── [MAを計算するだけの部品]
 ├── function calculate_ema(価格データ, period, source) ──── [EMAを計算するだけの部品]
 ├── function calculate_rsi(価格データ, period, source) ──────── [RSIを計算するだけの部品]
 ├── function calculate_deviation(価格データ, params) [params で指定した基準(MA, EMA, ボリンジャーバンド, 水平線等)からの乖離率を計算する部品。params は need_Deviation の配列の1要素のこと（={"target": "SMA", "period": 75} のようなオブジェクト）]
 ├── function calculate_macd_line(価格データ, fast_ema_period, slow_ema_period, source) [MACDライン(高速EMAと低速EMAの差)を計算する部品。※内部で calculate_ema を呼び出して使用する]
 ├── function calculate_macd_signal(macd_line, signal_ema_period) [MACDシグナル線(MACDラインのEMA)を計算する部品。※内部で calculate_ema を呼び出して使用する]
 ├── function calculate_macd_hist(macd_line, macd_signal) [MACDヒストグラム(MACDライン - シグナル線)を計算する部品]
 ├── function calculate_macd_velocity(macd_hist, smooth_ema_period, velocity_lookback) [MACDヒストグラムを短めのEMAで平滑化した後、指定本数前との差分(傾き)を取り、ノイズを抑えた「速度」を計算する部品。※内部で calculate_ema を呼び出して使用する]
 ├── function calculate_atr(価格データ, period) ──────── [ATRを計算する部品]
 ├── function calculate_volume_surge(出来高データ, period) ─ [直近の出来高が、指定期間の平均出来高の「何倍」かを計算する部品]
 ├── function calculate_bollinger_bands(価格データ, period, multiplier, ma_type, source) [ボリンジャーバンドの＋σ線と−σ線を計算する部品。引数名は kind "Bollinger_Bands" の params のうち period / multiplier / ma_type / source と一致させる。params のキー band（"both" / "upper" / "lower"）は引数ではなく、戻り値の使い方の指定。意味は config_json の例の直後の共通定義を正とする]
 ├── function calculate_bollinger_bandwidth(価格データ, period, multiplier, ma_type, source) [ボリンジャーバンド幅(収縮度合いを示す数値)を計算する部品]
 └── function calculate_all(生データ, config_json) 
     └── 役割：画面（マルチデッキ画面や個別チャート）に必要な加工データを一発で揃えてパックにして返す大ボス関数。
               引数は常に2つ（生データ, config_json）。何を計算するかは、直後の用語に従う。

※用語（この節で使う。先に定義し、以降この意味だけを使う）
・kind … IndicatorCatalog（3-D）の一覧で、矢印 ──> の左側に書く引用符付き文字列。例: "ATR", "change_rate", "MA"。一覧の全文は 3-D を正とする。
・config_json … calculate_all の第2引数になる JSON オブジェクト。
・キー … config_json のプロパティ名。例: "timeframe", "need_ATR"。
・need_ キー … キーのうち、文字列 need_ の直後に kind を1文字も変えず繋げたもの。例: kind が ATR なら need_ATR。kind が change_rate なら need_change_rate。
・IndicatorCatalog で「加工が必要」と書いてある kind:
  - 対応する need_ キーが config_json にある → その kind を計算し、calculate_all の戻り値に含める
  - 対応する need_ キーが無い → その kind は計算しない。戻り値にも入れない
  例: need_ATR がある → ATR を計算して戻す。need_RSI が無い → RSI は計算しない
・IndicatorCatalog で「生データそのもの」と書いてある kind（Price / Volume）:
  - need_ キーは付けない。生データから読む。上の「計算しない」には当たらない
・MACD_velocity … need_MACD_velocity があるとき、calculate_all は戻り値用の速度を作るため、内側で MACD_line → MACD_signal → MACD_hist → MACD_velocity を順に実行する。これは JSON で他キーを参照するのではなく、部品連携設計に従った固定手順である。途中結果を戻り値にも入れるときだけ need_MACD_line 等を別途付ける。

※各インジケーター関数の計算ロジックおよび部品連携設計
- データベースや他関数への二重記述（DRY原則違反）とバグを防ぐため、`calculate_macd_line` などの高次の関数内部では、すでに定義されている `calculate_ema` 等の低次の基礎関数を「適宜呼び出して（再利用して）」計算を行うこと。
- `calculate_macd_line` で算出したデータを `calculate_macd_signal` へ流し込み、その結果をさらに `calculate_macd_hist` に渡し、最終的に出来上がったデータを `calculate_macd_velocity` に順次引き渡す「バケツリレー方式（パイプライン設計）」を厳守することで、関数の独立性とパフォーマンスを極限まで維持する。


※config_json の書き方（need_ キーの作り方と「計算する / しない」は上記「用語」を正とする）
・Price / Volume は生データ参照のため、config_json に need_ キーを設けない。
・値の形：
  - 期間（本数）だけを渡す kind → 整数の配列（各要素が期間）。例: need_MA, need_EMA, need_RSI, need_ATR, need_change_rate, need_Volume_Surge
  - need_high_low_bounds → 文字列の配列。各要素は次のいずれかのみ: "52WH", "52WL", "ATH", "ATL"
  - パラメータが複数の kind → オブジェクト（複数セットならオブジェクトの配列）。オブジェクト内のプロパティ名と意味は、その kind の calculate_* の引数に対応させる（下の例および各関数定義を正とする）
・MACD：need_MACD_velocity があるとき、calculate_all はそのオブジェクトの数値だけを使い、内側で calculate_macd_line → calculate_macd_signal → calculate_macd_hist → calculate_macd_velocity の順に呼ぶ（部品連携設計）。config_json に line/signal/hist への参照は書かない。戻り値にライン等も入れるときだけ、それぞれの need_MACD_line / need_MACD_signal / need_MACD_hist を追加する。
・足（日/週/月）はキー timeframe で渡す（need_ キーではない。「計算する / しない」の判定対象外）。



# 例：config_json（このオブジェクトを calculate_all の第2引数に渡す）
calculate_all(
    raw_data=任天堂の生データ(OHLCV), 
    config_json={
        "timeframe": "daily",                // 「日足」か「週足」かを指定
        
        // ── 基本加工データの要求 ──
        "need_change_rate": [7, 10],         // 変化率を計算して！ 配列の各数が「何本前比」（＝period／metric_json.params.period）。例: 7本前比と10本前比
        "need_high_low_bounds": ["52WH", "52WL", "ATH", "ATL"], // 高値/安値を計算して！ 配列の各要素は "52WH" / "52WL" / "ATH" / "ATL" のいずれか


        // ── 移動平均・乖離率の要求 ──
        "need_MA": [5, 25, 75, 200],         // この4本のMAを計算して！ 配列の各数がMA期間
        "need_EMA": [9, 21],                 // この2本のEMAを計算して！ 配列の各数がEMA期間
        "need_Deviation": [ 
            {"target": "SMA", "period": 75}, 
            {"target": "EMA", "period": 21},
            {"target": "horizontal_line", "value": 150.0} 
            // ※上記は「75日SMA、21日EMA、150.0の水平線、それぞれからの現在値の乖離率を計算して！」という指示
            // ※キーは target（SMA / EMA / horizontal_line / bollinger）。chart_settings.target_type とは別
        ],


        // ── オシレーター・トレンドの要求 ──
        "need_RSI": [14, 7],                 // RSIを計算して！ 配列の各数が期間（＝period／metric_json.params.period）。例: RSI(14)とRSI(7) 
        // MACD速度だけ返す例。JSONに line/signal/hist は書かない。
        // calculate_all が下の数値を使い、内側で line→signal→hist→velocity の順に関数を呼ぶ（部品連携設計）。戻り値に入るのは速度だけ。
        "need_MACD_velocity": {"fast_ema_period": 12, "slow_ema_period": 26, "signal_ema_period": 9, "smooth_ema_period": 5, "velocity_lookback": 1, "source": "close"},

        "need_ATR": [14, 20],                // ATRを計算して！ 配列の各数が期間。例: ATR(14)とATR(20)
        "need_Volume_Surge": [20, 10],       // 出来高急増を計算して！ 配列の各数が平均期間。例: 20本と10本
        "need_Bollinger_Bands": {            // ボリンジャーバンドを計算して！
            "period": 20, 
            "multiplier": 2.0, 
            "ma_type": "SMA", 
            "source": "close",
            "band": "both"
        },
        "need_Bollinger_Bandwidth": {        // ボリンジャーバンド幅を計算して！
            "period": 20, 
            "multiplier": 2.0, 
            "ma_type": "SMA", 
            "source": "close"
        }
    }
)

※ kind が "Bollinger_Bands" のときの params（共通定義。need_Bollinger_Bands、metric_json、工房の indicator、baseline_json の indicator で同じキーを使う。次の5つをすべて書く）:
  - period / multiplier / ma_type / source … calculate_bollinger_bands の引数と同じ
  - band … 必須。値は "both"（上下両方）/ "upper"（＋σのみ）/ "lower"（−σのみ）。チャート用の need_Bollinger_Bands は "both"。工房の左辺・右辺や baseline の片側など、線を1本として使うときは "upper" または "lower"。3-F の画面項目「対象（＋σ / －σ）」は、"upper" / "lower" のときに対応する


# 例：マルチデッキ画面（複数分割チャート）
 │
 ├── ① [チャート1: 任天堂] の表示要求 ──> calculate_all(任天堂の生データ, config_json) ──> 計算結果をチャート1へ描画
 ├── ② [チャート2: ソニー] の表示要求 ──> calculate_all(ソニーの生データ, config_json) ──> 計算結果をチャート2へ描画
 ├── ③ [チャート3: トヨタ] の表示要求 ──> calculate_all(トヨタの生データ, config_json) ──> 計算結果をチャート3へ描画
 垂直に並列処理...

 ※各行の config_json は calculate_all の第2引数であり、chart_settings そのものではない。chart_settings（および画面状態）からその呼び出し用に組み立てて渡す。生データは銘柄ごと。config_json の中身は呼び出しごとに同じことも違うこともある。


3. データベース設計（サーバー・複数デバイス間同期前提）
-----------------------------------------------------------------
※すべてのテーブルは複数デバイス間でリアルタイム同期を行うため、「id（自動連番）」と「user_id（ユーザー識別キー）」を必須とする。
※柔軟な拡張とフロントエンドとの完全連動を実現するため、設定値は細かいカラムに分割せず `JSON型（TEXT）` で一括保存する。

プロジェクト全体を4つの箱に分ける
┌─────────────────────────────────────────────────────────┐
│ A. チャート表示設定（Chart Display）                      │
│    何を描くか・凡例・サブチャート ON/OFF                  │
├─────────────────────────────────────────────────────────┤
│ B. フィルター工房＆定義（Filter Workshop / Logic）        │
│    条件定義（部品と完成品の2層分離 ＆ 適用先の定義）      │
├─────────────────────────────────────────────────────────┤
│ C. トレード／ポジション線（Position / Lines）             │
│    損切り・利確・平均建値・支持線など                     │
├─────────────────────────────────────────────────────────┤
│ D. プレイリスト UI 設定（Playlist UI）                    │
│    動的列フィルター（複数列）・グローバルソート・全体抽出等│
├─────────────────────────────────────────────────────────┤
│ E. フロントエンドUI制御（Filter UI Builder）              │
│    条件作成ビルダーのプルダウン連動と表示モード切替ルール  │
└─────────────────────────────────────────────────────────┘


3-A. チャート表示設定（Chart Display）
     何を描くか・凡例・サブチャート ON/OFF 

〈MA, EMA, サブ指標〉
[仕様定義]
・MA/EMAおよびサブ指標の設定データはすべて【テーブル①】に保存されるが、適用優先順位として論理的な「全体プリセット層」と「銘柄個別オーバーライド層」の2階層構造を持つ。
  - 【全体プリセット層】(`target_type='global'`)：サイドメニューの「設定」にて設定する。国（JP/US）× 足（daily/weekly/monthly）の計6通りの組み合わせごとに管理。
  - 【銘柄個別オーバーライド層】(`target_type='symbol'`)：各銘柄の画面で編集・保存した際、その銘柄固有の設定として保存され、全体プリセットより最優先で適用される。（※日足の設定を変更した場合、日足のすべての設定が一気に保存される）
・1つの設定単位につき、SMA（1〜10）、EMA（1〜3）の計13本のインジケーター枠（入力値およびvisible状態）を保持する（将来的な本数拡張に対応できる構造とすること）。
・画面上の挙動および連動：
  1. 「国」と「足」を選択し、任意のMA枠（例: MA3）に数値を入力（例: 75）し、チェックボックスをONにした状態で保存すると、該当するチャートに「75日線MA」が表示される。
  2. チャート上の凡例（例:「MA75」）をクリックすると、表示状態（点灯 ↔ グレーアウト）が切り替わり、該当するライン単体の表示/非表示が切り替わる（legend_onの同期）。
  3. 凡例の左端にある「MA」自体をクリックした場合は、すべてのMAラインの一括表示/非表示を切り替える。
・データ同期の前提：
  - 各デバイスとサーバー間で常にリアルタイム同期を行う。
  - サーバー側DBでは、ユーザーを識別するため `user_id` 列を必須とする。


■ テーブル①：chart_settings
（全体設定、銘柄個別の上書き設定、およびサブ指標の設定をこの1つのテーブルに統合し、対象を target_type で切り分ける）

  - id (型: INTEGER) / 主キー・自動連番
  - user_id (型: TEXT) / ユーザー識別ID（デバイス同期用キー）
  - target_type (型: TEXT) / 'global'（全体設定） または 'symbol'（個別設定）
  - symbol_id (型: TEXT) / 銘柄コード（'global'の場合は NULL）
  - country (型: TEXT) / Country（固定値: 'JP' または 'US'）
  - timeframe (型: TEXT) / 足の種類（固定値: 'daily', 'weekly', 'monthly'）
  - settings_json (型: TEXT) / 設定値のJSONダンプ

▼ settings_json の保存データ例（Cursor指示用：MA1~10, EMA1~3までフルで枠を確保すること）
{
  // ── 移動平均線の設定（SMA 1〜10枠をすべて保持） ──
  "MA": [
    {"line_number": 1, "period": 5, "visible": true, "legend_on": true},   // デフォルト例: SMA1=5
    {"line_number": 2, "period": 25, "visible": true, "legend_on": true},  // デフォルト例: SMA2=25
    {"line_number": 3, "period": 75, "visible": true, "legend_on": true},  // デフォルト例: SMA3=75
    {"line_number": 4, "period": 200, "visible": false, "legend_on": false},
    {"line_number": 5, "period": 0, "visible": false, "legend_on": false}, // 未使用枠もデータとして保持
    {"line_number": 6, "period": 0, "visible": false, "legend_on": false},
    {"line_number": 7, "period": 0, "visible": false, "legend_on": false},
    {"line_number": 8, "period": 0, "visible": false, "legend_on": false},
    {"line_number": 9, "period": 0, "visible": false, "legend_on": false},
    {"line_number": 10, "period": 0, "visible": false, "legend_on": false} // 画面左側のチェックボックス状態 (visible) と、凡例のブラックアウト状態 (legend_on) を保持。初期値はすべて必ず true（点灯表示）で同期。
  ],
  // ── 移動平均線の設定（EMA 1〜3枠をすべて保持） ──
  "EMA": [
    {"line_number": 1, "period": 9, "visible": true, "legend_on": true},   // デフォルト例: EMA1=9
    {"line_number": 2, "period": 21, "visible": false, "legend_on": false},
    {"line_number": 3, "period": 0, "visible": false, "legend_on": false}
  ],
  
  // ── オシレーター系サブ指標の一括表示・期間設定 ──
  // マルチデッキ画面の右上チェックボックスと連動し、アプリ全体で一斉に表示/非表示を切り替える設定もここに統合
  "SubIndicators": {
    "VolumeMA": {"period": 20, "visible": true},                           // 出来高MAの計算期間と一括表示状態。初期値はtrue (1=ON)。
    "MACD": {"fast_ema_period": 12, "slow_ema_period": 26, "signal_ema_period": 9, "visible": true}, // 画面のMACD一式用。fast_ema_period / slow_ema_period はライン、signal_ema_period はシグナル線。visible は一括表示。初期値はtrue (1=ON)。
    "RSI": {"period": 14, "visible": true},                                // RSIの計算期間と一括表示状態。初期値はtrue (1=ON)。
    "BollingerBands": {                                                    // ボリンジャーバンドの設定
      "period": 20, 
      "multiplier": 2.0, 
      "ma_type": "SMA", 
      "source": "close", 
      "visible": false                                                     // ボリンジャーバンドは画面を占有するため初期値は非表示(false)とする
    }
  }
}


アプリ起動・描画時のデータ引き渡しロジック（連動の挙動）
-----------------------------------------------------------------

1. [サブ指標の一括判定]
   アプリはローカルの【テーブル①（chart_settings）の対象target_type='global'】から現在の設定を読み込み、MACDやRSIなどの表示状態（visible）、および各計算期間を取得。
   画面全体の各チャートに対して、一括で表示・非表示フラグおよび計算パラメーター（期間）を決定する。

2. [銘柄ごとのMA設定取得判定]
   特定の銘柄（例: 任天堂）のチャートを日足で描画する際、まず【テーブル①】を `target_type='symbol'`, `symbol_id='任天堂'`, `timeframe='daily'` でローカル検索し、個別上書きデータを優先して探す。（※ローカルの検索時に user_id の照合は不要）
   データがあればそれを100%優先。なければ【テーブル①】から `target_type='global'` として、任天堂の国（`JP`）、現在の足（`daily`）に一致するデフォルト設定データを読み込んで適用する。

3. [計算工場への発注]
上記で確定した設定の中から `visible=true`（チェックON）の期間だけを抽出し、`need_MA=[5, 25, 75, 200]` などを入れた config_json を作る。これを `calculate_all(生データ, config_json)` に渡して、必要な加工データだけ計算・描画する。

3-B. フィルター工房＆定義（Filter Workshop / Logic）
     条件定義（部品と完成品の2層分離 ＆ 適用先の定義）
-----------------------------------------------------------------
〈仕様定義：フィルター工房の根幹論理〉
本システムにおけるフィルター（条件）は、循環参照によるシステムクラッシュを完全に防ぐため、以下の「2層分離アーキテクチャ」を採用する。
  1. 【エレメント（部品）】：`RSI < 30` や `現在値 > 9EMA` など、これ以上分解できない単一のルール（比較あり）。他のエレメント／完成品を参照することはできない。
  2. 【条件（完成品）】：エレメントや直接入力したルールを `AND/OR` のカッコ付き数式で組み上げたもの。完成品の中に別の完成品を入れ込む（参照する）ことは禁止される。

〈仕様定義：2つの適用先（抽出とシグナル）〉
オレンジ／水色には、`user_filters` のエレメント（element）または完成品（condition）の id をアタッチできる。
  ① 【抽出フィルター（オレンジ）】：アプリ全体（場）にかかり、プレイリスト内の各銘柄について最新の確定足データに条件をかけ、合わない銘柄を落としてリストする用途。参照先は `app_ui_settings.preset_filter_id`。
  ② 【シグナル・ハイライト（水色）】：個別銘柄のチャートに対し、過去から未来にかけて条件合致箇所に網掛け（視覚的バックテスト）を行う用途。参照先は `symbol_signals.filter_id`。視覚的ノイズを防ぐため「1銘柄にセットできるシグナルは常に1つまで（後勝ち上書き）」とする。

〈仕様定義：ON/OFFフラグによる直感的な条件テスター〉
複雑な数式（例: `(A AND B) OR C`）を組んだ際、ユーザーが特定のルールだけを一時的に無効化し、結果の違いを確認できるようにする。確認先はオレンジ（リストの絞り込み）でも水色（チャートの網掛け）でもよい。各ルール配列内に `"is_active": true/false` のトグルを必ず持つ。計算側は `is_active=false` の要素を無視する。

■ テーブル②：user_filters
（フィルター工房で作成された「エレメント」と「条件（完成品）」のマスターデータ）

  - id (型: INTEGER) / 主キー・自動連番（他のテーブルやエレメント参照時はこのidを使用する）
  - user_id (型: TEXT) / ユーザー識別ID
  - filter_type (型: TEXT) / 'element'（部品） または 'condition'（完成品）
  - filter_name (型: TEXT) / 表示用の名前（例: "RSI<30", "A・転換確認型"）
  - conditions_json (型: TEXT) / このフィルターの中身（式）をJSONで保存する欄

  ※ここから【5】まで（および続く保存例）は、上の conditions_json の書き方である。
    ユーザーが工房で作った「比較の式」を、この欄にどう格納するかを定める。
    【1】は比較の左辺・右辺、【2】は辺が指標のときの詳細、【3】【4】は式全体の組み立て、【5】は保存時に拒否するもの、である。

  【1. 辺の指定（left / right で同じ形・左右対称）】
  conditions_json の比較1本（inline_rule）における左辺・右辺の形。
  left と right は、それぞれ次の3つのうちどれか一つとする。
  どれであるかはキー side_type で示す。値は "number" / "indicator" / "baseline" の3つ。

  ・number（固定数値）のとき（必須キー: side_type, value）
    - side_type … 常に "number"
    - value … 比べる数値

  ・indicator（指標）のとき（必須キー: side_type ＋【2】の4キー）
    - side_type … 常に "indicator"
    - 続けて【2. indicator（指標）の指定】と同じ4キー（kind / timeframe / params / lookback）

  ・baseline（基準ライン）のとき（必須キー: side_type, role）
    - side_type … 常に "baseline"
    - role … "entry" / "take_profit" / "stop_loss"（symbol_baselines の role と同じ）
    ※辺の材料として、ある銘柄を選択し、その銘柄のエントリー／利確／損切り（symbol_baselines の role）を使う、という意味である。チャート上にそれらの線を出すかどうかとは別の話（チャート側は 00_Master_Design.md の 3-C・symbol_baselines）。
    ※この辺の具体的な数値を出すときは、対象の銘柄と role で symbol_baselines を読みにいく。読んだ baseline_json の左右を、symbol_baselines 節と同じ手順で数値にする。
    ※conditions_json には銘柄コードを書かない。
    ※工房でこの辺の値を使用したフィルターを試す・作成する際、side_type に "baseline" を選ぶときは、対象の銘柄を必ず選ばせる。チャートを開いているときは、その銘柄を選択欄の初期値にしてよい（選ばなくてよい、という意味ではない）。

  【2. indicator（指標）の指定（左右で使い回す共通形）】
  【1】で side_type が "indicator" のときに使う、指標の共通形。次の4キーを1セットとする（必須）。
  - kind … 計算の種類。IndicatorCatalog の kind（一覧で矢印の左側の文字列）と一致させる（現在値も含む）
  - timeframe … 足。"daily"（日足）/ "weekly"（週足）/ "monthly"（月足）
    ※現在値でも、日足の終値と週足の終値は別なので timeframe は残す。
  - params … その kind 固有のパラメータ（期間など）。不要なら空の {} とする
  - lookback … 何本前の確定足を見るか。整数。初期値は 0

  【3. 塊の種類】
  conditions_json に保存するデータは、次のいずれか1つの塊から始まる。
  塊の種類はキー node_type で示す。値は次の3つだけ。

  - "group"
    複数の条件を AND / OR でまとめる括弧。
    同一括弧内は、子が複数あっても、つなぐ AND / OR は一種類だけとする。
  - "filter_reference"
    保存済みエレメントを、式の一部として参照する塊。
  - "inline_rule"
    画面で直書きした、比較1本分のルール。

  【4. 各塊が持つキー】

  ■ node_type が "group" のとき（必須: node_type, logical_operator, rules, is_active）
  - node_type … 常に "group"
  - logical_operator
    この括弧内の子どうしを、AND または OR のどちらでつなぐか。
    子が複数あっても、指定できる値は "AND" または "OR" のどちらか一つだけ。
  - rules
    この括弧の中に入る子の一覧（配列）。2つ以上とする。
    子は group / filter_reference / inline_rule のいずれか。
    より複雑な式は、子に group を入れて括弧を入れ子にする。
  - is_active
    true ならこの括弧を有効。false なら括弧ごと計算から外す（式からは消さない）。

  ■ node_type が "filter_reference" のとき（必須: node_type, filter_id, is_active）
  - node_type … 常に "filter_reference"
  - filter_id
    参照する user_filters の id。
    参照してよいのは filter_type が element の行だけ。完成品は参照しない。
  - is_active
    true なら有効。false なら計算から外す（式からは消さない）。

  ■ node_type が "inline_rule" のとき（必須: node_type, left, operator, right, is_active）
  - node_type … 常に "inline_rule"
  - left … 左辺。【1. 辺の指定】と同じ形。
  - operator … 比較の種類。例: "<", ">", "<=", ">=", "=", "GC", "DC"
  - right … 右辺。【1. 辺の指定】と同じ形。
  - is_active
    true なら有効。false なら計算から外す（式からは消さない）。

  【5. 保存時にエラーとするもの】
  ※ left と right がどちらも side_type "number" のとき（数値どうし）
  ※ operator が "GC" または "DC" のとき、left / right のどちらか一方でも side_type "number" のとき
  ※ group の rules が2つ未満のとき
  ※ 上記【1】【2】【4】に書いた必須キーが欠けているとき
  これらは保存せず、画面でエラーにする。実行時に黙って無視しない。

  【6. filter_type ごとの使い方】
  ※filter_type が element（部品）のとき
    conditions_json には inline_rule を1本だけ入れる。

  ※filter_type が condition（完成品）のとき
    式の本体は inline_rule である（1本だけでもよい）。
    複数本を AND / OR でつなぐときは group を使う。
    filter_reference でエレメントを式に入れてよい（必須ではない）。
    完成品の中に、別の完成品を入れてはならない。

  ※プレイリストに列を追加する操作では、user_filters に行を作らない。
    列に出す計算の中身は user_metrics に保存する。
    プレイリストの何列目に、どの user_metrics を表示するかは、app_ui_settings の columns_json に書く。
    列ごとの抽出（extract_chips）も columns_json 側であり、user_filters とは別である。

  ※フィルター工房でエレメントや完成品を作るとき、user_metrics は作らない。参照もしない。

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
            "side_type": "indicator",
            "kind": "MA",
            "timeframe": "daily",
            "params": { "period": 25 },
            "lookback": 0
          },
          "operator": ">",
          "right": {
            "side_type": "indicator",
            "kind": "MA",
            "timeframe": "daily",
            "params": { "period": 75 },
            "lookback": 0
          },
          "is_active": false
        },
        {
          "node_type": "inline_rule",
          "left": {
            "side_type": "indicator",
            "kind": "Price",
            "timeframe": "daily",
            "params": {},
            "lookback": 0
          },
          "operator": "<",
          "right": {
            "side_type": "number",
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
    "side_type": "indicator",
    "kind": "RSI",
    "timeframe": "daily",
    "params": { "period": 14 },
    "lookback": 0
  },
  "operator": "<",
  "right": {
    "side_type": "number",
    "value": 30
  },
  "is_active": true
}

■ テーブル③：symbol_signals
（銘柄ごとに個別に紐付けられる【シグナル・ハイライト（水色）】のアタッチ状態を保持する。「1銘柄につき1シグナル」の絶対ルールを適用するため、user_idとsymbol_idの組み合わせをUNIQUE制約とする）

  - id (型: INTEGER) / 主キー・自動連番
  - user_id (型: TEXT) / ユーザー識別ID
  - symbol_id (型: TEXT) / 銘柄コード（例: 'RACE'）
  - filter_id (型: INTEGER) / 適用する user_filters の id（element または condition 可）※外部キー：user_filters(id)
  ※データベース制約：(user_id, symbol_id) で UNIQUE 制約をかけ、新たなシグナルがセットされたらUPDATE（上書き）する。


3-C. トレード／ポジション線（Position / Lines）
     エントリー・損切り・利確・支持線など
-----------------------------------------------------------------
〈仕様定義〉
・エントリーライン／利確ライン／損切りライン（役割つきの基準ライン）は、テーブル symbol_baselines に保存する（同節の後述）。固定価格でも指標でもよい。プレイリスト列・チャート表示・工房の左辺／右辺・アラートは、この正本だけを見る。
・役割の無い水平線（支持線・抵抗線など）は、テーブル④ chart_drawings に保存する。将来のトレンドライン（斜め線）やフィボナッチも chart_drawings（drawings_json）に保存し、水平線と同じく symbol_baselines の役割へコピーできるようにする（切り取りではない。DBの列は増やさず JSON で持つ）。
・利確・損切りなどを chart_drawings と symbol_baselines の両方に同じ値段で持たない。役割つきは symbol_baselines のみ。
・chart_drawings 上の描画を、操作一回で symbol_baselines の役割（entry / take_profit / stop_loss）へコピーしてよい。コピーするときは、その時点の内容を baseline_json に書き込む。書き込み後は symbol_baselines が正であり、以後 chart_drawings 側の変更には合わせない。
・平均建値および資産管理に紐づく価格は扱わない（個人情報・管理コストのため）。

■ テーブル④：chart_drawings

  - id (型: INTEGER) / 主キー・自動連番
  - user_id (型: TEXT) / ユーザー識別ID（デバイス同期用キー）
  - symbol_id (型: TEXT) / 銘柄コード（例: '7203', 'AAPL'）
  - drawings_json (型: TEXT) / 描画オブジェクトのJSONダンプ（役割の無い描画用）

▼ drawings_json の保存データ例（Cursor解説用）
{
  // ── 役割の無い水平線（label は表示用の任意文字列） ──
  "horizontal_lines": [
    {"id": "line_1", "price": 145.5, "color": "gray", "label": "支持", "line_style": "dashed"},
    {"id": "line_2", "price": 160.0, "color": "gray", "label": "抵抗", "line_style": "solid"}
  ]
}


■ テーブル：symbol_baselines
（銘柄ごとの役割つき基準ライン。entry / take_profit / stop_loss の正本）

  - id (型: INTEGER) / 主キー・自動連番
  - user_id (型: TEXT) / ユーザー識別ID（同期および行の特定。同じ銘柄でもユーザーが違えば別行）
  - symbol_id (型: TEXT) / 銘柄コード（例: '7203', 'AAPL'）
  - role (型: TEXT) / 役割。"entry"（エントリーライン）/ "take_profit"（利確ライン）/ "stop_loss"（損切りライン）。必要な役割が増えたら値を足してよい
  - baseline_json (型: TEXT) / 左右2スロットの指定（下記）
  - updated_at (型: TEXT) / 任意。更新日時。同期や新旧判定用。無くてもよい

※同じ (user_id, symbol_id, role) は1行とする。
※このテーブルに output_kind は置かない。出力の種類（乖離率など）はプレイリスト列側（columns_json）が持つ。

〈baseline_json〉
常に left と right の2スロット。各スロットはキー slot_type で区別し、値は "quote_price" / "fixed_price" / "indicator" のいずれか一方。左右の割り当ては自由。
  ・quote_price … いまの評価用価格。気配を優先し、使えなければ最新終値。必須キー: slot_type
  ・fixed_price … 固定価格。必須キー: slot_type, value（工房の side_type "number" の value と同じ意味）
  ・indicator … 指標。必須キー: slot_type, kind, timeframe, params
    - kind … IndicatorCatalog の kind（一覧で矢印の左側の文字列）。対応する calculate_* で計算する
    - timeframe … "daily" / "weekly" / "monthly"
    - params … その kind 用のパラメータ（工房の indicator の params と同じ考え方）
    - kind が "Bollinger_Bands" のとき、params は config_json の例の直後に書く共通定義に従う（キー band を含む）


※期間などの数値はスロット内に書く。計算のたびに chart_settings から取り直さない。
※比較演算子（">" や "GC" など）は baseline_json に書かない。条件判定はフィルター工房側。

▼ baseline_json の例
  // 気配と固定8000円
  { "left": { "slot_type": "quote_price" }, "right": { "slot_type": "fixed_price", "value": 8000 } }

  // 気配と日足MA200
  {
    "left": { "slot_type": "quote_price" },
    "right": { "slot_type": "indicator", "kind": "MA", "timeframe": "daily", "params": { "period": 200 } }
  }

  // 日足MA50と日足MA200（指標どうし）
  {
    "left": { "slot_type": "indicator", "kind": "MA", "timeframe": "daily", "params": { "period": 50 } },
    "right": { "slot_type": "indicator", "kind": "MA", "timeframe": "daily", "params": { "period": 200 } }
  }

〈設定の決め方〉
・例: 「ソニー、日足MA200をエントリーラインに設定しますか？」と確認し、OKならその内容を baseline_json に書き込む。
・チャート上の MA 等を選ぶきっかけにしてよい。選んだ内容は baseline_json へコピーして書く（チャート上の MA 表示そのものは消えない）。確定後は baseline_json だけを見る。chart_settings の変更には合わせない。
・チャートにエントリー／利確／損切りの表示ON/OFFを置いてよい。ONのときは本テーブルの該当 role を読み、描く対象のスロット（fixed_price または indicator）に従って線を描く。



3-D. プレイリスト UI 設定（Playlist UI）
     動的列フィルター（複数列）・グローバルソート・全体抽出等
-----------------------------------------------------------------
〈仕様定義〉
・「どの指標を表示するか（列の定義・数は動的に増減可能）」および「抽出フィルター（オレンジ）」はアプリ全体（場）の共通設定とし、ユーザーの認知負荷を下げる（プレイリストを切り替えても維持される）。
・「現在のソート状態（どの指標で昇順/降順にしているか）」および「最後に開いていたプレイリスト」もアプリ全体のグローバル状態として保持する。これにより、個別チャートや管理画面を経由してマルチデッキに戻った際も、完全に直前の状態を復元できる。
・ソート基準は「スロットの位置」ではなく **column_id** で指定し、列の入れ替えによってソートが壊れるのを防ぐ。
・銘柄の手動並び替え（ドラッグ＆ドロップ）はコンセプト外のため廃止し、データソートを正とする。列の左右順（slot）の入れ替えは可。

※IndicatorCatalog（種類カタログ・コード定数。DBテーブルにしない）：
アプリが用意するメニュー／入力枠／参照先の対応。ユーザー操作では増減しない。3-F の入力枠辞書と一体。
JSON の kind は、次のいずれかと一致させる。勝手なキー名を作ってはならない。
・生データそのもの … 関数は呼ばず、OHLCV の該当列を使う
・加工が必要 … 右の calculate_* にマッピングする（本設計書の関数を漏れなく網羅）
  - "Price" ──────────────> 生データ（終値 close）。関数は呼ばない
  - "Volume" ─────────────> 生データ（出来高 volume）。関数は呼ばない
  - "change_rate" ────────> function calculate_change_rate()
  - "high_low_bounds" ────> function calculate_high_low_bounds()
  - "MA" ─────────────────> function calculate_ma()
  - "EMA" ────────────────> function calculate_ema()
  - "RSI" ────────────────> function calculate_rsi()
  - "Deviation" ──────────> function calculate_deviation()
  - "MACD_line" ──────────> function calculate_macd_line()
  - "MACD_signal" ────────> function calculate_macd_signal()
  - "MACD_hist" ──────────> function calculate_macd_hist()
  - "MACD_velocity" ──────> function calculate_macd_velocity()
  - "ATR" ────────────────> function calculate_atr()
  - "Volume_Surge" ───────> function calculate_volume_surge()
  - "Bollinger_Bands" ────> function calculate_bollinger_bands()
  - "Bollinger_Bandwidth" ─> function calculate_bollinger_bandwidth()

■ テーブル：user_metrics（指標インスタンス・比較なし）
（比較なしの計算定義。主用途はプレイリスト列の表示と、列ごとの抽出。比較は持たない）
※フィルター工房の user_filters（エレメント／完成品）とは別物。工房では作らない・参照しない。

  - id (型: INTEGER) / 主キー・自動連番。他から指すときは metric_id と呼ぶ（同じ番号）
  - user_id (型: TEXT) / ユーザー識別ID
  - name (型: TEXT) / 任意。表示用（例: "7日変化率"）。空でも可
  - metric_json (型: TEXT) / 指標インスタンスJSON

▼ metric_json の例
  { "kind": "change_rate", "timeframe": "daily", "params": { "period": 7 } }
  { "kind": "Deviation", "timeframe": "daily", "params": { "target": "SMA", "period": 50 } }

※ kind は上記 IndicatorCatalog（コード定数。DBテーブルにしない）のキーと一致させる。
※ kind が change_rate のとき、params の主なキー：
  - period … 何本前の終値と比べるか（必須。例: 7 なら「7本前比」。7日変化率の「7」）
※ kind が Deviation のとき、params の主なキー：
  - target … 乖離の基準の種類。"SMA" / "EMA" / "horizontal_line" / "bollinger"
  - period … 期間（SMA / EMA / bollinger のとき）
  - value … 固定価格（target が horizontal_line のとき）
※ 列削除時：その metric_id を参照する列・列ごとの抽出候補（extract_chips）がゼロなら user_metrics からも削除。他で参照中なら残す。

■ テーブル⑤：app_ui_settings（アプリ全体共通のUI状態）
（すべてのプレイリストで共通する列設定、抽出フィルター、最新のソート・選択状態を保持）

  - id (型: INTEGER) / 主キー・自動連番
  - user_id (型: TEXT) / ユーザー識別ID
  - last_playlist_id (型: INTEGER) / 最後に選択していた playlists の id
  - preset_filter_id (型: INTEGER) / NULL許可。場にかけているオレンジ用 user_filters.id（element または condition）。プレイリスト切替でも維持
  - columns_json (型: TEXT) / 列の定義・左右順・現在のソート状態。列は類型A（metric）または類型B（baseline）
※ 類型Aの列では、JSON のキー metric_id に user_metrics の id を入れる（外部キー。値は user_metrics.id と同じ番号）。
※ 類型Bの列では metric_id を持たない。テーブル symbol_baselines を見に行くのは、銘柄と role で特定する1行である（ローカルでは user_id の照合は不要）。output_kind は、読んだ baseline_json の left / right から得た数値を、どのような関数に渡し、プレイリストのその列・その銘柄のマスにどのような形で出すかの指定である。
▼ columns_json の保存データ例
{
  "columns": [
    // column_id … この列の不変ID（左右を入れ替えても変わらない。ソート指定に使う）
    // slot … 左から何番目か（入れ替えたら付け替える）
    // column_type … "metric"（類型A）または "baseline"（類型B）
    // label … 列見出しの表示名
    // 類型A: metric_id … user_metrics.id。計算の中身はそちら
    // 類型B: role … symbol_baselines の役割。output_kind … baseline_json の left / right から得た数値の出し方（例: "deviation"）
    {"column_id": "col_1", "slot": 1, "column_type": "metric", "label": "7日変化率", "metric_id": 101},
    {"column_id": "col_2", "slot": 2, "column_type": "metric", "label": "日足RSI(14)", "metric_id": 102},
    {"column_id": "col_3", "slot": 3, "column_type": "metric", "label": "日足25MA乖離", "metric_id": 103},
    {"column_id": "col_4", "slot": 4, "column_type": "baseline", "label": "エントリーライン乖離率", "role": "entry", "output_kind": "deviation"}
  ],
  // いまどの列で昇順/降順ソートしているか（スロット位置ではなく column_id で指定）
  "sort": {
    "column_id": "col_2",
    "order": "asc"
  }
}
※ sort.order の値は "asc"（昇順）または "desc"（降順）のみとする。
※ 類型Aで metric_id が指す user_metrics の例：
  id=101 → { "kind": "change_rate", "timeframe": "daily", "params": { "period": 7 } }
  id=102 → { "kind": "RSI", "timeframe": "daily", "params": { "period": 14 } }
  id=103 → { "kind": "Deviation", "timeframe": "daily", "params": { "target": "SMA", "period": 25 } }
※ 類型Aでは、指標の中身は columns_json に埋め込まない（metric_id で参照する）。
※ 類型Bでは、8000や期間200など銘柄ごとの中身は columns_json に書かない（symbol_baselines が持つ）。
※ output_kind は列を作るときに指定する。いま選べる値は "deviation" のみとする。"deviation" のときは、その銘柄の symbol_baselines（該当 role）の baseline_json にある left / right を数値にし、calculate_deviation に渡した戻り値を、プレイリストのその列・その銘柄のマスに出す。output_kind の値として、あとから値幅（引き算）などを追加してよい。
※ 列ごとの抽出（クリックで順に切替）の詳細は、後述の「指標・列・フィルターの関係」内【列ごとの抽出（クリックで順に切替）】を正とする。類型Bでも、出した数値に対する extract_chips は類型Aと同じ形で付けてよい。


-----------------------------------------------------------------
指標・列・フィルターの関係
-----------------------------------------------------------------

【役割の整理】

〈アプリ共通〉
・IndicatorCatalog … アプリが用意する指標の種類の一覧（変化率・RSI など）。コード上の定数。DBテーブルにはしない。

〈プレイリストの列〉
・columns_json … テーブル app_ui_settings の欄（型: TEXT）。どの列を、どの順で見せるか。いまのソート状態も含む。列は類型A（metric）または類型B（baseline）。
・テーブル user_metrics … 類型Aの列の計算定義（例: 7日変化率）。列ごとの抽出でも使う。
・テーブル symbol_baselines … 類型Bの列が見に行く銘柄ごとの基準ライン（3-C）。role は entry / take_profit / stop_loss。

〈フィルター工房〉
・エレメント（filter_type が element）… 比較まで含んだ1本の条件。工房で部品として保存・再利用する。中身は自己完結（user_metrics は作らない・参照しない）。
・完成品（filter_type が condition）… ユーザーが式を組み立てた条件。直書きが本体。必要なところだけエレメントをポン付けできる。完成品の中に完成品は入れない。

〈エレメント／完成品の使い道（工房の外）〉
・オレンジ（場の抽出フィルター）… プレイリストの銘柄を絞り込む。app_ui_settings.preset_filter_id で、エレメント（element）または完成品（condition）を1つ指定する。
・水色（銘柄チャートのシグナル網掛け）… 個別チャート上で条件に合う箇所を網掛けする。symbol_signals.filter_id で、エレメント（element）または完成品（condition）を1つ指定する。

【＋列の手順】
列には2種類ある。
  ・類型A（column_type が "metric"）… 1列について全銘柄が同じ計算内容。例: 3日変化率、全銘柄同じ日足25MA乖離
  ・類型B（column_type が "baseline"）… 役割（role）は列で共通。中身は銘柄ごとの symbol_baselines。出し方は output_kind。例: エントリーライン乖離率

共通（ここまで同じ）:
  1. ユーザーが＋を押す
  2. 候補一覧から1つ選ぶ。一覧には次の両方を出す。
     - IndicatorCatalog の kind（類型Aへ進む。例: 変化率）
     - symbol_baselines の role（類型Bへ進む。例: entry＝エントリーライン）

■ 手順2で IndicatorCatalog の kind を選んだとき（類型A）
  3. その kind に必要なパラメータ入力画面を出し、入力する（例: 7・日）
  4. その内容を user_metrics に1件保存する
  5. columns_json に列を1本追加する（column_id / slot / column_type:"metric" / label / metric_id）。metric_id には手順4の id を入れる
※初期は「既存の user_metrics 一覧から選ぶ」画面は出さない
※このとき user_filters には何も書かない

■ 手順2で role を選んだとき（類型B）
  3. 続けて出力の種類（output_kind）と表示名（label）を決める。例: output_kind が deviation、label が「エントリーライン乖離率」（role は手順2で選んだもの）
  4. columns_json に列を1本追加する（column_id / slot / column_type:"baseline" / label / role / output_kind）。user_metrics は作らない
※各銘柄について、その銘柄と role でテーブル symbol_baselines から1行読む（ローカルでは user_id の照合は不要。）。baseline_json の left / right を、3-C（symbol_baselines）に書いた手順で数値にする。output_kind に指定された内容に従い、その2つの数値を indicators.py（計算工場）内の関数に渡し、戻り値をプレイリストのその列・その銘柄のマスに出す。
  例: output_kind が "deviation" のとき、その2つの数値を calculate_deviation に渡し、戻り値をプレイリストのその列・その銘柄のマスに出す。
  
※このとき user_filters には何も書かない

【列ごとの抽出（クリックで順に切替）】
目的：
・ある列（例: 7日変化率）について、閾値つきの絞り込みを切り替える
・列の見出し付近などをクリックするたびに、候補が順番に切り替わる
  例のサイクル： なし → 3%以下 → 5%以下 → 10%以下 → 15%以下 → なし → …
・候補の閾値は、ユーザーが追加・変更して保存できる

前提：
・計算の中身は、その列の metric_id（user_metrics）を使う。候補を増やしても user_metrics は増やさない
・各候補が持つのは「どう比べるか」だけ（例: 「以下」と「3」）
・場全体のオレンジ抽出（preset_filter_id）とは別物。列に付く操作である

保存場所：
・各列の定義の中に持つ（columns_json の各列）。別テーブルは当面作らない
・キー名は次のとおり（名前は変えない）
  - extract_chips … その列の抽出候補の一覧
  - chip_id … 候補ひとつを識別する番号（文字列でも可）
  - operator … 比較の種類（例: "<=" は「以下」、「>=" は「以上」）
  - value … 比べる数値（例: 3 なら 3%）
  - active_chip_id … いま選ばれている候補の chip_id。絞り込みなしのときは null

▼ 列の保存例
{
  "column_id": "col_1",
  "slot": 1,
  "metric_id": 101,
  "extract_chips": [
    { "chip_id": "c1", "operator": "<=", "value": 3 },
    { "chip_id": "c2", "operator": "<=", "value": 5 },
    { "chip_id": "c3", "operator": "<=", "value": 10 },
    { "chip_id": "c4", "operator": "<=", "value": 15 }
  ],
  "active_chip_id": "c2"
}
※ operator と value は、その列の metric_id が計算した数値に対して使う
※ 初期の候補（3 / 5 / 10 / 15 など）は、指標の種類ごとにアプリが用意してよい。ユーザーが足した候補も同じ形で extract_chips に入れる
※ 有効な候補は、その列について同時に1つだけ。クリックするたびに次の候補へ進み、末尾の次は「なし（active_chip_id = null）」に戻る

実装の流れ：
  1. 列の値は metric_id から計算する（通常の列表示と同じ）
  2. active_chip_id があるときだけ、その候補の operator と value で銘柄を絞り込む
  3. 候補の追加・編集をしたら extract_chips を更新して保存する
  4. クリックされたら、候補リストの次へ active_chip_id を進めて保存する

やらないこと：
・候補を増やすたびに user_filters や user_metrics を新規作成する
・候補を完成品や場のオレンジ（preset_filter_id）へ自動変換する

【列を消すとき】
・columns_json からその列を消す
・その metric_id を、他の列や列ごとの抽出候補（extract_chips）が使っていなければ、user_metrics からも消す
・まだ使っていれば user_metrics は残す

【工房（エレメント／完成品）】
・完成品は、直書きで式を組むのが基本である
・エレメントは必須ではない。繰り返し使う条件だけ部品にして、完成品から参照できる
・工房でエレメントを一から作るとき、user_metrics は作らないし参照しない
・列に対して extract_chips（比較の候補）をかけて銘柄を絞り込む。工房のエレメントとは別物なので混ぜない

【計算への渡し方】
・表示中の列について、metric_id から user_metrics の metric_json を集め、calculate_all に渡す
・オレンジ／水色が付いているときは、指定されたエレメントまたは完成品の中身を読み取り、同様に計算に使う
・戻ってきた値を、列の表示・ソート・抽出・網掛けに使う

【やらない】
・列を足すたびに user_filters へ自動で行を作る
・オレンジ／水色にエレメントを付けたとき、裏で「そのエレメントだけの完成品」を自動作成して増やす




3-E. プレイリスト・監視銘柄・Hold（Playlist / Watchlist / Hold）
=================================================================

このセクションは、銘柄のグループ分け（プレイリスト）、日足・気配値の一括更新の対象名簿（監視銘柄）、保有中チェック（Hold）の3つを明確に分離して定義する。


-----------------------------------------------------------------
1. 概念の整理（重要）
-----------------------------------------------------------------

| 概念           | 役割                     | 説明                                                                 |
|----------------|--------------------------|----------------------------------------------------------------------|
| プレイリスト   | 見せ方の箱               | ユーザーが自由に作る銘柄のグループ。並び替えや分割表示の単位になる。 |
| 監視銘柄       | 自動更新の対象名簿       | 日足・気配値の一括更新対象（user_watchlist）。プレイリストとは別物。UI表示名は「監視銘柄」または「自動更新リスト」。 |
| Hold           | 保有中チェック           | 「今持っている」印。専用の Hold プレイリストに入る。 |

【絶対に守るルール】
・プレイリストに銘柄を追加したら → 監視銘柄（user_watchlist）にも自動登録する
・プレイリストに銘柄を追加しても → Hold には自動登録しない
・プレイリストから銘柄を外しても → 監視銘柄からは外さない
・プレイリストから銘柄を外しても → Hold からは外さない
・監視銘柄に追加しても → 普通のプレイリストと Hold には自動登録しない
・監視銘柄から明示削除したら → 全プレイリスト（Hold含む）からも外す
・監視銘柄から明示削除したら → 日足・気配値の一括更新の対象から外れる
・プレイリスト上の Hold チェックを付け外しできるのは、すでに監視銘柄に入っている銘柄だけとする
・〈プレイリスト〉Hold チェック ON → Holdプレイリストに入るだけ（監視・他プレイリストは変えない）
・〈個別チャート〉「Hold に登録」→ Holdプレイリストに入り、未登録なら user_watchlist にも追加する（他の普通プレイリストは変えない）
・〈プレイリスト・個別チャート共通〉Hold を外す（チェック OFF／解除）→ Holdプレイリストから外れるだけ（監視・他プレイリストは残す）
・個別チャート「監視銘柄に登録」→ user_watchlist に追加だけ（Hold には入れない）


-----------------------------------------------------------------
2. テーブル定義
-----------------------------------------------------------------

■ テーブル⑥：playlists（ユーザーのプレイリスト箱）

ユーザーが作る箱の情報を持つ。
Hold専用の箱もこのテーブルで管理する。

| カラム名       | 型      | 必須 | 説明                                              |
|----------------|---------|------|---------------------------------------------------|
| id             | INTEGER | はい | 主キー・自動連番                                  |
| user_id        | TEXT    | はい | ユーザー識別ID                                    |
| playlist_name  | TEXT    | はい | リスト名（例: 「半導体」「高配当」）              |
| kind           | TEXT    | はい | 'user'（普通のリスト）または 'hold'（保有専用）   |
| created_at     | TEXT    | はい | 作成日時                                          |
| updated_at     | TEXT    | はい | 更新日時                                          |

【制約・ルール】
・同じユーザーの中で playlist_name は重複不可
・kind は 'user' か 'hold' のみ
・1ユーザーにつき kind = 'hold' のプレイリストは必ず1つだけ（存在しない場合は自動作成する）
・kind = 'hold' のプレイリストはユーザーが名前変更・削除できない（システム予約）


■ テーブル⑦：playlist_symbols（プレイリストの中身）

箱の中にどの銘柄が入っているかを1行1銘柄で持つ（正規化）。

| カラム名    | 型      | 必須 | 説明                                              |
|-------------|---------|------|---------------------------------------------------|
| id          | INTEGER | はい | 主キー・自動連番                                  |
| playlist_id | INTEGER | はい | 紐づく playlists の id                            |
| symbol_id   | TEXT    | はい | 銘柄コード（内部形式: 7203.T / AAPL）             |
| is_pinned   | INTEGER | はい | 1 = ピン留め（常に上部表示）、0 = 通常（初期値0） |
| added_at    | TEXT    | はい | 追加日時                                          |

【主キー】
・(playlist_id, symbol_id) をユニークにする

【注意】
・手動の並び替え（ドラッグ＆ドロップ）は行わない方針のため、並び順用のカラムは持たない
・プレイリストを削除した場合、中身の行も一緒に削除する（CASCADE）


■ テーブル⑧：user_watchlist（監視銘柄）

このテーブルが日足・気配値の一括更新の対象を決める名簿になる。

| カラム名       | 型      | 必須 | 説明                                              |
|----------------|---------|------|---------------------------------------------------|
| id             | INTEGER | はい | 主キー・自動連番                                  |
| user_id        | TEXT    | はい | ユーザー識別ID                                    |
| symbol_id      | TEXT    | はい | 銘柄コード                                        |
| is_active      | INTEGER | はい | 1 = 更新対象、0 = 一時停止（初期値1）             |
| first_added_at | TEXT    | はい | 初めて監視に入れた日時                            |
| updated_at     | TEXT    | はい | 更新日時                                          |

【主キー】
・(user_id, symbol_id) をユニークにする

【重要な役割】
・DailyBarService の update_*_for_watchlist（ユーザ操作起点）は、このテーブルで is_active = 1 の銘柄だけを対象にする
・プレイリストに入っているかどうかは見ない


■ システム用デモプレイリスト（お試し期間用）

お試し期間中に最初からチャートを見せるための公式リスト。

system_playlists

| カラム名   | 型      | 必須 | 説明                          |
|------------|---------|------|-------------------------------|
| id         | INTEGER | はい | 主キー                        |
| name       | TEXT    | はい | 表示名（例: 「おすすめ銘柄」）|
| kind       | TEXT    | はい | 用途区分（例: demo）          |
| updated_at | TEXT    | はい | 更新日時                      |

system_playlist_symbols

| カラム名           | 型      | 必須 | 説明                    |
|--------------------|---------|------|-------------------------|
| system_playlist_id | INTEGER | はい | system_playlists の id  |
| symbol_id          | TEXT    | はい | 銘柄コード              |
| sort_order         | INTEGER | はい | 表示順                  |


-----------------------------------------------------------------
3. 操作と副作用のルール（必須で実装すること）
-----------------------------------------------------------------

| ユーザー操作 | playlist_symbols（普通） | user_watchlist | Holdプレイリスト |
|--------------|--------------------------|----------------|------------------|
| 普通のプレイリストに銘柄を追加 | その箱に追加 | 未登録なら登録 | 変化なし |
| 普通のプレイリストから銘柄を外す | その箱から削除 | 残す | 変化なし |
| 監視銘柄に追加（一覧／個別チャートの「監視に登録」） | 変化なし | 追加 | 変化なし |
| 監視銘柄から明示削除 | 全プレイリストから削除 | 削除 | 外れる |
| Holdチェック ON（監視済み銘柄） | 変化なし | 残す | 入る |
| Holdチェック OFF | 変化なし | 残す | 外れる |
| 個別チャート「Hold に登録」 | 変化なし | 未登録なら登録 | 入る |


-----------------------------------------------------------------
4. お試し期間との関係
-----------------------------------------------------------------
・お試し期間中は user_watchlist の登録数に上限を設ける（例: 50銘柄）
・初期状態で system_playlists のデモ銘柄をユーザーの監視銘柄とプレイリストにコピーして渡す
・お試し期間終了後に「デモ銘柄を整理する」機能を用意できるよう、system側とuser側を分離しておく


-----------------------------------------------------------------
5. 既存設計書との関係
-----------------------------------------------------------------
・00_Master_Design.md の⑥playlists・⑦playlist_symbols をこの内容で置き換える・拡張する
・01_Data_Layer.md の「登録した銘柄だけ日足をキャッシュする」方針は、この user_watchlist を正とする
・アラート機能（0723.md）は、基本的に user_watchlist に入っている銘柄を対象にすることを推奨する

=================================================================
以上
=================================================================


3-F. フロントエンドUI制御（Filter UI Builder）
     条件作成ビルダーのプルダウン連動と表示モード切替ルール
-----------------------------------------------------------------
〈仕様定義：UIレイアウト基本構造〉
フィルター工房での条件構築UIは、プログラミング知識がなくても直感的に操作できるよう、常に「左辺(A)・比較演算子(B)・右辺(C)」の3ブロック構造で表現する。ユーザーの選択結果は、最終的に `user_filters.conditions_json` に変換されて保存される。保存形の正は 3-B（node_type / side_type / left / right）とする。
  - 左辺(A)：【2. 辺の指定】と同じ。side_type は "number"（固定数値）または "indicator"（指標）。指標の例: 現在値, MA, RSI, MACDヒストグラム速度など
  - 比較(B)：判定ルール（例: ＜, ＞, ＝, ≦, ＞＝, GC, DCなど）。JSON では operator
  - 右辺(C)：左辺と同じ【2. 辺の指定】（左右対称）。side_type は "number" または "indicator"

〈仕様定義：共通UIルール（ルックバック）〉
※左辺または右辺の side_type が "indicator" のとき、「何本前の確定足を見るか」のルックバック（本数入力：初期値0）を必ず付ける。JSON では【1. indicator の指定】の lookback。
  例（両辺とも indicator）：現在値[1] ＜ MA[1]
※side_type が "number" の辺には、ルックバック枠は出さない。
  例（左辺は indicator、右辺は number）：RSI(14)[0] ＜ 30

〈仕様定義：論理演算（AND / OR）とグループ化（カッコ）の視覚的インデントUI表現〉
  - 複数の条件ブロックを追加する際、ブロック間に「AND」または「OR」の接続プルダウンを配置し、ユーザーが論理演算を指定できるようにする。※NORやNANDは実装せず、ANDとORのみとする。
  - 複雑な数式（カッコ）を視覚的に表現するため、UI上では「グループ化（カッコの開始）」を行う。カッコ `(` が選ばれるか、新しくグループがネスト（入れ子）されるたびに、UIコンポーネントが動的に右へ一段インデント（字下げ）され、視覚的なインサイド枠で囲まれる。
  - 閉じカッコ `)` が入力される、あるいはグループが終了した瞬間に、インデントは左へ1段階戻る構造とする。これにより、どれだけ深いネスト構造になっても、数式の優先順位（塊）を直感認識させるUIとする。これが conditions_json 内の node_type が "group" のネストと1対1で連動する。

〈仕様定義：全インジケーターのUIプルダウン連動辞書（スキーマ・全リスト化）〉
左辺または右辺で選択した「指標の種類」に応じて、UI側で動的に必要なパラメータ入力枠を表示・非表示させる。フロントエンド側で以下のマッピング（辞書）を完全保持して制御する。勝手な項目を作成してはならない。
- 【現在値 (Price)】 ───────────────────> [足 (日/週/月)] ＋ [ルックバック (本数入力：初期値0)]
- 【出来高 (Volume)】 ──────────────────> [足 (日/週/月)] ＋ [ルックバック (本数入力：初期値0)]
- 【変化率 (change_rate)】 ─────────────> [足 (日/週/月)] ＋ [期間 (数値入力：何本前比か)] ＋ [ルックバック (本数入力：初期値0)]
- 【MA (単純移動平均)】 ────────────────> [足 (日/週/月)] ＋ [期間 (数値入力)] ＋ [価格ソース(Source)] ＋ [ルックバック (本数入力：初期値0)]
- 【EMA (指数平滑移動平均)】 ───────────> [足 (日/週/月)] ＋ [期間 (数値入力)] ＋ [価格ソース(Source)] ＋ [ルックバック (本数入力：初期値0)]
- 【RSI】 ──────────────────────────────> [足 (日/週/月)] ＋ [期間 (数値入力)] ＋ [価格ソース(Source)] ＋ [ルックバック (本数入力：初期値0)]
- 【MACD本体 (MACD_line)】 ─────────────> [足 (日/週/月)] ＋ [高速EMA期間(Length)] ＋ [低速EMA期間(Length)] ＋ [価格ソース(Source)] ＋ [ルックバック (初期値0)]
- 【MACDシグナル (MACD_signal)】 ─────────> [足 (日/週/月)] ＋ [高速EMA期間(Length)] ＋ [低速EMA期間(Length)] ＋ [シグナルEMA期間(Length)] ＋ [価格ソース(Source)] ＋ [ルックバック (初期値0)]
- 【MACDヒストグラム (MACD_hist)】 ───────> [足 (日/週/月)] ＋ [高速EMA期間(Length)] ＋ [低速EMA期間(Length)] ＋ [シグナルEMA期間(Length)] ＋ [価格ソース(Source)] ＋ [ルックバック (初期値0)]
- 【MACDヒストグラム速度 (MACD_velocity)】 ──> [足 (日/週/月)] ＋ [高速EMA期間(5~20)] ＋ [低速EMA期間(15~40)] ＋ [シグナルEMA期間(5~15)] ＋ [ヒストグラム平滑化EMA期間(3~9)] ＋ [速度のルックバック(1~3)] ＋ [価格ソース(Source)] ＋ [ルックバック (初期値0)]
- 【乖離率 (Deviation)】 ────────────────> [ターゲットの種類 (水平線 / MA / EMA / ボリンジャーバンド)]
     ├── (MA/EMA選択時) ──> [足 (日/週/月)] ＋ [期間 (数値入力)] ＋ [価格ソース(Source)] ＋ [ルックバック (初期値0)]
     ├── (水平線選択時) ──> [固定価格 (数値入力)] ＋ [ルックバック (初期値0)]
     └── (ボリンジャーバンド選択時) ──> [足 (日/週/月)] ＋ [期間 (Length)] ＋ [乗数 (Multiplier)] ＋ [移動平均の種類 (Basis MA Type)] ＋ [価格ソース (Source)] ＋ [対象 (＋σ / －σ)] ＋ [ルックバック (初期値0)]
- 【高値/安値 (high_low_bounds)】 ──────> [種類 (52WH / 52WL / ATH / ATL)] ＋ [ルックバック (初期値0)]
- 【ATR】 ──────────────────────────────> [足 (日/週/月)] ＋ [期間 (数値入力)] ＋ [ルックバック (初期値0)]
- 【出来高急増 (Volume_Surge)】 ─────────> [足 (日/週/月)] ＋ [期間 (数値入力)] ＋ [ルックバック (初期値0)]
- 【ボリンジャーバンド (Bollinger_Bands)】 ──> [足 (日/週/月)] ＋ [期間 (Length: 数値入力)] ＋ [乗数 (Multiplier: 数値入力)] ＋ [移動平均の種類 (Basis MA Type: SMA/EMA等)] ＋ [価格ソース (Source: close/open等)] ＋ [対象 (＋σライン / －σライン)] ＋ [ルックバック (初期値0)]
- 【ボリンジャーバンド幅 (Bollinger_Bandwidth)】 ─> [足 (日/週/月)] ＋ [期間 (Length: 数値入力)] ＋ [乗数 (Multiplier: 数値入力)] ＋ [移動平均の種類 (Basis MA Type: SMA/EMA等)] ＋ [価格ソース (Source: close/open等)] ＋ [ルックバック (初期値0)]

〈仕様定義：表示モードの切り替え（数式 ⇔ 直感）〉
データベース（ASTツリーJSON）の構造は1種類のまま変更せず、フロントエンドの表示テンプレートを切り替えることで、ユーザーの好みに合わせた言語表示を提供する。条件作成画面のトグルスイッチで即座に切り替え可能とする。

【数式記号と直感日本語のマッピング対応テーブル（絶対指示）】
  - `GC` ────> `下から上に抜ける (ゴールデンクロス)`
  - `DC` ────> `上から下に抜ける (デッドクロス)`
  - `<` ─────> `より小さい`
  - `>` ─────> `より大きい`
  - `<=` ────> `以下`
  - `>=` ────> `以上`
  - `=` ─────> `と等しい`

【4つの具体的なモード対比表示例】
  - 【数式モード】： 以下の短い記号表現で表示する。
       1. `日足SMA(5) GC 日足SMA(25)`
       2. `日足RSI(14) < 30`
       3. `日足MACD_Vel(12,26,9,5,1) > 0`
       4. `Vol_Surge(20) > 2.0`
  - 【直感（自然言語）モード】： 以下の分かりやすい日本語表現に自動で翻訳して表示する。
       1. `日足の5SMA が 日足の25SMA を 下から上に抜ける (ゴールデンクロス)`
       2. `日足のRSI(14) が 30 より 小さい`
       3. `日足のMACDヒストグラム速度（12,26,9,平滑化5,LB1） が 0 より 大きい`
       4. `出来高 が 過去20日平均の 2.0倍 より 大きい`

〈仕様定義：エレメントの組み込みとON/OFF制御〉
  - ブロックの追加時には、新規の直書きルール（inline_rule）だけでなく、事前に作成済みの「エレメント（部品）」をプルダウンから呼び出して「1つのブロック（node_type が filter_reference）」としてポン付けできるUIを提供する。
  - inline_rule / filter_reference / group の各ブロックの横には「ON/OFF」のトグルを置き、JSON の is_active を true/false に切り替える。false の要素は一時的に計算から外し、リアルタイムにテストすることができる（式からは消さない）。
  - 確認先はオレンジ（リストの絞り込み）でも水色（チャートの網掛け）でもよい（3-B の ON/OFF と同じ）。


  