# TJA互換パーサ仕様

対象は game リポジトリ main fb12417。根拠表記 `G/` はそのルートからの相対パス、`::` 以降はシンボル。以下は特記しない限り実装上確定した挙動であり、一般的なTJA規格への準拠を意味しない。選曲時のディレクトリ走査、ジャンルの決定・表示は別章の責務であり、本章ではファイル内の GENRE 文字列の格納のみ扱う。

## TJA-01 入力・字句

`G/src/Common/Song/TJA.cs::tTJAParse,tStripComment`。Shift_JIS 指定で全ファイルを読み、.NET の BOM 自動検出にも従う。環境の改行を LF に置換して LF 分割、空文字行を除去。単独 CR は一般には改行と認識しない。各行の最初の `//` と直前の半角空白だけを削除する。引用符やURL内でも例外なし。タブは直前でも削らない。空白でインデントされたコマンドはコマンドにならない。ヘッダは最初の `:` のみで分割し、キーと値を Trim、大小文字は個別の明記された別名以外区別する。複数コロンの値は保持する。譜面行は一文字ずつ走査し、Unicode の digit を数字として処理、`,` が小節を確定、他文字は無視する。数字変換に失敗した digit は 0。小節は複数行にまたがれる。行頭 `#` は譜面数字として処理しない。コマンドは行全体を `^(#[A-Z]+)(?:\s?)(.+?)?$` で解釈し、未知コマンドは無視する。引数前の空白は最大1文字が消費され、残りは引数になる。

整数/実数は現行カルチャの .NET TryParse、失敗は 0、実数 NaN/Infinity も 0。実数除算のゼロ分母などその後の非有限値は別途防止されない。譜面内の小節が末尾カンマで確定されなければ未確定データは出力されない。`G/src/Common/Song/TJA.cs::DoubleParse,intParse,tNotesParse`。

## TJA-02 ヘッダと既定値

新規 Header の文字列は null、数値は 0、フラグは false。Course は Oni、LEVEL/SCOREINIT/SCOREDIFF/音符数 0、風船配列 null、分岐・HBSCROLL false。各難易度別に Course を持つ。ヘッダは通常行のどこでも評価され、全曲ヘッダと現在コースヘッダは両方処理される。`G/src/Common/Song/TJA.cs::Header,Course,tHeaderParse,tCourseParse`。

|キー（別名）|格納と意味|
|---|---|
|TITLE|曲名文字列|
|SUBTITLE|値が `--` をどこかに含むと先頭2文字を削除。それ以外はそのまま|
|BPM|実数、現在コースの BPM 表0番も更新|
|WAVE|音源パス、ネットワークパス除外|
|MOVIE / Movie|動画パス、同除外|
|OFFSET|秒。開始時刻は符号反転してミリ秒化。BPM表0のスクロール位置も更新|
|SONGVOL、SEVOL|整数|
|DEMOSTART|実数|
|SCOREMODE|整数|
|GENRE|文字列のみ。フォルダ等のジャンル決定は別章|
|STAGE / Stage|文字列|
|COURSE / Course|現在コースを変更、存在フラグを立てる。Edit は裏譜面フラグも立てる|
|LEVEL|現在コース整数|
|BALLOON / BALLOONNOR|現在コース普通配列。空文字・空白は空配列|
|BALLOONEXP、BALLOONMAS|玄人・達人配列。カンマ分割し各整数を Trim して変換|
|SCOREINIT、SCOREDIFF|現在コース整数|

コース値は `0/EASY/Easy/easy`、`1/NORMAL/Normal/normal`、`2/HARD/Hard/hard`、`3/ONI/Oni/oni`、`4/EDIT/Edit/edit/URA/Ura/ura`。他は Oni。STYLE は無視される。未指定 COURSE は初期 Oni だが存在フラグは自動設定されない。`G/src/Common/Song/TJA.cs::CourseParse,NowInfo.Init`。

## TJA-03 セッションと選択

ゲーム解析は初期状態でコース外、`COURSE` または `Course` の行頭キー、または行頭 `#START` でコース内になる。コース内で行頭 `#` を見ると譜面解析開始、以後 `#END` まで行を解析する。ゲーム解析中に同一 COURSE を再び宣言すると、そのコースオブジェクトと風船カウンタを新規化し、未確定音符と命令を消す。よって後の同難易度が前を置換する。`#START P1` / `#START P2` の行頭一致時、末尾が他プレイヤー名ならそのブロックを厳密な `#END` 行まで飛ばす。player 0 が P1、他は P2。無印は双方で解析される。P1/P2 の末尾に余分な文字があれば判定が変わる。`G/src/Common/Song/TJA.cs::tTJAParse`、`G/tests/TaikoDive.Tests/Tests/TJA譜面Tests.cs::同じ難易度の一人用と二人用があっても各プレイヤーの譜面だけ読む`。

非ゲーム解析は全行について `#BRANCHSTART` の行頭で現在コースの分岐フラグを立て、コロンのある行のみヘッダとコースを読む。音符は解析しない。フォルダ走査・ファイル採用はこのパーサの外。

## TJA-04 小節確定・時刻

`G/src/Common/Song/TJA.cs::tNotesParse,NowInfo.Init`。初期解析状態の時刻0、BPM120、スクロール1、拍子比1、スクロール積分0。`#START` は現在時刻を `-OFFSET×1000` ms、現在 BPM をヘッダ BPM、拍子比1、スクロール1、通常分岐、表示小節線、ゴーゴー無効にする。ただしスクロール積分をリセットしない。BPM表0はヘッダ BPM が正ならそれ、そうでなければ120だが、現在 BPM 自体はヘッダ値のままである。

数字を N 個蓄積しカンマで確定する。0 もスロットとして数える。i 番目に保存された BPM を Bᵢ、拍子比を Mᵢ とすると、音符の時刻と積分位置を現在値で記録後、時刻を `15000/Bᵢ × Mᵢ × 16/N` ms、積分を `Mᵢ × 16/N` 進める。空小節は現在 BPM・拍子で同じ式の N=1 相当を進める。SCROLL 値は記録するが時刻やこの積分に掛けない。値は double で保持し、パーサ内で整数ミリ秒に丸めない。ゼロ BPM などの非有限演算は不正入力として残る。小節線は確定時に1個作り、非空なら最初の音符位置、空なら現在位置。小節線の表示状態は確定時の値である。分岐終了後の小節線はループ内で分岐を変更するだけで複製されず、結果は Master の1個になる。

命令の多くは現在の未確定数字数を添えて保留し、カンマ時に対応する数字の直前で順に処理する。保留位置が N と等しい命令、および数字0個の小節の保留命令は処理されずクリアされる。`#DELAY` はその位置で時間を秒×1000、積分を秒×1000×確定時の現在 BPM/15000 進める。DELAY 自身の時刻・積分はこの経路では設定されず既定0。BPMCHANGE の BPM表への登録時刻はコマンドを読んだ時の現在値で、未確定小節の途中位置ではない。即時チップの SECTION 等も同様。`#START` と `#END` も保留され、次の確定数字がなければ出力されない。

|検証入力（各行を順に配置）|期待値|
|---|---|
|BPM:120、OFFSET:0、#START、1111,|音符時刻 0,500,1000,1500 ms、積分 0,4,8,12|
|BPM:120、OFFSET:-1.5、#START、1111,|1500,2000,2500,3000 ms|
|BPM:120、#START、1,、#DELAY 1、1,|音符 0,3000 ms。後者の積分は 24|
|BPM:120、#START、#MEASURE 2/4、11,、#MEASURE 4/4、11,|音符 0,500,1000,2000 ms|
|BPM:120、#START、1111,、#BPMCHANGE 240、1111,|0,500,1000,1500,2000,2250,2500,2750 ms|

これらの時刻・拍子・停止・積分は `G/tests/TaikoDive.Tests/Tests/TJA譜面Tests.cs::チップの時刻がBPMと小節の音数から決まる,OFFSETが開始時刻をずらす,停止は以降の時刻を後ろへずらす,HBSCROLLはノーツがある小節でも描画位置を進める` により裏付けられる。DELAY の積分 24 は式からの計算。

## TJA-05 音符と風船

`G/src/Common/Song/TJA.cs::ENotes,tNotesParse,n次の風船の打数を取る,GetRollBeginInRollEnd`。

|文字|種類|通常音符数に加算|
|---|---|---|
|0|空白|いいえ|
|1 / 2|小ドン / 小カツ|はい|
|3 / 4|大ドン / 大カツ|はい|
|5 / 6|連打 / 大連打|いいえ|
|7|風船|いいえ|
|8|連打終端|いいえ|
|9|くすだま|いいえ|

5,6,7,9 は最新の開始番号を保存。8 で同番号の開始を未確定リスト、次いで確定リストの後方から探して終端を関連付ける。開始は終端チップへの参照を持つ。分岐終了後は3分岐を探す。見つからなければ何もしない。7 だけ風船打数を消費。EXP/MAS が両方 null なら通常配列を全分岐共通の出現順で消費。どちらか存在すれば分岐ごとに独立カウンタを使い、対応配列が null の分岐は通常配列を使う。欠落、配列枯渇、0以下は7打。空の EXP/MAS は null ではなく空配列であり、通常配列にフォールバックしない。

1〜4 は TotalNotes に元データ1個あたり1加算。分岐前・分岐終了後は3分岐のカウンタ全てに加算、分岐中は現在分岐のみ。分岐終了後の音符は確定時に Normal/Expert/Master へ3複製されるが TotalNotes は3倍にならない。`G/tests/TaikoDive.Tests/Tests/TJA譜面Tests.cs::分岐ごとのBALLOONを使い分ける,分岐ごとに音符を数える`。

## TJA-06 コマンドと分岐

`G/src/Common/Song/TJA.cs::tNotesParse,BranchParse`。未列挙の命令は無視。コマンド行は大文字のみ。

|命令|動作|
|---|---|
|#START / #END|開始状態設定 / 解析終了。START/END を保留|
|#HBSCROLL|現在コースの HBSCROLL を有効化。#BMSCROLL は未対応で無視|
|#BPMCHANGE 値|現在 BPM を変更、BPM表へ登録し命令保留|
|#SCROLL 値|倍率変更、命令保留|
|#MEASURE a/b|拍子比 a/b に変更しチップ保留。チャンネル未設定の既定値は START|
|#DELAY 秒|停止命令保留|
|#GOGOSTART / #GOGOEND|状態を変更し命令保留|
|#BARLINEOFF / #BARLINEON|小節線表示フラグ変更。チップは出力しない|
|#LYRIC 文字列|歌詞配列に引数を追加し LYRICSCHANGE 保留|
|#SECTION / #LEVELHOLD|現在時刻のチップを即追加|
|#BRANCHSTART 条件,A,B|分岐開始、条件チップと復元用スナップショットを作る|
|#N / #E / #M|スナップショットの時刻・積分・BPM・スクロール・拍子を復元し分岐を切替|
|#BRANCHEND|通常分岐に戻し、以降の音符を3分岐複製。命令チップは出さない|

分岐引数から全空白を削除。2文字目が ASCII 数字ならその前にカンマを挿入し、カンマ区切りの条件、A、B を読む。条件 `p`,`r`,`s`、未知は p。BRANCHSTART の判定時刻は現在時刻、チップ時刻は `現在時刻 - 15000/BPM×拍子比×(rなら11、他は16)`。#N/E/M はゴーゴー状態や小節線を復元しない。#BRANCHEND 時点は最後に解析した経路の終点で、最長経路の計算はしない。開始前の #N 等、短い分岐引数は例外になることがある。分岐後の新たな #BRANCHSTART が終了状態を解除する。`G/tests/TaikoDive.Tests/Tests/TJA譜面Tests.cs::どの分岐も同じ時刻から始まる,分岐が終わったあとの音符は三分岐ぶんに複製される`。

ゲーム解析の最後に全5コースのリスト先頭へ RESETLAG（時刻 -6000 ms、BPM120、スクロール1、拍子1）を挿入する。リストは一般に時刻順にソートされない。

## TJA-07 HBSCROLL と BMS系の境界

HBSCROLL はコースフラグで、音符の積分位置を使う描画方法を選ぶ。各 BPM 表項目は登録時点の時刻・積分・分岐を保持する。BPM 表0は START 時に再生成され、OFFSET 行での以前の書換えが上書きされうる。`G/src/Common/Song/TJA.cs::tHeaderParse,tNotesParse,CBPM`、`G/src/Scenes/Game/NotePlay.cs::dbHBScroll現在位置`。BMSCROLL という命令を HBSCROLL の別名として扱わない。描画上の区間・分岐の優先順位と補間式は [02d 時計・描画詳細](02d-timing-and-render.md) を参照する。

## TJA-08 段位ファイル

`G/src/Common/Song/TJA.cs::tDanParse,GetType,GetRange`。Shift_JIS とコメント除去は共通。空行と先頭 `;` 行を無視。`TJA`, `COURSE`, `TITLE`, `EXAM1`〜`EXAM3` は最初のコロンで分割し現在の課題曲に代入。`NEXT` 行は現在課題曲を追加して新規化する。EOF で自動追加しない。`[Common]` 以降は共通モードが戻らない。共通 EXAM は既に追加済みの全曲へも適用し、同時に現在の課題曲にも書く。GAUGE は2要素、EXAMは4要素以上が必要。対象キーにコロンがない場合と要素不足は InvalidDataException。共通 `MAINTITLE`, `SUBTITLE`, `DANTYPE` を文字列として格納。

EXAM は `種類,赤,金,範囲`。種類 c=Combo、g=Gauge、h=Hit、jb=JudgeBad、jg=JudgeGood、jp=JudgePerfect、r=Roll、s=Score、未知は Combo。範囲 l=Less、他は More。数値は前述 TryParse。曲ごとの TITLE は実際に譜面をロードした後、その譜面の TITLE で上書きされる。

課題曲を相対パスで読み込み、指定 COURSE のチップを Oni レベル10の合成コースへ連結。最初の移動量は4000ms、最初の BGMCHANGE も4000ms。各曲は元リスト順で各チップ時刻と RollEnd 時刻に移動量を加算し、最後のチップ時刻+1500ms を次の境界とする。曲間 CURTAIN は境界、BGMCHANGE は境界+6000ms。次曲移動量はその BGMCHANGE 時刻−合成ヘッダ OFFSET×1000。合成ヘッダ BPM は最初の曲の BPM。連結時の参照共有やソートされていない最終チップによる影響を補正しない。Plate.png は段位ファイルのディレクトリから作る。`G/tests/TaikoDive.Tests/Tests/DanMalformedFileTests.cs`。

## TJA-09 安全性・失敗・非対応

WAVE、MOVIE、段位 TJA の値は Trim した検査値の先頭2文字が `/` または `\` の組合せなら空文字にする。先頭スキーム長が2以上の `://` も拒否。ローカル絶対パスは許可し、許可時は元値を返す。なお通常 TJA 行は先に `//` コメントが除かれるため URI や `//server` が検査前に切断されうる。`G/src/Common/Song/TJA.cs::tネットワークのパスを除く,tStripComment`。

数値不正は通常0にフォールバック、未知ヘッダ・未知命令は無視。BPM表更新の一部例外は握りつぶす。一方、ファイル不在・読取エラー、段位要素不足、引数不足による添字・null 例外はパーサから伝播する。ファイルサイズ・行長・小節分割数・再帰参照の明示上限はこの実装にはない。読込元の Shift_JIS 復号不能バイトの正確な代替文字、ロケール依存数値、異常な入れ子分岐や複数セッションをまたぐ未初期化状態は環境・実行順依存であり、ここでは完全互換の期待値は未確定。一般TJAの全機能、複素スクロール、BMSCROLL、STYLE に基づく選択は実装されていない。失敗時に選曲走査がどの単位で続行するかは呼出側の別章を参照し、パーサ自体が全例外を継続するとはみなさない。
