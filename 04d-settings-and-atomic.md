# 04d｜設定の境界値・原子置換・Score.dat の外形

## DATA-D01 設定の既定・正規化

ゲームの起動では `ConfigInI` という型名にかかわらず、設定の入出力は `Setting.json`。本体起動の経路に旧 `Setting.ini` の読み込み・変換はない。設定がなければ既定オブジェクトを書いてから読む。JSON.NET UTF-8 で読み、書くときは UTF-8 BOM・整形・既知フィールドの camelCase・enum 名文字列。欠けた既知キーはインスタンスの既定に残る。不正な JSON に対して安全な既定値へ黙ってリセットはしない。Release のトップレベル例外はログ行となる（終了コードだけで正常と扱わない）。`CharaPath.ini`、`Info/User.ini` 等は独自入力で、Setting.ini の移行とは別。

| 設定 | ゲームによる利用・選択肢 |
|---|---|
| `screenWidth` | 0 以下で `fullHD=true` なら1920、偽なら1280。それ以外は640〜7680に clamp。16:9 の高さは `2×round_even(width×9/32)`。標準 1280×720、1600×900、1920×1080、2560×1440、3840×2160。選択時に `fullHD=(幅>=1920)` にそろえる。標準値から外れても幅の履歴を破棄せずカスタム値として扱う |
| `frameRateLimit` | ゲーム UI に 無制限0、60、90、120、144、240、360 fps。0以下は追加で待たない。Present 後に前の締切＋周波数/FPS（整数）を使い、次締切が現在より1周期超過去なら現在に立て直す。変更時もアンカーをクリア。残り2ms超は1ms sleep、最後は spin。VSync をこの上限の代替としない |
| `masterVolume` / `musicVolume` / `soundEffectVolume` | 0〜100。ゲーム設定で5ずつ変える。入力欄の保存値型は整数 |
| `soundType` | DirectSound、Wasapi、WasapiExclusive、ASIO。起動時に選択。DirectSound では個別バッファ指定を使わない。Wasapi は共有／専有を切替、ASIO はドライバの経路 |
| `soundBufferSamples` | 0以下を自動0、正値は32〜8192。UI は1刻みだが 0 を増やすと32、32 を減らすと0。表示概算 ms は `samples×1000/48000` 小数1桁。実際のサンプルレートは機器が決める |
| `backgroundMovieLayoutMode` | `FullScreen` / `BlurredWithInset` の二択 |

ランチャーの標準幅も上記5種、読み込んだ幅が該当しなければ「カスタム」をリスト先頭へ加える。出力方式4種、映像切替2種。ランチャー楽曲の ZIP 取込はファイルエントリ 10000 件以下、宣言展開総サイズ 4 GiB 以下、転送時も当該エントリの長さを越える読み込みにしない。この数値は更新パッケージや TJA 本文の上限に流用しない。根拠：`G/src/Common/Config/ConfigManager.cs`、`G/src/Common/Config/ScreenResolutions.cs`、`G/src/Common/Config/SoundOutput.cs`、`G/src/Core/Util/FrameLimiter.cs`、`L/Pages/LaunchSettingsPage.xaml.cs` 10–25/138–146、`L/Services/SongImportService.cs` 17–18/67/141–174。

## DATA-D02 競合・バックアップ

| 実装元 | 一時ファイルと置換の契約 |
|---|---|
| ゲーム汎用 JSON と曲スコア | 同フォルダーの `.元ファイル名.<GUIDのN表記>.tmp` を排他的に新規作成。読み取り専用の既存先は事前拒否。エンコードした全内容を flush し、可能な範囲でディスクに flush。既存なら置換して `元パス.bak` に直前内容、新規なら移動するだけ。Unix で新規一時ファイルは所有者のみ読み書き、既存を置き換える時は元の permission を継承。失敗しても既存先を削除してやり直さない。最後に自分の一時ファイルだけ除去し、副次 I/O 失敗は隠す |
| ランチャーの汎用ファイル | 親を作り `元パス.launcher.tmp` に全内容を書き、既存でバックアップ指定（既定真）なら `元パス.launcher.bak` へコピー上書き。最後に、一時ファイルを元パスへ上書き移動。コピー/移動の失敗では一時ファイル削除を試みて例外を再送。書込み自体が try より前なので、書込み時の失敗では残渣があり得る |
| ゲームのランチャー用履歴 | 規定 record に対し同フォルダーの GUID 一時パスから移動。対象ディレクトリは非同期予約時に確定。設定やスコアとバックアップ方式を混同しない |

ランチャーの通常設定保存はインストール先ごとではない。`TaikoDive` 名のプロセスのうちランチャー自身が追跡するプレビュー以外が一つでもあれば保存拒否する。設定に対する未知キー保持と `.launcher.bak` 読み戻しを確認する。ゲーム側の `.bak`/ランチャー側の `.launcher.bak` それぞれ一世代で、一定時間の自動競合解決・同時書込の保証はない。根拠：`G/src/Core/Util/AtomicFile.cs` 10–60、`G/src/Common/LauncherBridge.cs`、`L/Services/FilePersistence.cs` 7–38、`L/Services/GameProcessService.cs` 9–22、`L/Services/GameSettingsStore.cs` 60–65。

## DATA-D03 Score.dat 書き出しのバイト編成

Shift_JIS（.NET のコードページ932、BOMなし）、Windows なら各行 CRLF。ファイル外形は `{`、4 スペースの `"scores": `（コロン後も1スペース）、4 スペースの `[`。各難易度は6スペースの二重引用符つき名前 `Easy`→`Normal`→`Hard`→`Oni`→`Edit`、次行に6スペースの `{`。8 スペースをつけ `Score`、`Gauge`、`Great`、`Good`、`Miss`、`RollCount`、`MaxCombo`、`Crown`、`ScoreRank`、`AIWin`、`AIScore`、`AIWardCrown` の順で各行 `キー=値`、値の引用符や行末カンマはない。数値は元コードのカルチャで文字列化（Gauge は浮動小数）。AI 区間文字列 null は空。6スペースの `}`、最後の Edit 以外は `},`。全体を4スペースの `]`、`}` で閉じる。これは JSON ではなく、一般 JSON ライターへ渡してはならない。高いスコアを保存しても曲ごとの他コースは保持し、同値なら内訳を置換しない。根拠：`G/src/Common/Score/ScoreDat.cs::tWriteDatFile,tSaveDatFile` 54–146。テスト推奨：先頭/末尾、5難易度、改行、キー名、バックアップの旧内容を各々照合。
