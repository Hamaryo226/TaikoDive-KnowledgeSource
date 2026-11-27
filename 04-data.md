# 04｜設定とデータ交換の基本契約

対象は固定コミット G `fb12417`、L `a1267f0`。この章は共有形式の入口。ファイルごとの検証・型・変換・書込み順は [リプレイとユーザー](04b-replay-users.md)、[曲カタログ](04c-song-catalog.md)、[設定と原子置換](04d-settings-and-atomic.md) を併用する。競合時は具体的な補足を優先する。

## DATA-01 設定の形式と既定値

ゲームは作業ディレクトリの `Setting.json` を読む。無ければ新規既定値を保存し、読み込み直後にも保存する。JSON は UTF-8、camelCase の公開フィールド、enum は文字列、整形あり。既知型への再シリアライズなので未知キーを保持する契約ではない。根拠: `G/src/Common/Program.cs::Main`、`G/src/Common/Config/ConfigManager.cs::GetConfig,SaveConfig`、`G/src/Common/Config/ConfigInI.cs::ConfigInI`。

| JSON キー | 型 | 新規既定 |
|---|---|---|
| guestMode | bool | true |
| b2PMode | bool | false |
| fullScreen, borderlessWindow | bool | false, false |
| fullHD, verticalSync | bool | true, true |
| screenWidth | int | 0（未指定なら fullHD により決定） |
| frameRateLimit | int | 0（無制限） |
| titleShow, collaboBack, saveBestReplay | bool | true, false, true |
| detailedResult, freePlay | bool | false, false |
| masterVolume, musicVolume, soundEffectVolume | int | 100, 100, 100 |
| backgroundMovieLayoutMode | enum文字列 | FullScreen（他に BlurredWithInset） |
| soundType | enum文字列 | DirectSound（他に Wasapi, WasapiExclusive, ASIO） |
| soundBufferSamples | int | 0（ドライバ任せ） |
| fontOedo | string | FOT-大江戸勘亭流 Std E |
| fontDFGothic | string | ＤＦ太丸ゴシック体 Pro-5 |
| fontSeurat | string | FOT-スーラ Pro B |
| fontDomCasual | string | Dom Casual |
| fontFallback | string | Comic Sans MS |
| p1Keys, p2Keys | object | 下記 |
| reduceTextureColorTo16bit, reduceBgTextureColorTo16bit | bool | false, false |
| useCompressedSongSound | bool | true |

`bTwoPlayerMode` はゲーム内部の別名で JSON には書かない。キーオブジェクトの `kaLeft`, `donLeft`, `donRight`, `kaRight` は整数配列。1P の既定は順に `[32]`, `[33,34]`, `[35,36]`, `[37]`、2P は `[44]`, `[45]`, `[46]`, `[47]`。対応する `kaLeftControllers`, `donLeftControllers`, `donRightControllers`, `kaRightControllers` は各々空配列。要素は `vendorId`, `productId`（ushort）、`deviceIndex`, `deviceOrdinal`, `inputIndex`（int、既定0）、`deviceName`（文字列、既定空）、`inputType`（Button または Switch、既定 Button）、`inputValue`（int、既定1）。VID/PID と同型機器内順で共有する。根拠: `G/src/Common/Config/ConfigInI.cs::KeyBindConfig,ControllerInputBinding`。

Launcher は設定保存前にゲーム実行中なら拒否する。既存 JSON をオブジェクトとして読んで対象キーのみ更新し、それ以外は保持するが、廃止済み `charaAnimationFrameSkip` と `reduceCharaTextureColorTo16bit` は大小文字無視で削除する。音量を 0–100、保存幅を 640–7680、バッファを 0 または 32–8192 に制限し、`fullHD` は幅が1920以上かで更新する。ゲーム側の許容値と Launcher の保存制限を混同しない。JSON が不正・非オブジェクトなら保存失敗。根拠: `L/Services/GameSettingsStore.cs::LoadAsync,SaveAsync`。

## DATA-02 スコアとパス

基準はゲーム実行ディレクトリ。譜面フルパスを Windows 区切りに揃え、実行ファイル基底ディレクトリ内ならその接頭辞を外し、外部ならフルパスを使う。これを invariant 大文字化し UTF-8 バイト列の SHA-256 を大文字16進にしたものが曲キー。外部曲はゲーム一式の移動後も同じキーになるとは限らない。ユーザー名は Windows ファイル名禁止文字を `_` に置換、空白のみなら `Default`。同じ正規化結果の名前同士の衝突に注意。根拠: `G/src/Common/Score/ScorePath.cs::tGetSongKey,tGetRelativeSongPath,tSanitizeUserName`。

| 内容 | パス |
|---|---|
| 曲別成績 | `Info/ScoreData/<ユーザー>/<曲キー>/Score.dat` |
| リプレイ | 同曲フォルダ内 `Replay/course-<難易度>/best.json` |
| 曲の覚書 | 同曲フォルダ内 `Song.txt`（UTF-8 BOM、曲名と正規化パスを各1行、既存なら更新しない） |
| 最近の曲 | `Info/ScoreData/<ユーザー>/Recent.json` |

旧 `Info/ScoreData/<ユーザー>/<曲名>.dat` は新規パスが無いとき探索しコピー、元は残す。禁止文字を置換した曲名も探索し、旧ユーザー名の未変換フォルダも見る。同名曲への曖昧なコピーがあり得る。旧 `Info/Replay/<ユーザー>/<旧キー>` は新 `Replay` が無いとき移動する。旧キーはフルパスを Windows 区切り・大文字化した UTF-8 の SHA-256。根拠: `G/src/Common/Score/ScorePath.cs::tTryMigrateLegacyScore,tTryMigrateLegacyReplay,tGetLegacySongKey`。

`Score.dat` は名前に反して JSON ではなく Shift_JIS の独自テキスト。難易度 Easy, Normal, Hard, Oni, Edit の順に `Score`, `Gauge`, `Great`, `Good`, `Miss`, `RollCount`, `MaxCombo`, `Crown`, `ScoreRank`, `AIWin`, `AIScore`, `AIWardCrown` を `名前=値` で保存する。通常スコアは新規または厳密に高い場合のみ、ゲージと判定内訳などをまとめて差し替える。同点は差し替えない。王冠とランクはスコア更新とは独立に向上可能。AI 勝利回数は勝利時に増加し、AI 自己ベストと区間王冠は厳密に高い場合だけ置換。区間王冠は S 銀、G 金、R 虹、- 音符なし。根拠: `G/src/Common/Score/ScoreDat.cs::tWriteDatFile,tSaveDatFile,tSaveAIResult`。

## DATA-03 リプレイ

**この版で確認したリプレイはバイナリではなく JSON**。`best.json` は `ConfigManager` により UTF-8・camelCase・enum 文字列で書かれる。したがって固定長レコードやバイトオフセットを作ると互換にならない。別のバイナリ形式の存在は未検証。現行 `version` は 2。主なフィールドは `songPath`, `songKey`, `chartHash`, `songTitle`, `course`, `recordedAtUtc`, `playerName`, `playerTitle`, `namePlateType`, `charaType`, `options`, `inputs`, `judgements`, `result`。後から追加されたプレイヤー装飾フィールドのない version 2 も読める。根拠: `G/src/Common/Replay/ReplayManager.cs::ReplayData,TryLoadBest`、`G/src/Common/Config/ConfigManager.cs::SaveConfig`。

`inputs` 要素は `time`（long）, `hand`（int）。`judgements` は `time`（long）, `chipNumber`（int）, `branch`, `note`, `judgement`（enum文字列）, `timeLag`（int）, `isDon`, `isBigNote`（bool）。`result` は `score`, `great`, `good`, `bad`, `roll`, `maxCombo`（int）, `gauge`（double）。`options` は `speed`（double、既定1）, `doron`, `reverse`, `skip`（bool）, `random`（enum）, `notesPosition`（double）, `donChanVoice`（bool、既定true）, `hitSound`（int）。根拠: `G/src/Common/Replay/ReplayManager.cs::ReplayInputEvent,ReplayJudgementEvent,ReplayResultData,ReplayOptionData`。

記録開始条件は設定の保存有効かつリプレイ再生・オート使用・ゲスト・永続2Pのいずれでもないこと。セッション終了時には保持していた2P設定と1P/2Pオプションを復元する。best 更新・時刻・互換条件・中断は [04b](04b-replay-users.md)。根拠: `G/src/Common/Replay/ReplayManager.cs::PrepareForPlay,CompleteRecording,EndSessionAndRestoreMode`。

## DATA-04 履歴と Launcher ブリッジ

段位履歴のファイル名は `DanHistory.jsonl`。1行1 JSON、enum は文字列。レコードは `Version=1`, `PlayedAt`, `DanPath`, `MainTitle`, `DanType`, `Gauge`, `GaugeRed`, `GaugeGold`, `Passed`, `GoldPassed`, `StartedSongCount`, `FailedDuringPlay`, `DisplayRank`, `Total`, `Songs`, `Conditions` を持つ。成績は Score, Great, Good, Bad, Roll, Hit, MaxCombo、曲は Tja, Title, Course, Result、条件は ExamNumber, SongNumber（nullable）, Type, Range, Red, Gold, Actual, Passed, GoldPassed。JSON の大文字小文字と保存条件・パスは [04b](04b-replay-users.md)。根拠: `G/src/Common/Score/DanPlayHistory.cs::Record,Stats,Song,Condition,Append`。

Launcher 向け履歴は別物。`Info/TaikoDiveLauncher/history/<id>.json` に、`id`, `playedAtUtc`, `playerName`, `songTitle`, `course`, `score` を保存する。演奏ごとの GUID とプレイヤー番号で ID を作り、既存なら重複保存しない。履歴保存は非同期で I/O 失敗を握りつぶす。設定 `Info/TaikoDiveLauncher/integrations.json` の既定は version 1、discordEnabled=false、discordShowSong=false、overlayEnabled=false、recordHistory=true、goal 空。16,384 バイトを超える設定、version 不一致等は既定値に戻す。根拠: `G/src/Common/LauncherBridge.cs::Options,Initialize,RecordPlay`。

Launcher リプレイ起動引数は `--launcher-replay <path> --user-slot <1..9>` の4個のみ。パスは `Info/ScoreData` 以下の既存 `best.json`、最大32 MiB。version 2、course 0–4、譜面実在、選択ユーザーの正規保存先、譜面ハッシュ、現在の曲ライブラリを検証し、失敗時は拒否する。起動中はゲストと2Pを無効化し通常演奏モードへ移る。これらを永続設定と混同しない。根拠: `G/src/Common/LauncherBridge.cs::ParseArguments,TryStartReplay`。

## DATA-05 曲ライブラリ・順序・最近の曲

Launcher の曲順ファイルは各ジャンル直下 `.taikodive-order`。UTF-8 BOM なし、ジャンルからの相対 TJA パスを `/` 区切りで1行ずつ。読み込みは大小文字を区別せず初出を採用、未指定曲は列挙順を維持して末尾。保存はゲーム実行中に拒否し、ジャンルが現在の Songs 直下であること、指定集合に重複がなく現在の再帰列挙集合と一致することを確認する。書込みは原子的置換とバックアップを使う。根拠: `L/Services/SongOrderService.cs::LoadSongs,SaveOrderAsync,SaveOrderWithoutProcessCheckAsync`。

Launcher ライブラリ走査は `.tja` を大小文字無視で列挙し、タイトルの文化依存・大小文字無視順にする。ファイルサイズ 8 MiB 超は問題として扱う。UTF-8 厳格デコードを試し失敗時 CP932、BOM は除去。コメント `//` を除いたヘッダから TITLE、WAVE、SUBTITLE、GENRE、DEMOSTART を読み、WAVE が Songs 外なら拒否、無いファイルを警告する。GENRE 未指定なら祖先の `box.def` から `#GENRE:` を探す。これと曲順編集時のタイトル抽出は別実装で、後者は先頭512行・1行16 Ki文字まで、`TITLEJA` 優先、`#START` で終了する。根拠: `L/Services/SongLibraryService.cs::ScanAsync,ReadChart,ReadFolderGenre`、`L/Services/SongOrderService.cs::ReadTitle,ReadHeaderLine`。

最近の曲は最大30件、新しいものが先。存在しない・移動した譜面は表示時に飛ばし、現在の曲とパスを大小文字無視で照合する。根拠: `G/src/Common/Song/RecentSongs.cs::tLoad,tBuild`。

## DATA-06 保存と競合

ゲームの一般 JSON と成績は `AtomicFile` を使う。Launcher 設定は既存 JSON に部分更新して保存し、ゲーム起動中の同時書換えを拒否する。Launcher のブリッジ用出力は同ディレクトリに GUID 付き一時ファイルを書いて上書き移動し、一時ファイルを finally で削除する。原子的置換は複数プロセス間の read-modify-write の排他そのものではない。バックアップ世代や障害時は [04d](04d-settings-and-atomic.md)。根拠: `G/src/Common/Config/ConfigManager.cs::SaveConfig`、`G/src/Common/Score/ScoreDat.cs::tWriteDatFile`、`G/src/Common/LauncherBridge.cs::WriteAtomic`、`L/Services/GameSettingsStore.cs::SaveAsync`。

## DATA-07 確認テストと残作業

次を実データを使わない一時ディレクトリ上で確認する。現時点では実行していない。

1. 空設定の生成、未知キーを持つ設定のゲーム保存と Launcher 保存、壊れた JSON、ゲーム実行中の保存拒否を比較する。
2. 譜面の相対・外部パス、異なる大文字小文字、同名曲、禁止文字を含むユーザー名について曲キーと移行元の残存を確認する。
3. 新記録・同点・低得点でスコア内訳と王冠が独立して更新されること、AI自己ベストと勝利数が混ざらないことを確認する。
4. version 2 の装飾フィールド欠落、譜面変更、別スロット、32 MiB 超のリプレイ、および再生中断後の設定復元を確認する。
5. 並べ替えファイルの重複、欠落、曲追加、ゲーム実行中の拒否、UTF-8/CP932 譜面、履歴重複を確認する。

別OSでのファイルシステム、旧データの全組合せ、複数の書込み競合や異常終了後の復旧は [08](08-conformance.md) と [14](14-coverage.md) に従い検証する。
