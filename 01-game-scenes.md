# 01｜ゲーム画面と状態遷移

G は固定コミット `fb12417` のゲーム本体。ここは画面横断の概要であり、[01b](01b-flow.md) の曲決定・段位・結果、[01c](01c-entry-and-options.md) の受付・紹介・オプション・検索の詳細を併用する。競合時はより具体的な補足を優先。番号は DxLib キーコード、設定された打撃入力とは割当経路を区別する。出典 `G/相対パス::シンボル` は照合用。実機検証の範囲は [13](13-baseline-verification.md)。

## SCENE-01 Demo

**確認**：デモの候補ディレクトリは Texture/Demo/Animes、次に Texture/00.Demo/Animes。最初に存在する方だけを使い、整数名の子ディレクトリにある Anime.aup2 を番号昇順で再生し、最後から先頭へ循環する。最初のアニメは初回描画時にロードする。読み込めない作品は後続を順番に試し、すべて読めない場合は Title に進む。作品間は終端の完全な黒を描いてから次を読み、1作品のみ保持する。Esc はアプリ終了。出典：`G/src/Scenes/Demo/Demo.cs::tScanAnimes`、`G/src/Scenes/Demo/Demo.cs::tDraw`、`G/src/Scenes/Demo/Demo.cs::tLoadAnimation`。

黒のフェード長は min(30, max(1, 総フレーム数の整数除算3分の1)) フレーム。先頭は黒から透明、末尾は透明から黒へ線形に変え、両方に該当するときは大きい不透明度を採用する。60 fps なら30フレームは0.5秒。コイン投入、フリープレイでのドン、または数字キーによるカードで終了開始。ドンでは打音を鳴らし、カードはユーザーを Title に予約する。終了開始後はアニメを静止、紹介曲を減音しながら500 msで黒へ、Title に移り500 msで黒から明ける。コイン検知はメインループ側なのでフェード中も投入を失わない。出典：`G/src/Scenes/Demo/Demo.cs::tFadeOpacity`、`G/src/Scenes/Demo/Demo.cs::tStartExit`、`G/src/Scenes/Demo/Demo.cs::tEnterTitle`。

### SCENE-01-SHOWCASE 楽曲紹介

**確認**：番号2の作品に限り、楽曲集合があればテンプレートから13枠を読み、主曲、最大4行の副曲、ジャンルやパネル画像を差し替える。枠 i は選曲集合の i を集合数で割った余りを使う。テンプレート上の主曲画像の最小開始フレームを表示開始、対応音声項目の開始・終了フレームを音声区間とし、開始・終了両端を含む。曲名ごとの実フレーム値は素材から決まるため固定値ではない。テンプレートが不適合などの場合は標準の紹介アニメに戻す。出典：`G/src/Scenes/Demo/Demo.cs::tCreateAnimation`、`G/src/Common/Song/DemoSongTemplate.cs::Read`、`G/src/Common/Song/DemoSongTemplate.cs::AudioSlot`、`G/src/Scenes/Demo/DemoSongShowcase.cs::ReplaceImage`。

音声区間に入るとその枠の主曲のみを保持し、DemoStart 秒から再生する。再生開始が区間途中なら経過時間を加算する。DemoStart が既知の音源長以上なら0へ戻し、再生長は区間長と残り音源長の小さい方とする。音量は曲音量設定にプレビュー用ゲインを掛ける。区間外または枠変更時は停止・破棄。素材フレームを f、フレームレートを r、音声開始フレームを a とすると区間経過時間は (f−a)×1000/max(1,r) ms。出典：`G/src/Scenes/Demo/DemoSongShowcase.cs::Update`。

ゲイン曲線と13枠の時刻・曲選択は [01c のデモ紹介](01c-entry-and-options.md)。

## SCENE-02 Title・Entry・モード

**確認**：数字キー1〜9に対応するユーザーのカードを受け付ける。フェードイン中のカードは予約し、明けてから処理する。1P はエントリー画面、2P は1Pがモード選択に入った後に受け付ける。フリープレイではドンでもエントリーでき、そのドンで同時に太鼓・モード決定しない。1P と2Pの太鼓エントリーにはそれぞれ条件があり、2P は1Pエントリー済み・バー表示後・2Pロード中でないことを要する。コイン精算は毎フレーム行い、カードとコインの前後関係や画面切り替えフレームでの投入を許容する。出典：`G/src/Scenes/Title/Title.cs::tDraw`。

太鼓選択は右・つかわない・左の3択で初期値は中央のつかわない。番号32で左方向、37で右方向へ移り端では止まる。番号33または36で決定。右・左なら演奏側を設定し決定演出、つかわないならキャンセル音、エントリー取消と消費コイン返却、カード読み込み・選択演出をリセットする。音声はコイン不足なら入れてね、投入済みまたはフリープレイなら太鼓をたたいてスタートを繰り返し、キャンセル音中は呼びかけを止める。コイン待ちのキャラは暗く表示する。出典：`G/src/Scenes/Title/Title.cs::tDraw`。

モードバー表示完了後のみ移動・決定。番号32/37で範囲内移動、2人エントリーでは通常演奏に固定し他のバーを300 msで消す。決定には精算済みエントリーが1人以上必要で、2Pロード中や決定演出中は受け付けない。モード音声は選択可能かつ設定画面・シーンフェード・2Pロード中でない場合に制御し、案内音声終了後0.5秒かつバーが開いてから鳴らす。Esc は設定画面が閉じているとき終了。番号16は1人目を精算済みとして選曲への近道、20は同様にモード選択への近道。出典：`G/src/Scenes/Title/Title.cs::tDraw`、`G/src/Scenes/Title/ModeVoice.cs::tUpdate`。

表示の基準値としてエントリー案内の2行は (840,468)、(840,644)、名前札は (798,578)。設定案内は (1700,1046) を描画倍率で変換する。出典：`G/src/Scenes/Title/Title.cs::tDraw`。

決済・各モードの左右順・演出期限・2P は [01c の受付](01c-entry-and-options.md)。

## SCENE-03 SelectSong・コース・オプション

**確認**：選曲中の Esc はロード画面開始前かつ設定・検索が閉じている場合に Title のモード選択へ戻る。選曲BGM、選曲ボイス、試聴、ジャンル音声を止め、選曲状態をリセットする。BOX開閉中または設定・検索表示中はバー操作をしない。コース選択が開いている間もバーのキー処理はしない。出典：`G/src/Scenes/SelectSong/SelectSong.Input.cs::bタイトルへ戻る入力を処理する`、`G/src/Scenes/SelectSong/SelectSong.Input.cs::t選曲の入力を処理する`。

番号32・37でそれぞれ左・右へ目標位置を100移動。連続受付期間中に同方向を再入力するとスキップ音と演出が付く。逆方向への切り替えは連続とみなさない。入力優先は左、右、決定、再読込。番号33または36の決定はスクロール位置が0かつ中央バーのカウンタが500以上でのみ有効。SCOREならコース選択、その他はBOX開閉を開始し、ドン・決定演出を鳴らす。番号82は曲再読込。出典：`G/src/Scenes/SelectSong/SelectSong.Input.cs::tKeyProcessing`。

コース選択中、1P未決定なら番号32/37で選択可能な難易度を移動する。決定で位置0は戻る、1はオプション、2以上の選択可能コースは1Pを確定する。裏譜面が存在し切替可能な場合、所定の位置で Tab または右端での右入力10回により表裏を切り替える。番号87は通常・1人・難易度位置2以上のときリプレイ要求モードを巡回。番号64は通常→バトンタッチ→特訓→通常と切り替え、通常以外ではリプレイ再生要求を無効化する。出典：`G/src/Scenes/CourseSelect/CourseSelect.cs::tKeyProcessing`。

オプションの項目順は、ノーツ速度、ドロン、あべこべ、ランダム、演奏スキップ、ノーツ位置、どんちゃん声、打音。開いた直後の入力フレームは無視し、1Pは32/37で値を範囲内で増減、33〜36で次項目へ進む。最後を決定すると保存して閉じる。2PはZ/Vで値変更、X/Cで次項目。出典：`G/src/Scenes/CourseSelect/CourseSelect.cs::tOptionKeyProcessing`、`G/src/Scenes/CourseSelect/CourseSelect.cs::tOptionKeyProcessing2P`、`G/src/Scenes/CourseSelect/CourseSelect.cs::tSaveオプションData`。

試聴・難易度初期値・2P確定・オプション・検索 UI は [01c](01c-entry-and-options.md)。BOX と検索式は [04c](04c-song-catalog.md)。

## SCENE-04 段位一覧・LoadingSong・演奏接続

段位一覧、難易度確定後のロード、演奏接続は [01b](01b-flow.md)。

## SCENE-05 Pause

**確認**：開始時に選択を0に戻しポーズ音を鳴らす。演奏時刻を停止、再生中音声の位置を記憶して音声と動画を停止する。復帰時は演奏時計を再開し、停止前に再生中だった同一音声だけを保存位置から再生する。曲間や未再生・再生終了後の音声を誤って開始しない。譜面時刻が非負なら動画も再開する。出典：`G/src/Scenes/Pause/Pause.cs::tPause`、`G/src/Scenes/Pause/Pause.cs::tResume`。

| 順 | 項目 | 決定後 |
|---|---|---|
| 0 | 演奏を続ける | 上記の位置から復帰 |
| 1 | 初めからやり直す | 譜面を再読み込み |
| 2 | 選曲画面に戻る | 演奏中 Esc と共通の復帰処理 |
| 3 | 演奏オプション | 現在値で編集開始、終了時は適用して譜面を最初から再読み込み |
| 4 | 特訓モードでプレイ | 特訓に切り替え再読み込み。特訓中は「演奏モードでプレイ」と表示し通常へ戻して再読み込み |

番号37で次、32で前、選択は循環する。33・34・36で決定。描画は幅1300、高さ150の5行を画面中央に積み、選択行だけ左へ50移す。F3:閉じる、ドン:決定、K:次、D:前の説明を画面下に表示する。オプションで変更するノーツ速度は音源の再生速度ではない。出典：`G/src/Scenes/Pause/Pause.cs::tHandleMainInput`、`G/src/Scenes/Pause/Pause.cs::DrawMainMenu`、`G/src/Scenes/Pause/Pause.cs::ApplyOptionsAndRestart`。

特訓の独立した停止・再開は [02c](02c-modes.md)。F3 は演奏シーンの先頭で開閉し、オプション編集中でもポーズ全体の Resume を呼ぶ。メインポーズ中では K、D を独立に処理（同時なら双方順にカーソルを巡回させる）、F/G/J が決定。ポーズ時の通常ノーツ処理は停止する。根拠：`G/src/Scenes/Game/NotePlay.cs` 546–564、`G/src/Scenes/Pause/Pause.cs` 210–256。実機では [08](08-conformance.md) に従って比較する。

## SCENE-06 通常・AI・段位 Result

**確認**：通常結果は数値演出カウンタ8230以上で入力可能。番号33〜36のドンで退出フェード開始、2000カウンタ単位で黒へ遷移し、結果BGMを減音・停止、リプレイ状態と演奏・選曲・結果をリセットして選曲に戻る。AIでは同じ入力可能期間中に32または37で区間勝敗パネルと区間別内訳を切り替え、カッ音を鳴らす。出典：`G/src/Scenes/Result/Result.cs::tDraw`、`G/src/Scenes/Result/Result.cs::tRecovery`。

段位結果はメインカウンタ11980以上、かつゲージ演出17が終端に達するまで退出不可。その後、番号32〜37の押下または押し続けを受け付けてドン音、演出18を7000から開始。7000〜9000で結果音を減衰、黒を重ねて11000到達時に演奏・選曲・結果状態をリセットし段位選択へ戻る。上部に試験ヘッダを (352,30)、名前札を (58,933) に描く。出典：`G/src/Scenes/Result/DanResult.cs::tDraw`。

通常・AI・段位の詳細は [01b](01b-flow.md)、王冠とランク閾値は [02b](02b-scoring.md)。

## SCENE-07 SettingsOverlay

**確認**：初回に設定ページからタブを作り、再オープン時は前回のタブを選ぶ。タブ選択・ページ移動時は先頭項目、スクロール先頭に戻し、ドロップダウンを閉じる。開いている間だけマウスカーソルを表示し、閉じると元の表示状態に戻す。設定入力の優先順位はキー割当待ち、ドロップダウン、通常のマウス、通常のキーボード。出典：`G/src/Scenes/Settings/SettingsOverlay.cs::tOpen`、`G/src/Scenes/Settings/SettingsOverlay.cs::tSelectTab`、`G/src/Scenes/Settings/SettingsOverlay.cs::tHandleInput`。

Tab/Shift+Tab は次/前タブ、上下は選択を循環、PageUp/PageDown は表示行数分の移動、左右は値変更、Enter は決定、Esc または右クリックは上のページへ戻り、ルートなら閉じる。ドロップダウン中は Esc/右クリックで一覧を閉じ、上下で選択、Enterで決定する。クリックはタブ・行・閉じるボタンに作用し、パネル外クリックも閉じる。スライダーはドラッグ中に値を更新し、離したとき一度保存する。ホイールは値変更可能な行の上なら値を変更、それ以外は一覧選択をスクロール。マウスが移動した場合だけホバーで選択を奪う。キー割当待ちでは Esc で取消、Delete で割当を消去する。出典：`G/src/Scenes/Settings/SettingsOverlay.cs::tHandleKeyboard`、`G/src/Scenes/Settings/SettingsOverlay.cs::tHandleMouse`、`G/src/Scenes/Settings/SettingsOverlay.cs::tHandleDropdown`、`G/src/Scenes/Settings/SettingsOverlay.cs::tHandleKeyCapture`。

F4 の入力優先と項目順は [01c](01c-entry-and-options.md)、選択肢と値域は [04d](04d-settings-and-atomic.md)。

## SCENE-08 受入ケース（静的仕様から導出、未実行）

| ID | 前提・操作 | 期待結果 |
|---|---|---|
| SCENE-AC-01 | Demo、フリープレイでドン | ドン音、静止画を500 msで黒へ、Titleで500 msフェードイン |
| SCENE-AC-02 | Demo再生中にカード、同一フレームでコイン | 両方を保持しTitleでカードの処理とコイン精算が可能 |
| SCENE-AC-03 | 太鼓選択で初期中央を決定 | キャンセル音、1Pエントリー取消、消費コイン返却 |
| SCENE-AC-04 | 2人エントリー後にモード左右移動 | 通常演奏固定、他バーが消える |
| SCENE-AC-05 | 選曲バー移動中に決定 | 停止・中央演出の条件を満たすまでコース選択しない |
| SCENE-AC-06 | Pause中、曲間で再開 | 未再生の音声を勝手に開始しない |
| SCENE-AC-07 | Pauseからオプションを最後まで決定 | 値を適用し現在位置からではなく最初から再読み込み |
| SCENE-AC-08 | AI結果の入力可能時に縁 | 区間勝敗と内訳を切り替え、選曲へは戻らない |
| SCENE-AC-09 | 段位結果で条件成立前に打つ | 退出しない。成立後は押し続けも受け付ける |
| SCENE-AC-10 | 設定でドロップダウン中にEsc | 設定画面全体ではなくドロップダウンだけ閉じる |

## SCENE-09 実測の境界

固定版の起動条件と開始シーンは [13](13-baseline-verification.md)。起動後の全経路や全フレームの描画・実音は実機比較が必要。
