=================================================================
01_Data_Layer（データ取得・正規化・端末キャッシュ）骨子
=================================================================
※本書は「生データをどう取り、どう揃え、端末にどう残すか」の設計骨子。
※設定・フィルター・UI状態のJSON設計は 00_Master_Design.md 側。ここでは時系列の株価データのみを扱う。
※プロトタイプは参考のみ。新設計の正とする。


-----------------------------------------------------------------
0. この設計で決めること / 決めないこと
-----------------------------------------------------------------
【決める】
・データ取得元（ソース）の種類と優先順位
・アプリ内部の共通データ形（正規化スキーマ）
・銘柄コードの統一ルール
・終値の扱い（調整後を正とするか）
・日足・週足・月足の保存方針
・端末への保存の考え方（何を残し、何を捨てるか）
・取得タイミングと更新の考え方（概要）
・ユーザーによるデータソース選択と、Adapterの抽象化方針
・銘柄マスタ取得用SymbolAdapterの抽象化
・銘柄検索（search_cache）の役割分担

【決めない（別設計 or 後続）】
・インジケーター計算ロジック本体 → indicators.py / 00_Master_Design
・プレイリスト・フィルターUI → 既存設計書
・アラート通知の詳細
・サーバー経由配布（商用ライセンス後）のAPI詳細


-----------------------------------------------------------------
1. 全体方針（一言）
-----------------------------------------------------------------
・各データ会社の生レスポンスは、必ず「共通の日足OHLCV形」に変換してからアプリ内部で使う。
・アプリ本体（チャート・計算工場・スクリーニング）は、共通形だけを見る。ソース固有の形は見ない。
・時系列の株価はJSON一括保存しない。正規化した行データとして端末にキャッシュする。
・設定系（MA表示、フィルター等）は従来どおりJSON。株価データとは保存形式を分ける。
・具体的なAdapter名（YahooJpAdapterなど）をアプリ全体に散らばせない。抽象化された JP_Adapter / US_Adapter を通して呼び出す。
・銘柄マスタ（シンボル一覧や属性情報、検索用キーワードなど）の取得・検索処理についても、同様に抽象化する。直接データソースごとのAPIやテーブルへアクセスする実装は必ず避ける。  
・必ず「SymbolAdapter（抽象インターフェース）」を介し、データ取得・正規化・保存・検索までを一手に担わせる。
・例えば JP_Adapter/US_Adapter には get_symbol_master(), search_symbols(keyword), update_master_cache() 等の共通メソッドを用意し、どの国/市場であってもアプリ本体側の呼び出し方は統一する（分岐・条件分けはAdapter側のみで吸収）。
・取得したシンボル、マスタ情報、検索キーワードは全て “2. 共通データ正規化規則” で定義した関数を通して加工・保存すること。ユーザー入力、生データ、そのまま保存は絶対にしない。



-----------------------------------------------------------------
2. 共通データ正規化規則（normalize）
-----------------------------------------------------------------
検索・マスタ登録・キャッシュ保存など、アプリ内で文字列を扱う際の正規化ルールをここに定義する。  
このルールは特定のAdapterやテーブルに依存せず、アプリ全体で共通に使用する。

【基本方針】
・正規化は「検索の一貫性」と「重複排除」のために必須である
・各Adapter内で独自に正規化処理を書かない
・必ず共通の正規化関数を通してから、search_cache や symbols に保存する

【正規化ルール】

| 種別        | 正規化内容                                           | 例                                                                 |
|-------------|------------------------------------------------------|--------------------------------------------------------------------|
| symbol      | 全角→半角・大文字化・前後空白除去・不要な接尾辞除去  | `" 7203.t "` → `"7203.T"` <br> `"aapl.us"` → `"AAPL"`             |
| name        | 前後空白除去・空白の完全除去・中黒などの記号統一      | `"トヨタ  自動車"` → `"トヨタ自動車"`                               |
| furigana    | カタカナ→ひらがな変換・空白除去                      | `"トヨタ　ジドウシャ"` → `"とよたじどうしゃ"`                       |
| romaji      | 小文字化・全角→半角・不要記号削除                    | `"TOYOTA 123"` → `"toyota123"`                                     |


【実装方針】
・プログラムでは `util/normalizer.py`（または同等の共通モジュール）に実装する
・関数例：
  - `normalize_symbol(text: str) -> str`
  - `normalize_name(text: str) -> str`
  - `normalize_furigana(text: str) -> str`
  - `normalize_romaji(text: str) -> str`
・SearchService および各 SymbolAdapter は、必ずこの共通関数を通してからキャッシュやマスタに保存すること

【注意】
・ユーザー入力の「生の」文字列はsearch_cacheには絶対に保存せず、**必ず共通正規化関数で加工した後の結果のみを search_cache.keyword や symbols 等に保存すること。**
・すなわち、キャッシュやマスタには「正規化済みデータのみ」を保持し、「入力そのままの文字列」は一切使わない。


---

### ◆ 正規化関数の定義例

本設計においては、以下のような正規化関数を `util/normalizer.py` などの共通モジュールに実装し、全Adapter・Serviceでこれを利用することを想定しています。  
関数のインタフェースと処理内容は下記例に準拠してください。

#### Python 例

```python
def normalize_symbol(text: str) -> str:
    """
    銘柄コードの正規化
    ・全角→半角
    ・大文字化
    ・前後空白除去
    ・不要な接尾辞（.us, .t, .jp等）除去（内部表記ルールに合わせる）
    """
    if text is None:
        return ""
    import unicodedata, re
    text = unicodedata.normalize('NFKC', text)
    text = text.strip()
    text = text.upper()
    # EODHD形式やYahoo US等の".US"など不要な接尾辞を除去、ただし日本株 .T は残す
    # 例: "AAPL.US" → "AAPL", "GOOG.JP" → "GOOG", "7203.T" → "7203.T"
    # ".T" 以外のドット付きサフィックスは除去
    if re.match(r'^\d+\.T$', text):
        # 日本株はそのまま（例：7203.T）
        return text
    text = re.sub(r'\.(US|JP|NYQ|NASDAQ|AMEX|L|LS|AX|HK|F|SI|V|TO|PA|DE|ST|SS|SZ|KS|KQ|TW|SG|MI|MC|BR|VI|SW|BA|BC|AT|DU|BE|MU|SA|OL|HE|IR|CO|IC|TA|PR|MX|IS|VN|KL|BO|ID|CA)$', '', text)
    # 記号を削除（記号を残したい場合は必要な記号を残す処理へ）
    text = re.sub(r'[^A-Z0-9\.]', '', text)
    return text

    
def normalize_name(text: str) -> str:
    """
    名前の正規化
    ・前後空白除去
    ・スペース・全角空白などの空白文字をすべて除去
    ・中黒など同一視する記号を統一
    """
    if text is None:
        return ""
    import re
    text = text.strip()
    text = re.sub(r'\s+', '', text)  # 空白を完全に除去（スペース・全角スペース含め）
    text = re.sub(r'[･・·‧●•◦·]', '・', text)  # 中黒など記号を統一
    return text


def normalize_furigana(text: str) -> str:
    """
    ふりがなの正規化
    ・カタカナ→ひらがな
    ・空白除去
    """
    if text is None:
        return ""
    import re
    text = re.sub(r'\s+', '', text)
    def kata_to_hira(s):
        return ''.join([chr(ord(c)-0x60) if 'ァ' <= c <= 'ヶ' else c for c in s])
    text = kata_to_hira(text)
    return text

def normalize_romaji(text: str) -> str:
    """
    ローマ字名の正規化
    ・小文字化
    ・全角→半角
    ・記号除去
    """
    if text is None:
        return ""
    text = text.lower()
    import unicodedata, re
    text = unicodedata.normalize('NFKC', text)
    text = re.sub(r'[^a-z0-9]', '', text)  # 英数字以外削除
    return text


```

※ 依存のある import 文は共通モジュール先頭等でまとめてください。

---

【運用上の注意点】

- 上記関数は **必ず** search_cache や symbols 保存時、および検索時に通すこと。
- 追加で媒体特有の正規化仕様が発生した場合も、ここに追記し全プロジェクトで統一すること。
- もし他言語で実装する場合も、上記方針と互換のある形を守ってください。

---
// 【正規化処理仕様まとめ（表）】
//
// | 種別     | 正規化内容                                           | 例                                                                 |
// |----------|------------------------------------------------------|--------------------------------------------------------------------|
// | symbol   | 全角→半角・大文字化・前後空白除去・不要な接尾辞除去  | " 7203.t " → "7203.T"<br> "aapl.us" → "AAPL"                      |
// | name     | 前後空白除去・空白の完全除去・中黒などの記号統一      | "トヨタ  自動車" → "トヨタ自動車"                                  |
// | furigana | カタカナ→ひらがな変換・空白除去                     | "トヨタ　ジドウシャ" → "とよたじどうしゃ"                          |
// | romaji   | 小文字化・全角→半角・不要記号削除                   | "TOYOTA 123" → "toyota123"                                        |
//
// ▼ symbol用正規化関数（大文字／内部フォーマットに合わせる）






-----------------------------------------------------------------
3. データソース（取得元）
-----------------------------------------------------------------
【実装優先度】
  1. Yahoo（JP） … 日本株・（日本版からの）米国株取得用
  2. Yahoo（US） … 米国版の米国株取得用
  3. EODHD（US） … 米国株。将来の商用配布・安定供給用。サンプル受領済み
  4. Jクオンツ（JP） … Adapterの骨格は他と同様に実装する（中身は後から埋めても可）
  ※ここは Adapter 実装の優先度。取得経路の6通り（Business Concept）とは別物。EODHD／Jクオンツは個人契約と商用配布で同一 Adapter を共用する。

【取得経路の想定】（00_Business_Concept の6通りに準拠）
・Yahoo無料・個人契約（EODHD／Jクオンツ）：ユーザー端末が各取得元へ直接取りに行き、端末にキャッシュする
・商用配布（EODHD／Jクオンツ）：ユーザー端末が当社サーバーへ取りに行き、端末にキャッシュする（当社サーバーが上流から取得）
・いずれの場合も、取得後は Adapter で共通形に変換してから保存する
・有料ソースの Adapter は個人契約／商用配布で共用し、違いは api_key の渡し方（ユーザー入力 vs サーバー側）とする

【ソース差分の閉じ込め】
・各ソースごとに Adapter（変換係）を1つ置く
・Adapterの仕事：取得 → 共通形へ変換 → （必要なら）銘柄コード変換
・中身のフィールド名・日付形式・ティッカー表記の違いは、すべてAdapter内で吸収する


-----------------------------------------------------------------
4. 銘柄コードの統一ルール（アプリ内部）
-----------------------------------------------------------------
【内部の正式な書き方（Yahoo寄せ）】
・日本株：`7203.T` 形式（数字 + `.T`）
・米国株：`AAPL` 形式（ティッカーそのまま。`.US` は付けない）

【外部表記の変換例】
・EODHD の `AAPL.US` → 内部 `AAPL`
・（将来）Jクオンツのコード体系 → 内部 `7203.T` へ変換

【補足】
・画面表示用の名前（例: Apple Inc. / トヨタ自動車）は銘柄マスタ側で持つ
・国識別（JP / US）も銘柄マスタ側で持つ


-----------------------------------------------------------------
5. 終値の扱い（計算の正）
-----------------------------------------------------------------
【決定】
・チャート表示・移動平均・スクリーニング等の計算の正は「調整後終値」とする
  （株式分割後も証券アプリのようにチャートがつながって見えることを優先）

【保存】
・取れるソースでは次を両方保存する
  - close … その日の素の終値（実価格）
  - adj_close … 調整後終値（分割・配当などをならしたもの）
・計算工場へ渡す主価格は adj_close
・素の close は「その日の実価格が後で欲しくなったとき用」。カラム1本増えるだけなので容量影響は小さい

【ソースによって片方しか無い場合】
・ある方を両方にコピーする、などAdapter側で吸収ルールを決める（詳細は後続で詰める）


-----------------------------------------------------------------
6. 日足・週足・月足の保存方針
-----------------------------------------------------------------
【決定】
・端末に保存する「確定した日足」は daily_bars のみ
・当日の未確定足（進行中の最新値）は latest_snapshots に分離して保持する
・週足・月足は保存しない。表示・計算が必要なとき、日足からその場で作る
・分足・秒足は扱わない（ビジネス設計どおり）

【確定足の考え方（概要）】
・当日の未確定足は、原則として履歴テーブルに入れない
・市場クローズ後に確定した日足を取りに行く
・取得タイミングは、アプリ起動時やプレイリスト表示時など、ユーザーごとにずらす（負荷分散）

【気配値・最新スナップショット（最新足）】
・目的：当日進行中の「気配値／現在値＋日中OHLCV」を保持し、以下の用途に使う
  - チャート上で「今日のローソク足」をライブ描画（LIVEラベル＋色分けで確定足と区別）
  - 価格固定アラート（サーバー側15〜60分間隔チェック）
  - プレイリスト／ウォッチリストの現在値列表示
  - 乖離率などの簡易リアルタイム評価
・daily_bars とは明確に分離
  - daily_bars = 「市場クローズ後に確定した過去分」のみ（不変）
  - latest_snapshots = 「当日進行中」の可変スナップショット（定期更新・上書き）
・更新方針
  - 同じ (symbol_id, trade_date) の行を upsert で上書き
  - 市場クローズ後もその日の最終スナップショットを保持（翌営業日まで有効）
  - 市場クローズ後に「確定処理」して daily_bars に移すかは将来オプション（今は不要）
・取得頻度（目安）
  - クライアント側：表示時 or 1時間に1回程度（watchlist対象 or 表示中銘柄に絞る）
  - サーバー側（アラート用）：15〜60分に1回（0723.md 参照）
・設計上の位置づけ
  - 「確定履歴」と「当日の仮足／最新足」を分ける余白を実際に埋める
  - 将来のEODHDリアルタイム／遅延気配にも対応しやすい形にする


-----------------------------------------------------------------
7. 共通で必ず持つ項目（正規化スキーマ骨子）
-----------------------------------------------------------------
※ここがアプリ内部の「共通の形」。計算工場もチャートもこれだけ見る。


7-A. 銘柄マスタ（1銘柄1行）
-----------------------------------------------------------------
必須：
・symbol_id（内部銘柄コード。例: AAPL / 7203.T）
・country（JP / US）
・name（銘柄名）
・data_source（いま主に使っている取得元。例: yahoo_jp / yahoo_us / eodhd）

あると良い（取れる範囲で）：
・exchange（取引所）
・currency（通貨）
・（将来）isin など

※ソース固有の巨大なファンダ情報は、初期では取らない／保存しない


7-B. 日足（1銘柄×1営業日で1行）
-----------------------------------------------------------------
必須：
・symbol_id
・trade_date（日付）
・open / high / low
・close（素の終値。取れる場合）
・adj_close（調整後終値。計算の正）
・volume
・source（この行をどのソースから入れたか）
・ingested_at（取り込んだ日時。任意だがあるとデバッグしやすい）

持たない（初期）：
・週足・月足の複製
・インジケーター計算結果（都度計算。爆速のため visible なものだけ計算する既存方針を維持）


7-C. 保存形式の方針
-----------------------------------------------------------------
・日足・銘柄マスタ：正規化テーブル（端末のローカルDB想定。SQLite等）
・設定・UI・フィルター：JSON（00_Master_Design どおり）
・「全部JSON」「全部細カラム」の極端は避ける


7-D. 当日最新スナップショット（latest_snapshots）
-----------------------------------------------------------------
1銘柄 × 1営業日で1行（当日分のみ）。定期的に更新される「ライブの今日の足」。

【決定方針】
・正規化テーブルで保存（daily_bars と同じ）。JSON一括保存は避ける
  - 理由：クエリ効率（symbol_id + trade_date で高速更新・参照）、データ整合性、calculate系との親和性
・「気配値（現在値）」だけでなく「今日の始値・高値・安値・出来高」も取れる限り取得
  - 今日の始値を取ることで、チャート上で「今日のローソク足」を自然に描ける
・調整後価格の概念は不要（当日分のため分割影響はほぼない）
・bid/ask は初期では必須とせず、取れるソースのみ任意で保持（後で拡張）

【テーブル定義：latest_snapshots】

| カラム名     | 型      | 必須 | 説明・補足 |
|--------------|---------|------|------------|
| symbol_id    | TEXT    | はい | 内部銘柄コード（7203.T / AAPL） |
| trade_date   | TEXT    | はい | 日付（YYYY-MM-DD）。当日分のみ |
| open         | REAL    | いいえ | 今日の始値（取れなければ null。Adapterで吸収ルール定義） |
| high         | REAL    | はい | 日中最高値 |
| low          | REAL    | はい | 日中最低値 |
| last_price   | REAL    | はい | 現在の価格（regularMarketPrice相当。チャート上の「終値」扱い） |
| volume       | REAL    | はい | 当日累計出来高 |
| updated_at   | TEXT    | はい | このスナップショットを取得・更新した日時（ISO8601） |
| source       | TEXT    | はい | 取得元（yahoo_jp / yahoo_us / eodhd など） |

【主キー】
・(symbol_id, trade_date)

【表示・利用ルール】
・チャート描画時：今日の日付なら latest_snapshots を優先（存在しなければ daily_bars）
・「LIVE」表示：updated_at が一定時間以内 かつ 市場営業中 の場合にラベル・色分け
・計算工場：last_price を「現在値」として使用（MAタッチ・乖離率などの簡易チェック用）

【ソース差分の吸収（Adapterの責務）】
・Yahoo（JP/US）：yfinance fast_info または /quote エンドポイント → open/high/low/last_price/volume を抽出
・EODHD：real-time / last quote エンドポイント（個人・商用共通） → 同上
・J-Quants：将来対応（/equities/prices 最新行 or trades から推定）
・取れない項目（例: open が quote に含まれない場合）は Adapter 内で null にするか、1d履歴から補完するルールをここに記述
・EODHD個人契約と商用ライセンスは同一 Adapter で対応（api_key と rate-limit 差のみ config で吸収）

【daily_bars との関係】
・latest_snapshots は「仮の今日の足」
・市場クローズ後に最終値を daily_bars に正式登録する処理は将来拡張（今は latest_snapshots をそのまま保持）
・両テーブルを併用することで「確定分は不変」「当日分は最新に更新」という設計を明確に分離


-----------------------------------------------------------------
8. Adapter（変換係）の役割と抽象化設計
-----------------------------------------------------------------

【基本方針】
・各データソースごとに1つの具体的なAdapterクラスを持つ
・アプリ本体は「具体的なAdapter名」を直接書かない
・ユーザーが選んだソースに応じて、抽象的な JP_Adapter / US_Adapter を通して呼び出す
・銘柄マスタ取得・検索用のSymbolAdapterも同様に抽象化する


【価格データ用 Adapter がやること】
  1. ソースからデータを取る
  2. 銘柄コードを内部形式へ直す
  3. 日付・数値を共通スキーマへ直す
  4. close / adj_close の有無を吸収する（daily_bars 用）
  5. 気配値・スナップショット用に fetch_snapshot を実装する
  6. 共通形の配列/行として返す


【新規追加メソッド（価格データ用）】
・def fetch_snapshot(self, symbol_id: str) -> dict:
    """最新スナップショット（open/high/low/last_price/volume 等）を返す"""
    # 各ソース固有のquoteエンドポイント／ライブラリをここで呼び出し
    # 正規化して latest_snapshots 行形式の dict を返す


【共通メソッド名（価格データ用・必須）】
すべての具体的な価格Adapterは、以下のメソッド名を必ず実装すること：

・fetch_daily_bars(symbol_id, ...) → 確定日足を共通スキーマで返す
・fetch_snapshot(symbol_id) → 最新スナップショット（latest_snapshots形）を返す

この共通化により、呼び出し側は中身がYahooかEODHDかを気にせずに済む。


【ユーザーによるデータソース選択】
アプリインストール時または設定画面で、ユーザーに以下を選択させる（※個人契約・Yahoo無料の範囲）：

・日本株用ソース
  - yahoo_jp
  - jquants（個人契約・APIキー）

・米国株用ソース
  - yahoo_us
  - eodhd_personal（個人契約・APIキー）

※商用配布（EODHD／Jクオンツ）はユーザー選択肢に並べない。当社が商用配布を開始した環境では、端末は当社サーバー向け Adapter／接続先に切り替える（api_key はサーバー側）。

選択結果（Yahoo／個人契約）はユーザー設定として永続化する。


【価格データ用 Adapter の解決方法（Factory）】
ユーザー設定（Yahoo／個人契約）を元に、実際のAdapterインスタンスを返す小さな工場を用意する。
商用配布時はユーザー設定ではなく、アプリ／接続設定側で「当社サーバー向け」に切り替える。

例：

def get_jp_adapter():
    if 設定["jp_source"] == "yahoo_jp":
        return YahooJpAdapter()
    elif 設定["jp_source"] == "jquants":
        return JquantsAdapter(api_key=ユーザーのキー)

def get_us_adapter():
    if 設定["us_source"] == "yahoo_us":
        return YahooUsAdapter()
    elif 設定["us_source"] == "eodhd_personal":
        return EodhdAdapter(mode="personal", api_key=ユーザーのキー)

# 商用配布時（将来）：端末は当社サーバーから取る接続に切り替える
# （例：EodhdAdapter(mode="commercial") をサーバー側で使う／端末は自社APIクライアントを使う）
# api_key はサーバー側の秘密情報。ユーザー設定の選択肢には出さない。

アプリの他の部分（日足バッチ、気配値更新、チャート描画など）は、
必ずこの get_jp_adapter() / get_us_adapter()（または商用時の接続切替）を通して取得し、
具体的なクラス名を直接書かない。


【価格データ用 Adapter 一覧】
・YahooJpAdapter … fetch_daily_bars + fetch_snapshot 実装
・YahooUsAdapter … 同上
・EodhdAdapter … 同上（個人契約／商用ライセンスで共用。__init__ で api_key と mode を受け取る）
・JquantsAdapter … fetch_daily_bars + fetch_snapshot 実装（中身の詳細は優先度低めで後から埋めても可。骨格は他と揃える）


-----------------------------------------------------------------
8-B. 銘柄マスタ・検索用 SymbolAdapter
-----------------------------------------------------------------

【基本方針】
・銘柄マスタ情報の取得と、キーワード検索もソースごとに専用のSymbolAdapterを持つ
・価格データ用と同様に、抽象化された Factory を通して呼び出す
・アプリ本体は具体的なクラス名を直接書かない


【SymbolAdapter がやること】
  1. ソースから銘柄マスタ情報を取る
  2. 銘柄コードを内部形式へ直す
  3. 共通の銘柄マスタ形に直す
  4. キーワード検索（search_symbols）を提供する
  5. 共通形の dict / list として返す


【共通メソッド名（SymbolAdapter・必須）】
・fetch_symbol_info(symbol_id) → 1銘柄のマスタ情報を返す
・search_symbols(query: str) → キーワードに一致する候補リストを返す


【SymbolAdapter の解決方法（Factory）】
ユーザー設定（Yahoo／個人契約）を元に解決する。
商用配布時はユーザー設定ではなく、アプリ／接続設定側で「当社サーバー向け」に切り替える。

例：

def get_jp_symbol_adapter():
    if 設定["jp_source"] == "yahoo_jp":
        return YahooJpSymbolAdapter()
    elif 設定["jp_source"] == "jquants":
        return JquantsSymbolAdapter(api_key=ユーザーのキー)

def get_us_symbol_adapter():
    if 設定["us_source"] == "yahoo_us":
        return YahooUsSymbolAdapter()
    elif 設定["us_source"] == "eodhd_personal":
        return EodhdSymbolAdapter(mode="personal", api_key=ユーザーのキー)

# 商用配布時（将来）：端末は当社サーバーから取る接続に切り替える
# EODHDへの接続はサーバーだけが行い、端末は自社サーバーから受け取る
# api_key はサーバー側の秘密情報。ユーザー設定の選択肢には出さない。


【SymbolAdapter 一覧】
・YahooJpSymbolAdapter
・YahooUsSymbolAdapter
・EodhdSymbolAdapter（個人契約／商用ライセンス共用）
・JquantsSymbolAdapter（骨格は他と同様。詳細実装は後回し可）


【search_cache との役割分担】
・SymbolAdapter は「外部から候補を取ってくるだけ」に専念する
・search_cache テーブルへの読み書きは SearchService（上位の検索ロジック）が担当する

流れ：
  1. ユーザーが検索窓にキーワードを入力
  2. SearchService がまずローカルの search_cache を確認
  3. キャッシュにあればそれを返す
  4. なければ該当する SymbolAdapter の search_symbols() を呼ぶ
  5. 取得結果を search_cache に保存してから返す

この分離により、Adapterの責務が明確になり、キャッシュ戦略を後から変更しやすくなる。


【設計の利点】
・後から新しいデータソース（例: 欧州株、別のUSプロバイダ）を追加しても、
  影響範囲は「設定画面」と「Factory」だけに抑えられる
アプリ本体のコードはソースの違いを知らなくて済む
・テスト時にモックAdapterを差し替えやすい
・価格データ用と銘柄マスタ用で同じ抽象化パターンを使えるため、学習コストが低い



### 8-C. search_cache テーブルと SearchService の設計

search_cache は「**検索体験を高速化するための補助テーブル**」です。  
銘柄マスタ（symbols）とは明確に役割を分け、検索特化のデータを保持します。

#### テーブル定義：search_cache

```sql
CREATE TABLE search_cache (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    keyword TEXT NOT NULL,                    -- 正規化された検索対象語（シンボル・正式名・ふりがな・ローマ字など）
    symbol_id TEXT NOT NULL,                  -- 内部銘柄コード（symbols.symbol_id と紐づけ）
    display_name TEXT NOT NULL,               -- 表示用銘柄名（「トヨタ自動車」「Tesla Inc.」など）
    furigana TEXT,                            -- ふりがな（JP銘柄のみ。US銘柄はNULL可）
    match_type TEXT NOT NULL,                 -- 'symbol' / 'name' / 'furigana' / 'romaji' など
    match_score INTEGER DEFAULT 100,          -- 一致度（高いほど優先）
    search_count INTEGER DEFAULT 1,           -- この組み合わせで検索された回数（人気順ソート用）
    last_searched_at TEXT,                    -- 最終検索日時（ISO8601）
    created_at TEXT NOT NULL,                 -- 初回キャッシュ作成日時
    source TEXT NOT NULL,                     -- どのAdapterから取得したか（yahoo_jp / eodhd / jquants など）
    
    UNIQUE(keyword, symbol_id, match_type)    -- 同じ正規化キーワード＋銘柄＋マッチ種別の重複を防ぐ
);




-- 検索性能向上のためのインデックス例
CREATE INDEX idx_search_cache_keyword ON search_cache(keyword);
CREATE INDEX idx_search_cache_symbol_id ON search_cache(symbol_id);
CREATE INDEX idx_search_cache_last_searched_at ON search_cache(last_searched_at);
```

#### カラム説明と設計意図

| カラム             | 目的・意図                                                                 | 備考 |
|--------------------|-----------------------------------------------------------------------------|------|
| `keyword`          | 正規化された検索対象語（シンボル・正式名・ふりがな・ローマ字など）         | ユーザの生入力は入れない |
| `symbol_id`        | symbolsテーブルとの紐付け                                                   | FK的な役割 |
| `display_name`     | 画面に表示する名前                                                          | 検索結果リストでそのまま使える |
| `furigana`         | JP銘柄の読み（検索マッチングに使用）                                       | US銘株はNULL |
| `match_type`       | どの種類のキーワードか（後でフィルタや表示に活用可能）                     | 将来的な拡張に強い |
| `match_score`      | 一致の強さ（exact=100, prefix=80 など）                                    | ソート時に使用 |
| `search_count`     | 人気度（よく検索される銘柄を上位に）                                       | 更新時に +1 |
| `last_searched_at` | 最終検索時刻（古いキャッシュの掃除や優先度に使用）                         | - |
| `source`           | どのデータソースから来たか                                                 | デバッグ・更新管理用 |



#### SearchService の役割と処理フロー

SearchService は「検索の窓口」として、以下の責務を持ちます：

- search_cache の読み書きを一元管理
- SymbolAdapter への問い合わせを制御
- 検索結果の正規化・ランキング（**正規化は必ず「2. 共通データ正規化規則」の関数を使う**）

**基本的な検索フロー（SearchService.search(query)）**:

1. ユーザー入力 `query` を共通正規化関数で処理する
2. `search_cache` から `keyword` が前方一致するレコードを検索
   - ヒットしたら `search_count` を +1、`last_searched_at` を更新
   - 結果を `match_score` + `search_count` でソートして返す
3. ヒットしない、または結果が少ない場合：
   - 現在のデータソース設定に応じて適切な SymbolAdapter を取得
   - Adapter の `search_symbols(query)` を呼ぶ（生のqueryを渡す）
   - 返ってきた候補を、共通正規化関数（`normalize_symbol` / `normalize_name` / `normalize_furigana` / `normalize_romaji`）で整える
   - 整えた keyword を `search_cache` に UPSERT する
4. キャッシュした結果を再度検索して返却

**擬似コード例**:

```python
def search(self, query: str):
    # 1. ユーザー入力を共通正規化関数で処理
    candidates = [
        normalize_symbol(query),
        normalize_name(query),
        normalize_romaji(query),
        normalize_furigana(query),
    ]
    candidates = [c for c in candidates if c]  # 空文字を除去
    
    # 2. キャッシュから検索
    cached = []
    for norm in candidates:
        cached += self.db.search_cache.find_by_keyword_prefix(norm)
    
    if cached:
        self._increment_search_count(cached)
        return self._rank_and_dedup(cached)
    
    # 3. キャッシュにない場合 → Adapter に問い合わせ
    results = []
    if self.is_jp_mode or self.search_both:
        adapter = get_jp_symbol_adapter()
        results += adapter.search_symbols(query)   # 生のqueryを渡す
    
    if self.is_us_mode or self.search_both:
        adapter = get_us_symbol_adapter()
        results += adapter.search_symbols(query)
    
    # 4. 結果を共通正規化関数で整えてから search_cache に保存
    for item in results:
        norm_symbol = normalize_symbol(item.symbol_id)
        norm_name   = normalize_name(item.name)
        norm_furi   = normalize_furigana(item.furigana) if getattr(item, "furigana", None) else None
        norm_romaji = normalize_romaji(item.name)
        
        keywords_to_save = [
            (norm_symbol, "symbol"),
            (norm_name, "name"),
        ]
        if norm_furi:
            keywords_to_save.append((norm_furi, "furigana"))
        if norm_romaji:
            keywords_to_save.append((norm_romaji, "romaji"))
        
        for kw, mtype in keywords_to_save:
            if not kw:
                continue
            self.db.search_cache.upsert({
                "keyword": kw,
                "symbol_id": norm_symbol,
                "display_name": item.name,          # 表示用は元の名前を残す
                "furigana": getattr(item, "furigana", None),
                "match_type": mtype,
                "source": item.source,
                "created_at": now()
            })
    
    # 5. 再度キャッシュから取得して返却
    cached = []
    for norm in candidates:
        cached += self.db.search_cache.find_by_keyword_prefix(norm)
    return self._rank_and_dedup(cached)
```

#### 設計のポイント

- **symbols テーブルとは分離**している理由  
  symbols は「公式情報」の保存場所。search_cache は「検索体験を良くするためのインデックス兼キャッシュ」。
- search_cache には「正規化された検索対象語」を保存する（ユーザの生入力は入れない）
- US株も積極的にキャッシュする（「tes → Tesla」のような予測表示のため）
- 同じ `keyword + symbol_id + match_type` の組み合わせは重複させない（UNIQUE制約）
- 将来的に「ローマ字検索」「あいまい検索」などを追加しやすい構造
- 古いキャッシュの掃除は `last_searched_at` を見て定期的に行う（任意）

この設計にすることで、**JP株もUS株も同じ検索UIでサクサク動く**ようになります。
```




-----------------------------------------------------------------
9. 「全部取る」か「共通だけ取る」か
-----------------------------------------------------------------
【決定方針】
・初期は「共通スキーマに必要なもの＋安く取れて後で効くもの」だけ取る
・各社の全フィールドを端末に溜めない（容量・設計の肥大化を防ぐ）

【必ず取る】
・日足OHLCV（close / adj_close / volume 含む）
・銘柄の基本識別情報（コード・名前・国）

【今は取らない（必要になったら足す）】
・詳細ファンダ全部
・ニュース、財務諸表の深い項目など

【将来枠】
・ソース固有の追加情報を置きたくなったら、別領域（拡張用）に逃がす余地だけ残す
・最初から本体スキーマをソース固有項目で汚さない


-----------------------------------------------------------------
10. 端末キャッシュの範囲
-----------------------------------------------------------------
・日米の全銘柄を端末に落とさない
・ユーザーが登録した銘柄（プレイリスト等）に限定して日足をキャッシュする
・Yahoo系は1日あたりの取得上限があるため、登録銘柄数に上限を設ける余地あり（数は別途検討）
・EODHD等のAPIキー利用時はYahoo上限の対象外だが、端末容量への軽い警告は出す


-----------------------------------------------------------------
11. EODHDサンプルから分かっていること（根拠メモ）
-----------------------------------------------------------------
※ notes/EODHD — US SAMPLE DATA  Mr.Igor より

【EOD（日足）】
・項目例：date, open, high, low, close, adjusted_close, volume
・→ 本書の共通日足スキーマとほぼ一致。adjusted_close を adj_close にマップ

【銘柄メタ】
・Code, Name, Exchange, Currency など
・→ 銘柄マスタの「あると良い」項目の根拠

【リアルタイム / 遅延気配】
・1銘柄の最新気配を1回で取れる形（open/high/low/close/volume 等）
・→ latest_snapshots として正式に採用（6番・7-D参照）


-----------------------------------------------------------------
12. 計算工場との境界
-----------------------------------------------------------------
・Data Layer の出口：共通日足（必要ならその場で作った週足・月足）
・indicators.py の入口：上記＋注文書（config_json）
・Data Layer はインジケーターを計算しない
・indicators.py はソースの違いを知らない


-----------------------------------------------------------------
13. 後続で詰める明細（TODO）
-----------------------------------------------------------------
・[ ] Yahoo JP / Yahoo US の実際のレスポンス項目対照表
・[ ] EODHD のエンドポイント一覧と、共通スキーマへの項目マップ表
・[ ] close が片方しか無いときの正式な吸収ルール
・[ ] 週足・月足の集約ルール（週の区切り、月の区切り、四本値の取り方）
・[ ] upsert（同じ日付が来たときの上書き）規則
・[ ] 端末DBのテーブル名と型の正式定義
・[ ] 取得失敗・レート制限時のリトライ / ユーザー向けメッセージ
・[ ] 商用配布時の自社サーバーAPIの詳細（エンドポイント・認証・エラー形式）。切替方針自体は §3 取得経路および Factory に記載済み
・[ ] Jクオンツの詳細実装
・[ ] データソース選択画面のUI仕様
・[ ] AdapterFactory の正式な実装方針
・[ ] search_cache テーブルの正式なカラム定義
・[ ] SearchService の実装方針


-----------------------------------------------------------------
14. 00_Master_Design との関係（修正メモ）
-----------------------------------------------------------------
Master Design 冒頭の「最終的にOHLCV共通フォーマットへ」は本書で具体化する。
生データの保存形式は「設定のJSON」とは別物である点を、実装時に混同しないこと。


=================================================================
以上（骨子）
=================================================================
