# TaikoDive 本体＋Assets ←→ Launcher の連携契約

両方をつないで使うときの境界だけをまとめる。**本体＋Assets** と **Launcher** は別プロセス・別仕様。Launcher がなくても、対応する Assets を配置した本体はゲームフォルダーを作業ディレクトリとして直接起動できる。Launcher はインストールを管理するクライアントで、演奏・判定の状態を自身で生成しない。

## プロセス境界

| ID | 契約 | 根拠 |
|---|---|---|
| PLAT-04 | キャラクタープレビューは `--character-preview`、リプレイ連携は先頭引数 `--launcher-replay`、パス、`--user-slot`、1〜9 の順を使う。リプレイは保存場所、利用者、形式、譜面ハッシュを本体側でも検証する。リプレイ中のゲスト・2人プレイの一時変更を設定ファイルに永続化しない。 | G/src/Common/Program.cs、G/src/Common/LauncherBridge.cs、G/docs/launcher-bridge.md |

通常起動のファイル選択、プロセスの重複検査と作業ディレクトリは [Launcher の起動](launcher/07-update-and-launch.md)。リプレイのパス・スロット・譜面照合は [本体データ仕様](game/04-data.md)。描画専用子プロセスの設定JSON・PNG・終了コードは [本体のプレビュー契約](game/22-resource-lifecycle.md)、呼出しと結果表示は [Launcher の見た目編集](launcher/20-ui-persistence-details.md)。

## ファイルの所有権

| ファイル／置き場 | 本体側 | Launcher 側 |
|---|---|---|
| `Setting.json` | 起動時に読込み・既定値の生成と実行時保存。`p1Keys`／`p2Keys` はここに入る | 停止中に編集。ゲーム起動中はプロセス名によって保存を拒否。起動中に設定を注入しない |
| `Info/User.ini`、`Info/Chara3D.json` | 選択したユーザー・名前札と外観を読み、演奏に使用 | プロフィールや外観を既存の形式で編集し、専用プレビューは別起動 |
| `Info/ScoreData/<ユーザー>/<SongKey>/` | 演奏に応じ、互換形式で `Score.dat`、`DanHistory.jsonl`、`Replay/course-<Course>/best.json` を記録 | スコア等を読む。リプレイ再生を要求できるが、新しい成績は作成しない |
| `Info/TaikoDiveLauncher/history/<id>.json` | 完了した対象演奏をIDごとに記録 | 日別・週別・累積を集計。`Score.dat` の変更日時をプレイ回数とみなさない |
| `Songs/`、`.taikodive-order`、`Info/TaikoDiveLauncher/demo-songs.json` | 譜面・曲順・デモ設定を規定通り読む | 曲ライブラリを管理し、順序・デモ設定を編集。外部 Songs の指定・復元は Launcher の操作 |
| `TaikoDiveLauncher/launcher.json`（通常はローカルアプリデータ） | 本体の設定として読み込まない | Launcher 固有の設定。ゲーム配置にある `Setting.json` と無関係 |

詳細なバイト形式は [本体の保存仕様](game/04-data.md)、Launcher が更新する範囲と競合防止は [Launcher のデータ編集](launcher/04-data-editor.md)。同じディレクトリに一時ファイルを置いて原子的に置換しても、複数プロセスの同時編集が自動で排他されるわけではない。

## 外部連携設定

| ID | 契約 | 根拠 |
|---|---|---|
| PLAT-07 | `Info/TaikoDiveLauncher/integrations.json` の version 1 で Discord とオーバーレイを制御する。Discord 公開および曲名公開は既定で無効。オーバーレイはローカルファイルで、状態を原子的に更新し、5 秒以上古ければ表示せず、終了時にクリアする。外部サービスが利用できなくてもゲーム本体の判定を変更しない。 | G/docs/launcher-bridge.md、G/src/Common/Discord/Discord.cs |

音量 API や起動プローブは本体が提供する。これらと OBS・Discord を Launcher の必須内部サービスに含めない。音量 API・起動診断は [本体の外部連携](game/07-platform-distribution.md)。

## 配布と相互運用の受入

ゲーム本体・Assets の出力側は [配布物の仕様](game/07-platform-distribution.md)、Launcher の取得・検証・配置・復旧は [更新仕様](launcher/07-update-and-launch.md)。本体と Assets の版識別は独立し、Launcher 自身の版はどちらとも別。

合成インストールで、ゲーム停止時に Launcher から設定変更→本体直接起動で読み戻し、本体開始中の編集拒否、プレイ履歴の読取、リプレイ受渡しと拒否条件、3Dプレビューと通常起動の区別をそれぞれ確認する。ゲームなしで通る Launcher の試験、Launcher なしで通る本体の試験、両方を接続した試験は結果を混ぜない。
