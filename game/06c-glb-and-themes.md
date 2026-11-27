# 06c｜GLB 姿勢計算とテーマの差し替え契約

## MEDIA-C01 GLB の読み込み・データ境界

TaikoDive は汎用 glTF 実装でなく、同梱モデルに必要なサブセット。次の値・順序で別ライブラリの出力を合わせる。

| 要素 | 読み方 |
|---|---|
| コンテナ | リトルエンディアンの 32bit magic `46546C67`（ファイル先頭の ASCII は `glTF`）、バージョン 2、宣言長はファイル長と一致。その後「長さ32bit、種類32bit、値」の繰返し。型 `4E4F534A` は UTF‑8 JSON、`004E4942` は BIN。範囲超過を拒否、未知型は無視。どちらも必須。同種チャンクを複数読んだ場合は後勝ち |
| bufferView | 埋込 BIN の buffer=0 のみ。`byteOffset` 省略は0、`byteLength` を使う。負数や末端越え不可。外部 URI を開かない |
| accessor | `SCALAR/VEC2/VEC3/VEC4/MAT4` で幅 1/2/3/4/16。型番号 5121=U8、5123=U16、5125=U32、5126=F32。配列は浮動小数点32bitで返す。float は正規化しない、`normalized:true` の整数は 255 / 65535 / 4294967295 の型最大値で割る。`byteStride` 既定は width×bytes、accessor 側 `byteOffset` 既定 0。負 count、短い stride、負 offset、`offset + max(0,count−1)×stride + width×bytes` がビューを超える場合、sparse は拒否 |
| ノード | translation=(0,0,0)、scale=(1,1,1)、rotation=(0,0,0,1) を既定とし、指定回転は `(x,y,z,w)` を正規化。子リストで親を設定し、不正 index・重複親・循環を拒否。ノードの matrix を TRS の代替としては読まない |
| クリップ | `animations.glb` を基準に名前完全一致。各 channel は rotation/translation/scale のみ。sampler は既定 LINEAR または STEP。時刻と値は同じ非ゼロ長。全 track の末尾時刻の最大が clip duration。通常モデルを差し替えてもこのモーション辞書を共有 |
| メッシュ | `useCostume` 真なら `cos/<id>.glb`、偽は `body/<id>.glb`→`head/<id>.glb`。同名骨は基準骨を使い、衣装固有ノードを追加し親 index を写像。メッシュがあるノードの skin の joints はその写像で解決。inverseBindMatrices 必須。mode は既定 4（triangle）、それ以外不可。頂点数は65535以下。index は unsigned 16 へ収まる値で頂点配列内 |
| 必須属性・画像 | `POSITION`、`TEXCOORD_0`、`WEIGHTS_0`、`JOINTS_0`、indices、material、`pbrMetallicRoughness.baseColorTexture.index`→texture.source→image.bufferView。テクスチャを URL から取得しない。mesh ごとの position/scale/rotation を皮膜行列に重ねない |

根拠：`G/src/Common/Chara/Model3D/GlbDocument.cs` 13–66、`DonModelData.cs` 83–129、132–257。表中のファイル表記は実行時基準。一般 glTF の未対応機能を勝手に有効化すると互換性検査のエラー条件が変わる。

## MEDIA-C02 姿勢と画面への投影

毎回、全ノードを静止 TRS に戻し、要求時刻を [0, duration] に制限してトラックを適用。サンプル時刻完全一致ならその値、前は最初、後は最後。区間中 STEP は直前値。LINEAR は `u=(t−a)/(b−a)` で位置・尺度を `Va+(Vb−Va)u`、回転は正規化四元数の球面線形補間。ローカル行列は行ベクトル規約で `Scale × Rotation × Translation`、グローバルは `Local × World(parent)`。親が配列の後ろにいても親を先に評価する。列ベクトル規約へ移すなら全体を転置相当で順序反転し、部分的な掛け順変更をしない。

関節の皮膜行列は `InverseBind × World(joint)`。1頂点につき最初の4重みを読み、正値項だけ変換した頂点×その重みを足す。重みを勝手に再正規化しない。姿勢後を Y 軸回りに通常 +27.5°、反対側 −27.5° 回転して別方向から見せる（画面を単純に反転しない）。回転後座標を `(640+1100X, 550−1100Y, 0.5−Z)` へ投影。UV はそのまま、垂直反転フラグのパーツだけ `v=1−v`。向き、clip、時刻がすべて同一なら描画先を再利用。

通常は深度検査・書込み、普通の面は alpha≥254、顔は後回しで alpha>0 かつ深度を書かない。背面除去指定は投影後の画面 XY 外積の符号が負の三角だけ残す。色パレットは最近傍、それ以外バイリニア。repeat は UV を頂点毎に剰余で変更せずサンプラー。境界輪郭を描き、半透明部（該当材のみ）を深度非書込で重ねる。輪郭用バッファではモデル全体の alpha を広げて外形を作り、内部境界と別に扱う。この章での 6px / 約1.7px は元の 1280×720 描画先基準。合成後に呼び出し元の描画先、合成、アルファテスト、描画範囲を戻す。

`extras.shaderType=taikoEffectFace` が顔。`taikoFaceSheet=true` の顔はモデル内シート、そうでない顔面のXY最大辺を 0.137 にする（衣装では拡大しない）、中心を維持。`taikoEffectChangeColors` は色分け。三角形の3頂点の手足重み合計が 1.5 を超えると手足側、それ以外は別パーツ。同じ青マスクを部位で別の色へ割り当てる。`arcadeFaceRadiusRatio` の存在は専用面、`taikoCullBackFaces`、`taikoSkipOutline`、`taikoOutlineTranslucent` は既定偽。深度の前後関係を保ったうえでテクスチャ合成する。

根拠：`G/src/Common/Chara/Model3D/DonModelData.cs::Evaluate,LoadPart,Track.Sample`、`DonModelRenderer.cs::Render` 342–453。顔面曲率補正やパーツ個別材質の外観までを、この数式だけで合格とはしない。

## MEDIA-C03 クリップの対応

| 状態 | Clip |
|---|---|
| 演奏・通常 / クリアループ / クリア突入 / 魂到達 | `don_normal` / `don_norm_loop` / `don_norm_up` / `don_full_gage` |
| ゴーゴーループ / 開始 | `don_sabi` / `don_sabi_start` |
| 10コンボ / MAX時の10コンボ | `don_combo` / `don_full_combo` |
| 風船継続 / 成功 / 未達 | `don_balloon_loop` / `don_balloon_success` / `don_balloon_failure` |
| 共通常時 / クリアループ / クリア突入 | `don_general_loop` / `don_select_loop` / `don_general_jump` |
| 失敗突入 / ループ | `don_result_failure` / `don_result_failure_loop` |
| 大成功突入 / ループ | `don_full_combo` / `don_result_full_loop` |
| 曲開始 / AI曲開始 / スキップ | `don_combo` / `don_combo` / `don_select_paneldown` |
| 選曲クリアループ特例 / 選曲エントリー / 結果クリア突入 | `don_result_clear_loop` / `don_entry_loop` / `don_result_clear` |

G のフレームビューは `count=max(2,ceil(duration/interval))`、interval は共通0.00383 s、演奏は 1/60 s、各 index `i` のサンプル時刻は `duration×i/(count−1)`。ゴーゴーの演奏カウンターは `60/BPM×拍数×2×10⁶/count` µs。通常より長い `don_sabi` を1周期へ圧縮しない。ユーザー別テクスチャは共通モーションを複製しないが姿勢領域は別。

根拠：`G/src/Common/Chara/Model3D/DonModelMotions.cs`、`G/src/Scenes/Loading/TextureLoad.cs::tLoad3Dキャラ`。

## MEDIA-C04 置換可能な背景・演出

以下は `Texture/` 下のレイアウト。画像のみ、または `.aup2` と参照ファイルの組を用意する。詳細な資源名は `11-asset-inventory.tsv` と `12-static-resource-paths.tsv`。

| 機能 | 置き場と制御 |
|---|---|
| タイトル背景 | `Entry/Bg` の `.aup2` をループ。フォルダーなし/未準備なら従来の2枚を重ねる。背景アニメとモードの前景を独立させる |
| 操作ガイド | `Common/Control_Guide` の導入 In→Loop、`StartIntro` で入れ直し、`CompleteIntro` は導入を飛ばして Loop 先頭、透明度を全体に掛ける。小サイズ表示は静止スプライト経路と区別 |
| 分岐変更 | `Enso/Taiko/Branch`。幅370×高さ144の対応素材の上レーン位置 (239,180)、下 (239,756)。元の太鼓の右端499から75左、高さ位置は上端276・下端804から計算。下は上下関係を逆にする |
| 上背景 | `Enso/Bg_up`。曲ロード時に抽選・準備し、コラボ用の選択は下表に従う |
| 下背景 | `Enso/Bg_down`。通常 `1`〜`5`、クリアは `c/1` 松雲／`c/2` 招き猫ダルマ／`c/0` の `.aup2` 富士山。コラボ用の選択・クリア遷移は下記 |
| 通常ダンサー | `Enso/Dancer` の `MainConfig.json`: `dancerShowCount`、`dancerTypeLock`（−1/無効フォルダーなら曲ごと抽選）、`dancerGaugePercent`（閾値）、`songGenreID`（`default` は制限なし）。種別の `Config.json`: `in/loop/outMotionMeasureCount`（4拍を1小節）、`in/loop/outMotionBasePoint`（3×3点の0〜8）、`dancerIn/Loop/OutX` と Y、同 MotionIndex（番号PNG順）、`useSameImageForInOut` と `useSameMotionForInOut`（Out不足時はIn絵で、後者真なら入場逆回し／偽ならOut順）。aup2 があるモーションの連番指定は不要 |
| コラボダンサー | `Enso/Dancer/<コラボ>`。Config の `dancerGaugePercent` 既定は等分、`dancerOrder` は左から順、`dancerWidth=384`、`dancerEdgeInset=96`（半間隔まで）、`dancerInnerInset=32`、`dancerX=0,dancerY=540`、`inMotionMeasureCount=0,loopMotionMeasureCount=2.5`（0は素材FPS）。5人のXは中心±352、中心±672と中心、枠の順序による。ゲージで In↔逆再生の向きだけ反転し、到達で Loop、未到達で In の逆回しへ |
| 演奏終了 | `Enso/End` の `Failure`/`Clear`/`FullCombo`/`DondaFullCombo`。クリア未達、ゲージ到達かつ不可あり、不可ゼロ、可不可ゼロを区別。最初の2種は曲ロードで開始、後者2種は解析だけ前倒し、最後の判定後に該当テクスチャを読む。最初の絵が揃ってから終了音。終了暗転までクリア6.0s、フルコンボ6.0s、全良7.5s、失敗7.0s。結果または選曲で解放。特訓では読まない |

### コラボ背景

`CollaboBack` 有効時に曲ロードで選び、`Enso/Bg_up/<ID>` と `Enso/Bg_down/<ID>` 以下を参照する。コラボ背景には、上下を個別に準備する従来型と、上下を一組として確定する型がある。従来型の副題キーは表の上から最初に該当するものを採用し、特記以外は大文字小文字を区別する。`STAGE:None` は従来型も抑止する。

| ID | 選択条件・型 | 構成上の違い |
|---|---|---|
| `MIKU` | 副題に `初音ミク` または `miku`。従来型 | 上は通常／クリアのクロスフェード。下は `drawNormalUnderClear=true` の素材構成を利用可 |
| `GUMI` / `IA` | 副題に `GUMI`、次に `IA`。従来型 | 従来の固定画像 |
| `TOHO` | 副題に `東方`。従来型 | 上は通常／クリアのクロスフェード。下のアニメ読込中は従来の1枚絵を使用 |
| `LOVELIVE` | 副題に `ラブライブ` または `Aqours`（後者は大小文字を無視）。従来型 | 専用素材のないフッター・モブ・ダンサーは通常のもの |
| `GAKUMAS` | 学マス。G2/A2 以降の上下セット型で、1人の通常演奏のみ。`STAGE` はトリム・NFKC 後に `GAKUMAS`（大小文字無視）、`学園アイドルマスター` または `学マス` に一致する場合のみ候補。タグが空のときだけ、TITLE・SUBTITLE・GENRE または最初の `Songs` 配下のディレクトリ名で後二者を部分一致検索（いずれも NFKC、譜面ファイル名は除外） | 両帯に `Normal/In/Loop/Out`。上は `Palette.aup2` と `Decor`（`00-GradientLight`、`10-Grid`、`20-Mascots`）。下は G2/A2 で `Motion.aup2`＋動画、G3/A3 で `Stage.aup2`＋透過PNG。下にダンサー等を含むため、通常ダンサー・フッター・モブを重ねない |

- **選択とフォールバック**：上下セット型は従来型より先に候補を評価し、必須シーンと全参照素材が正常に読み込めたときにだけ採用する。片側の欠落や失敗で上下を混在させず、部分確保を解放して従来のコラボ／通常背景の選択を続ける。G2/A2 では同じ状態に列挙された追加シーンも同期検証する。G3/A3 では必要なシーンだけを採用し、旧ファイルを無視する。画像専用ローダーは対象シーンに動画参照があれば画像ロード前に拒否する。従来型のアニメ資源は非同期で、専用素材がないパーツには通常素材を使う。
- **配置と順序**：上下セット型の上帯は1920×276で `(0,0)`、下帯は1920×540で `(0,540)`。上→下、帯ごとに状態→装飾。状態ごとの `.aup2` は状態フォルダー直下を使い、なければタイプ直下の状態名接頭辞で探す。重ね順はファイル名の序数昇順・大小文字無視。上下が揃った同一フレームに開始し、1P のゲージがコースのクリア閾値以上かどうかを両帯に渡す。状態ごとの尺や装飾の周期は独立。曲固有動画はロード完了後に優先し、ロード中は採用済みコラボ背景も描画し得る。
- **クリア遷移**：下背景アニメと上下セット型は `Normal → In → Loop`。In 中に閾値を下回れば現在位置から逆再生し、再到達で正方向に戻す。Loop 中に下回れば Out（なければ In 末から逆再生）。Out 中に再到達なら In を先頭から開始。逆再生が先頭、Out が終端に達すると通常へ戻る。同フレームで閾値を跨いだ場合は状態変更を終端判定より優先する。In が任意の従来型では、In がなければ最初から Loop、Loop がなければ In の末尾を表示する。In/Out の終端は最長レイヤーの総フレーム数 N、1刻みは先頭レイヤーのFPSに対する `1000000/fps` μs。描画位置は `clamp(k + 刻み内の経過割合, 0, N)`、逆方向では N から引く。反転時は `N−k` へ整数で移し、端数を持ち越さない。短いレイヤーの表示判定は各自の最終有効コマまで。`Config.json:drawNormalUnderClear`（既定偽）が真なら Normal を下で回し続けて復帰時も位相を維持。偽なら復帰時に再始動。上下セット型はリトライで両帯を Normal に戻し、通常レイヤーと装飾をそれぞれ先頭から再生する。

根拠：`G/docs/bg-up-folder.md`、`bg-down-folder.md` と `BackUp`、`BackDownAnime`、`BackUpAnime`。上下セット型は G2/G3 の `BackUp`、`BackDownAnime`、`GakumasStage`、`AnimationTexture`、`NotePlay`、`ResetGame`。[検証記録](26-r2-verification.md) と [構成比較](27-architecture-tradeoffs.md) の固定版を参照。受入は各タイプについて、選択条件、素材欠落、曲固有動画との優先順位、ゲージ境界、リトライを比較する。

描画原点表では画面 Y が下へ増加する。`15-render-coordinates.tsv` は G の数値リテラルのある描画呼出しの座標と切出し候補、`16-animation-counters.tsv` は時間カウンターの固定オペランド。動的値やレイヤーが重なる場合は媒体・シーン章の適用式を優先。`動的` はゼロや欠損を意味しない。根拠：`G/docs/dancer-folder.md`、`dancer-collabo-folder.md`、`end-anime-folder.md`、`control-guide-folder.md`、`branch-anime-folder.md` と G の対応する描画シンボル。演出の全フレームの画素同等性は別途比較する。
