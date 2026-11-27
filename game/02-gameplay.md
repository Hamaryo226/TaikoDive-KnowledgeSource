# 02｜演奏エンジンの共通仕様

対象は固定コミット G `fb12417`。本文の演奏時刻は特記なき限りミリ秒。[採点の詳細](02b-scoring.md)、[モード別仕様](02c-modes.md)、[時計と描画](02d-timing-and-render.md) を併用する。競合時は具体的な補足を優先。以下と関連する元テストの実行結果は [13](13-baseline-verification.md)。

## PLAY-01 時計とフレーム順

演奏時計 `ctNowTime` は開始時、通常ロードで -3000、パス指定ロードで -4800 に設定される。再生速度 `dbNowSpeed` に対し時計の刻みは `1000 / dbNowSpeed` に再設定する。音声がまだ始まっておらず、時刻が非負かつ段位譜面でないとき、音声再生を要求し、正の時刻から開始する場合はその位置を渡す。段位音声は別経路である。根拠: `G/src/Scenes/Game/NotePlay.cs::tTJA読み込み`, `SongPlay`, `tSpeedChangeInit`。

描画フレーム冒頭でポーズ切替を処理し、非ポーズかつ特訓なら再生中に限り `SongPlay` を呼ぶ。非ポーズならリプレイを描画より前に適用する。その後に描画を進め、フレーム末尾側の `Update` にて操作入力、1P共有チャンネル、2P分岐、ゴースト分岐、スキップ、終了の順に処理する。時計 tick はこの後。つまり録画再生と通常キー処理を同じ段階に移動してはならない。`G/src/Scenes/Game/NotePlay.cs::tDraw,Update`。

## PLAY-02 手入力と判定

1P入力は左ドン、右ドン、左カ、右カを個別に録画するが、同一フレーム内の左右同色は1回の判定処理にまとめ、左を優先する。ドンの処理後にカを処理。2Pも同色左右をまとめ左優先、ドン→カ。リプレイ再生時は1P実入力を受け付けない。ゴースト時は通常2P実入力を処理しない。特訓停止中、ドンは選択小節の再開、左右カは小節移動。Spaceによる同フレーム操作済みならドンを処理しない。`G/src/Scenes/Game/NotePlay.cs::Update`。

判定は `lag = 時刻整数 - 譜面時刻` をゼロ方向へ整数化し、その絶対値を閾値と比較する。各閾値は**以下**を含む。難易度 index 0–1 は良 41.7、可 108.4、不可 125.1；それ以外は良 25、可 75、不可 108.4。いずれも超えれば通り越し。通常2Pは自分の難易度、AIは1P基準。実際に比較される lag は整数なので 41.7 の境界で 41 は良、42 は可。`G/src/Scenes/Game/NotePlay.cs::t難易度の判定幅,GetNowJudge`。

候補は可視かつ未処理の通常音符と、条件に合う連打を走査する。前後候補がある場合、整数化済みの絶対差が等しければ前候補。後候補の方が近くても前候補が未処理で、遅れが不可窓以下なら前候補を優先する。後候補が有効時間内の連打・大連打・風船なら後候補に上書きする。この後者の範囲は両端を含む。継続中連打の前向き走査条件は終端が現在時刻より**大きい**。[02d の探索端点](02d-timing-and-render.md) に、始点・末尾・巻き戻しと連打の特例を列挙する。`G/src/Scenes/Game/Objects/GetNoteProcess.cs::tGetNoteProcess,b継続中の連打である`。

## PLAY-03 通常音符・連打・風船

通常音符の良・可でコンボを1増やし、不可・通り越し・お手付きではコンボを0にする。可は基本点の整数除算による半額、良は基本点を加点する経路を使う。ゲージは特訓の1Pでは更新しない。`G/src/Scenes/Game/Objects/OnpuProcess.cs::tノーツ処理`。

連打・大連打は終端がなければ無視し、開始≦入力時刻≦終端で1打を数える。各打は音符・分岐・プレイヤーの連打数、および対象なら段位の現在曲連打数を増やす。2P通常モードでは1打100点。風船も同じ時刻境界と終端必須条件、必要打数が0以下なら1打に補正し、既に達成済みなら追加しない。最後の打も分岐・合計・段位の連打数に含む。未達打は100点加算経路、最後の打は [02b](02b-scoring.md) のとおり 0 点。`G/src/Scenes/Game/Objects/RendaProcess.cs::tRoll処理,tBalloon処理,t段位の連打数を加算`。静的テスト根拠: `G/tests/TaikoDive.Tests/Tests/DanBalloonCountTests.cs::DanBalloonCountTests`。

## PLAY-04 分岐

SECTION で対象プレイヤーの分岐集計を初期化。p条件の指標は 良÷(良+可+不可)×100、分母0なら0。r条件は連打数。指標がA未満なら普通、A以上B未満なら玄人、B以上なら達人。r条件の判定後はその連打数を0にする。LEVELHOLD中は切替しない。2P分岐コマンドは処理時刻順、同時刻なら元の格納順で処理し、共有BGMや段位切替を2重実行しない。分岐時刻以降の音符・小節線は選択された分岐だけ可視にする。ゴーストは記録された分岐を用い、記録がなければ直前の分岐を維持する。`G/src/Scenes/Game/ChannelProcessing.cs::e精度に合う分岐を選ぶ,t2PBranchProcessing,tGhostBranchProcessing,tInvisibleExceptForSpecifiedCourses`。

強制分岐判定は、A≥101かつB≥102なら普通、そうでなくA≤-1かつB≥101なら玄人、そうでなくA≤-2かつB≤-1なら達人。`G/src/Scenes/Game/ChannelProcessing.cs::DetermineIfItIsAForcedBranch`。

## PLAY-05 座標

通常スクロールにおける判定位置からの横方向距離は `(譜面時刻 − 現在時刻 − 整数化した NotesPosition) × 音符BPM × 音符SCROLL × (Speed + 1.5) / 419`。負SCROLLは向きを反転する。HBSCROLLでは代わりに `(音符のBMScrollTime − 現在PBMTime) × 25 × 2.5` を使用する。通常スクロールの描画間引きは、譜面時刻が単調でない、実効速度の絶対値が0.001未満、分岐譜面、HBSCROLLのいずれかでは安全とみなさない。`G/src/Scenes/Game/Objects/DrawNotes.cs::tDrawNotes,tComputeCullingParams`。

## PLAY-06 配点・ゲージ・ランク

基本点を計算する。通常音符数 N は非分岐なら TotalNotes − 通常風船数、分岐なら達人の総音符数。R は終端付き通常連打・大連打の長さ合計ms、Bは通常風船の要求打数合計。控除 D は Edit なら -2000、そうでなくレベル9なら35000、それ以外0。N≤0なら基本点0。非分岐では `round_even((1000000 − D − 15 × R/1000 × 100 − (B+1) × 100) / N / 10) × 10`。分岐では `round_even((1003300 − floor(floor(R/1.5)/46) × 100) / N / 10) × 10`。round_even は中点を偶数側に丸める。極の基準は分岐なら1000000、非分岐なら `max(1000000, 整数化(基本点 × TotalNotes + R/1000 × 16.6 × 100 + B × 100))`。`G/src/Scenes/Game/NotePlay.cs::t譜面の配点を計算する`。

ゲージは0で初期化し、判定ごとに良で成長量、可で成長量の半分、それ以外で成長量×減少倍率を減算、毎回0〜100に制限。分岐譜面は音符の分岐に対応する成長量を用いる。通常クリア境界は難易度 index 0〜4 についてそれぞれ 60、70、70、80、80 以上。難易度・レベル別係数は [02b](02b-scoring.md)。`G/src/Scenes/Game/Objects/Gauge.cs::Init,GaugeCalculation,dbClearGaugeMem`。

## PLAY-07 モード・操作・保存条件

演奏中の速度変更は上下で0.1刻み、範囲0.2〜4.0、小数第1位を中点でゼロから遠ざける丸め。1Pオートの途中変更は録画を取り消す。通常記録の保存は通常演奏・非ゲスト・非リプレイを前提とし、当該プレイヤーが演奏中に一度もオートを使っていないこと。2Pにはさらに2Pモードかつ非ゴーストが必要。`G/src/Scenes/Game/NotePlay.cs::Update,b通常記録を保存できる`。

段位の得点・判定・コンボ・連打は音源開始数ではなく `nNowSong` が指す現在課題曲に帰属する。曲別コンボは曲ごとに独立し、不可でその曲の現在コンボを0にする。`G/src/Scenes/Game/NotePlay.cs::現在曲の段位スコア`, `G/src/Scenes/Game/Objects/OnpuProcess.cs::tノーツ処理`。静的テスト: `G/tests/TaikoDive.Tests/Tests/DanScoreAttributionTests.cs::DanScoreAttributionTests`。

Esc は終了演出中または既に選曲画面へ戻る要求中でなければ戻り要求を立てる。ポーズ中は通常ノーツ処理をしない。`G/src/Scenes/Game/NotePlay.cs::Update,tDraw`, `G/src/Scenes/Game/Objects/OnpuProcess.cs::tノーツ処理`。Pause の音声再開位置の退避・破棄についての静的テストは `G/tests/TaikoDive.Tests/Tests/ReplayAndPauseRegressionTests.cs::ReplayAndPauseRegressionTests`。

## PLAY-08 受入例（静的仕様から導出、未実行）

1. 難易度 index 2、lag が ±25 なら良、±26 は可、±75 は可、±76 は不可、±108 は不可、±109 は通り越し。
2. index 0 で現在時刻1000、譜面時刻958.9なら差41.1を41へ整数化して良。譜面時刻957.9なら差42.1を42へ整数化して可。
3. 分岐 p、良7・可2・不可1、A=70、B=90なら玄人。良9・可1・不可0なら達人。分母0、A=1なら普通。
4. 風船の開始1000、終端2000、要求0なら1000での1打で達成。2001では受理されない。終了紐付けがない場合は受理しない。
5. 左右ドンが同一フレームで押された場合、録画上は両入力、通常判定呼出しは左ドン1回。ドンとカが同時ならドンを先に処理する。

## PLAY-99 同等性の残条件

判定候補が複数の連打と通常音符・分岐をまたいで同一時刻に重なる異常譜面、およびフレーム落ちと実機オーディオ遅延の同時影響は網羅検証していない。時計の積算・色の変換・小節線は [02d](02d-timing-and-render.md)、特殊モードは [02c](02c-modes.md)、得点・ゲージの値は [02b](02b-scoring.md) を使用する。
