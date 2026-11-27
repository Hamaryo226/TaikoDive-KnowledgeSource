# 06 描画・音響・素材・3D互換仕様

入力はゲーム G `fb12417` と Assets A `7c5bb71` の固定スナップショット。`G/` は本体ソースのルートを表し、`::` の右は根拠シンボル。[媒体詳細](06b-media-details.md)、[GLB とテーマ](06c-glb-and-themes.md) を併用する。画像・音声・GLB の内容そのものは別途必要である。実物がなければ画素・音色・モデル形状の同一性は検証できない。本章は画面遷移・判定の仕様を定めない。

## MEDIA-01 座標・画像・所有権

描画 API は画像の元矩形 `(x,y,w,h)` を切り出し、指定した9点基準（上中下×左右中央）の点を描画座標に合わせ、各軸 Scale、Rotation、左右反転を適用する。矩形省略時は画像全体。画面側の座標と拡大率に `Program.dbDrawScale` を乗じる。色は RGB 明度、透明度は 0〜255、合成は通常・加算・減算・乗算を区別し、描画後に合成状態を戻す。3Dモデルのフレームには描画時 Y に実装上 10 を加える（コメントの5pxとコードが食い違うためコードを採用）。根拠：`G/src/Core/Rendering/Texture.cs::Draw`。

`Texture` は実画像のハンドルとその非所有共有ビューを区別する。共有元の二重解放を行わず、遅延グループの `tEnsureLoaded`／`tEnsureLoadedAsync`、`tUnload` と `Dispose` の違いを維持する。破棄済み・読み込み未完了・失敗時の描画を安全に扱う。根拠：`G/src/Core/Rendering/Texture.cs::tCreateSharedView`、`tEnsureLoadedAsync`、`tUnload`、`Dispose`。

**受入 MEDIA-01**：9基準点それぞれで同じ矩形の四隅を一致させ、非等方拡大・回転・反転・4合成を比較する。共有ビューを先に破棄しても実体が有効で、所有者の終了時に一度だけ解放されることを確認する。

## MEDIA-02 アニメーション aup2

UTF-8、BOM可。空行と行頭 `;` は無視し、`[section]` と最初の `=` で分けたキー値を読む。最初の `[scene.*]` の `video.width`、`video.height`、`video.rate` のみ使用。`[n]` はオブジェクト、`[n.m]` はエフェクトであり、後者の `effect.name` によりキーの意味を切り替える。`[n]` の `layer` と `frame` を認識し、frame はカンマ区切りの先頭・末尾を開始・終了、中間値を中間キーフレームとする。総フレーム数は素材解決失敗で破棄されたものも含め最大終了値＋1。根拠：`G/src/Core/Rendering/AnimationTexture.cs::AnimationTexture`。

| effect.name | 対応キー | 意味・既定 |
|---|---|---|
| 画像ファイル | ファイル | 画像参照 |
| 動画ファイル／音声ファイル | ファイル、再生位置 | 再生位置の先頭数値を秒として読む |
| グループ制御 | X、Y、拡大率、対象レイヤー数 | X/Y=0、拡大率=100 |
| 音声再生 | 音量 | 先頭数値 |
| 標準描画／映像再生 | X、Y、拡大率、縦横比、中心X、中心Y、Z軸回転、透明度、合成モード、音量 | 拡大率100、他の数値変形は0、音量は先頭数値 |
| 反転 | 上下反転、左右反転 | 輝度・色相・透明度反転は非対応 |
| 拡大率 | 拡大率、X、Y | いずれも既定100 |

パラメータは数値列と移動種別を扱い、直線、瞬間移動、加減速移動を区別する。合成モードは通常、加算、減算、乗算。未認識キーや効果を対応済みと偽らない。[Launcher のプレビュー](../launcher/06b-media-preview.md)は別実装であり、本体の動画・音声やグループ制御を代替しない。`Texture/Common/Control_Guide/Normal`／`XS` の In→Loop、両方なければ Anime 優先・名前順先頭をループ。欠けた場合の静止絵への復帰は呼び出し側が行う。`Texture/Enso/Don/light.aup2` と `kirakira.aup2` は全画面座標で、無ければ描かない。根拠：`G/src/Common/左上太鼓描画.cs::tDrawXS`、`G/src/Common/Chara/MaxEffectAnime.cs`。

**受入 MEDIA-02**：中間キーフレーム・3補間・各合成・動画/音声開始位置を含む実素材でフレーム比較する。未知キーが落下せず無視され、欠落画像を含む全体時間が変わらないことを確認する。

## MEDIA-03 音量・音源・動画

音量はベース値にカテゴリ（Music または効果音）とマスターの百分率を掛け、整数に丸める。式は `round(base × category/100 × master/100)`。設定変更は保持中の該当ハンドルへ反映する。個別音源は再生位置をミリ秒で扱い、同じ実体への再生が互いを打ち切り得るため、1P/2Pの同時再生には別実体を使う。根拠：`G/src/Core/Sound/Sound.cs::ApplyVolumeToHandle`、`Play`、`PlayNoLoop`、`PlayAsMusic`。

標準打音は `Sound/dong.ogg`、`Sound/ka.ogg`。二人用の `Sound/Taiko/Taiko/{dong,ka}_{1p,2p}.ogg` は欠けたら標準に戻すが、それぞれ別 Sound インスタンス。左右パンは1P=-10000、2P=10000。別打音はドン・カの両方が存在するときだけ採用。コンボ音声 `Sound/Combo/{1p,2p}/<数>.wav` は 10、30、50、100〜3000の100刻みの存在するもののみ読み、2pディレクトリがなければ1pから別実体を作る。10・30は簡単／普通のみ。根拠：`G/src/Scenes/Loading/SoundLoad.cs::InitializeHitSoundVariants`、`tCombo`、`GetPlayerHitPan`。

音声ファイル名をOSのANSIコードページで表現できない場合は、ファイルをメモリへ読みピン留めした画像からDxLibへ渡し、ハンドル解放時にピンも解く。動画は読み込み中・無効ハンドルを区別し、再生・一時停止・ミリ秒シークを提供する。停止状態で時刻>0または時刻の逆行を終端として扱い、逆行時は停止する。背景ぼかしは全画面へずらした重ね描き、前景は縦横比を保つインセット。根拠：`G/src/Core/Sound/SoundFile.cs::tLoadSoundMem`、`tDeleteSoundMem`、`G/src/Core/Rendering/Video.cs::TryUpdateMovie`、`DrawBlurredFullScreenWithInset`。

**受入 MEDIA-03**：カテゴリとマスターを変更し再生中の音量も更新されること、Unicodeパス音源の再生と解放、1P/2P同時再生・パン、欠落打音・コンボ音声の代替、動画停止／シーク／終端と無効ハンドルを確認する。

## MEDIA-04 文字と非同期

フォントは `G/src/Core/Rendering/FontRender.cs` と `G/src/Common/FallbackableFontFamily.cs` のフォールバック、縦書き、縁取りと描画キャッシュに従う。フォントファイル・字形・フォールバック先が異なると画素互換を主張できない。非同期の解読と描画資源のアップロードは区別し、完了前に未初期化ハンドルを使わず、破棄後に遅延処理が所有権を復活させない。

**受入 MEDIA-04**：日本語・英数字・欠落字形の横書きと縦書き、縁取り、初回生成と再利用を実フォントで比較し、非同期の途中破棄を検査する。具体的なフォントサイズ・アウトライン生成規則は未抽出。

## MEDIA-05 キャラパス・ぷち・設定

`Info/CharaPath.ini` は既定表を先に読み、同名キーをINIで上書きする。`|` は候補優先順、最初の実在ディレクトリ／ファイルを選び、全欠落なら左端の候補を返す。先頭 `/` または `\` はルートからの相対として扱い、その他はキャラルートとの結合。キーは大小文字を区別しない。未登録キーはキー名そのもの。`Chara_Root` は既定 `Info/Chara/<Type>`、`Chara_AI_Root` は `Info/Chara/AIDON`。`Chara_Config` は `Config.json|Enso/Config.json`、`Chara_ConfigIni` は `Enso/CharaConfig.ini|CharaConfig.ini`。モーションキーと候補は `G/src/Common/Chara/CharaPath.cs::str既定の対応表` を正とする。

選択したぷちは `Texture/PetitChara/<ID>.png`、ID 0 または未配置なら標準シート。`Info/Chara3D.json` の `users.<番号>.petitCharaId` → ルート → 旧設定の順。個別画像は同時2種類まで。取り込み済みの個別素材は透過512×256、256×256の2コマ横並び、1コマ素材は同じ画像を両コマに置く。BPMによる切替と跳躍は既存描画に従う。根拠：`G/src/Common/Chara/PetitCharaSettings.cs::Resolve`、`G/docs/3d-character.md`。

**受入 MEDIA-05**：候補を順番に欠落させた場合の解決、ユーザー別・共通・旧設定の優先順、未知IDの標準復帰、2コマ切替を確認する。

## MEDIA-06 GLB・着せ替え・モーション

プレイヤー共通キャラは3D専用で、AIドン・ぷち・背景・連打効果は独立。`Info/Chara3D.json` の `modelsPath` は build からの相対パス、旧 `enabled` は読み込み条件ではない。必須構成は `animations.glb`、`body/0.glb`、`head/0.glb`、`face/face_000000.png`、任意で `face/animations.json`。モデル・必要なクリップが不足した場合に自動取得せず、エラーを明示する。`parts` の `useCostume`、`head`、`body`、`costume`、`colors` の `body`、`limbs`、`face`、`rim` を読み、色は `#RRGGBB`。全身衣装は頭と胴の組を置き換えるが両選択値を保持。`users` の項目単位で共通に上書きし、同時にユーザーモデルを最大2つ保持する。根拠：`G/docs/3d-character.md`、`G/src/Common/Chara/Model3D/DonModelAppearance.cs`。

GLB 2.0 [1] のすべてを実装する要求ではない。原作の読込範囲は埋め込みバッファ、スキニング、LINEAR／STEPの位置・回転・スケール。モデル側とアニメ側の骨名を対応付け、追加衣装骨は静止姿勢で親骨に追従する。特殊シェーダーや揺れ物物理は再現対象外。範囲外は明示的エラーとする。1280×720 の共有モデル描画先・縁取り描画先に CPU で姿勢を計算し、全外形6px、手足等の内部境界約1.7pxを深度判定付きで描く。パレットは点サンプリング。根拠：`G/src/Common/Chara/Model3D/GlbDocument.cs`、`DonModelData.cs`、`DonModelRenderer.cs`、`G/docs/3d-character.md`。

表情は同一クリップ時刻で切り替え、次キーまで保持する。JSONは `name` と `[秒,表情ID]`、旧形式 `comment` と `[開始,終了,表情ID]`（開始・終了はクリップ千分率）を受ける。衣装顔材質 `extras.shaderType=taikoEffectFace` および `extras.taikoFaceSheet=true` なら埋込みの縦並びシート、それ以外は標準顔。`extras.taikoCullBackFaces`、`taikoSkipOutline`、`taikoOutlineTranslucent` を各々裏面除去・内部縁省略・半透明外縁指定として扱う。風船連打にトラックがない場合は表情ID 7。根拠：`G/src/Common/Chara/Model3D/DonModelFaceMotions.cs`、`DonModelRenderer.cs`、`G/docs/3d-character.md`。

演奏は概ね1/60秒、タイトル等は3.83ms単位のフレームビューを持ち、描画時 `Counter.db端数` でフレーム間の時刻を補間し、クリップ末尾を越えない。端数は描画後にリセット。ユーザーモデルは別スレッドで解析・PNG準備し、メインスレッドで1フレーム2枚ずつアップロード、未完了で必要になれば同期で仕上げる。根拠：`G/src/Core/Rendering/Texture.cs::tコマの途中`、`G/src/Common/Chara/Model3D/DonModelRenderer.cs::tPrepareUserModel`、`tPumpPreparedUsers`。

**受入 MEDIA-06**：標準・頭胴・全身衣装とユーザー別上書きを切替え、骨名対応・未対応データの明示エラー・追加骨の追従、LINEAR/STEP、表情、半透明・深度・輪郭を基準画像で比較する。準備途中の同期仕上げと2ユーザー上限、再起動と終了時の解放を検証する。

## MEDIA-07 他章と検証境界

aup2 の式と合成は [06b](06b-media-details.md)、テーマと GLB の具体式は [06c](06c-glb-and-themes.md)、試聴は [01c](01c-entry-and-options.md)、パスは [10](10-assets-index.md)。[19](19-audio-runtime.md) は音源のゲインと再生同期、[22](22-resource-lifecycle.md) と [24](24-loader-resource-bindings.tsv) は素材の寿命と固定リテラルの初期化表。リテラルから組み立てる変数、曲フォルダーの独自アート、全画面のレイヤー・時刻の組み合わせ、フォントの全画素と実音出力は網羅照合していない。

## Sources

[1] https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html — Khronos glTF 2.0 Specification
