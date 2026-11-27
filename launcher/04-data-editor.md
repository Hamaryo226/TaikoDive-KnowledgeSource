# 04｜Launcher のゲーム設定・曲データ編集

対象は Launcher L。ここで扱うのは既存のインストールを編集するクライアント側の処理。書式・ゲーム側の既定は [本体データ仕様](../game/04-data.md)、入出力の所有権は [連携契約](../INTEROP.md)。Launcher 自身の `launcher.json`、配置、テーマは [Launcher の永続化](20-ui-persistence-details.md)。

## ゲーム設定の読込み・編集・保存

Launcher は設定保存前にゲーム実行中なら拒否する。既存 JSON をオブジェクトとして読んで対象キーのみ更新し、それ以外は保持するが、廃止済み `charaAnimationFrameSkip` と `reduceCharaTextureColorTo16bit` は大小文字無視で削除する。音量を 0–100、保存幅を 640–7680、バッファを 0 または 32–8192 に制限し、`fullHD` は幅が1920以上かで更新する。ゲーム側の許容値と Launcher の保存制限を混同しない。JSON が不正・非オブジェクトなら保存失敗。根拠: `L/Services/GameSettingsStore.cs::LoadAsync,SaveAsync`。

ランチャーの標準幅は1280、1600、1920、2560、3840の5種（16:9）、読み込んだ幅が該当しなければ「カスタム」をリスト先頭へ加える。出力方式4種、映像切替2種。ランチャー楽曲の ZIP 取込はファイルエントリ 10000 件以下、宣言展開総サイズ 4 GiB 以下、転送時も当該エントリの長さを越える読み込みにしない。この数値は更新パッケージや TJA 本文の上限に流用しない。根拠：`L/Pages/LaunchSettingsPage.xaml.cs` 10–25/138–146、`L/Services/SongImportService.cs` 17–18/67/141–174。

## 書き込み制御・バックアップ

| 実装元 | 一時ファイルと置換の契約 |
|---|---|
| ランチャーの汎用ファイル | 親を作り `元パス.launcher.tmp` に全内容を書き、既存でバックアップ指定（既定真）なら `元パス.launcher.bak` へコピー上書き。最後に、一時ファイルを元パスへ上書き移動。コピー/移動の失敗では一時ファイル削除を試みて例外を再送。書込み自体が try より前なので、書込み時の失敗では残渣があり得る |

ランチャーの通常設定保存はインストール先ごとではない。`TaikoDive` 名のプロセスのうちランチャー自身が追跡するプレビュー以外が一つでもあれば保存拒否する。設定に対する未知キー保持と `.launcher.bak` 読み戻しを確認する。ゲーム側の `.bak`/ランチャー側の `.launcher.bak` それぞれ一世代で、一定時間の自動競合解決・同時書込の保証はない。根拠：`G/src/Core/Util/AtomicFile.cs` 10–60、`G/src/Common/LauncherBridge.cs`、`L/Services/FilePersistence.cs` 7–38、`L/Services/GameProcessService.cs` 9–22、`L/Services/GameSettingsStore.cs` 60–65。

## プロフィールの `Info/User.ini` 編集

ランチャーは既存の文字コードと改行を保持し、未知キー・コメントを残して選択スロットの4項目を更新、直前のファイルを `.launcher.bak` に保存する。新規ファイルは BOM 付き UTF-8。ゲーム実行中は保存拒否。名前とキャラは空不可、称号は空可、改行と `=` は不可、プレート番号は0以上にする。ランチャーの読み取りはキー大小無視、`#` コメントも認識し、最初の `=` より後すべてを値とするため、ゲーム側とは厳密には一致しない。既存のキー表記を保持する更新方法なので小文字キーを自動的にゲーム互換に正規化する保証はない。

根拠：`L/Services/UserProfileStore.cs`、`L/Services/FilePersistence.cs`。ゲームが解釈するキーと既定値は [本体の User.ini](../game/04b-replay-users.md)。

## 曲の発見・順序と外部 Songs

Launcher ライブラリ走査は `.tja` を大小文字無視で列挙し、タイトルの文化依存・大小文字無視順にする。ファイルサイズ 8 MiB 超は問題として扱う。UTF-8 厳格デコードを試し失敗時 CP932、BOM は除去。コメント `//` を除いたヘッダから TITLE、WAVE、SUBTITLE、GENRE、DEMOSTART を読み、WAVE が Songs 外なら拒否、無いファイルを警告する。GENRE 未指定なら祖先の `box.def` から `#GENRE:` を探す。これと曲順編集時のタイトル抽出は別実装で、後者は先頭512行・1行16 Ki文字まで、`TITLEJA` 優先、`#START` で終了する。根拠: `L/Services/SongLibraryService.cs::ScanAsync,ReadChart,ReadFolderGenre`、`L/Services/SongOrderService.cs::ReadTitle,ReadHeaderLine`。

Launcher の曲順ファイルは各ジャンル直下 `.taikodive-order`。UTF-8 BOM なし、ジャンルからの相対 TJA パスを `/` 区切りで1行ずつ。読み込みは大小文字を区別せず初出を採用、未指定曲は列挙順を維持して末尾。保存はゲーム実行中に拒否し、ジャンルが現在の Songs 直下であること、指定集合に重複がなく現在の再帰列挙集合と一致することを確認する。書込みは原子的置換とバックアップを使う。根拠: `L/Services/SongOrderService.cs::LoadSongs,SaveOrderAsync,SaveOrderWithoutProcessCheckAsync`。

- L は再帰 tja 一覧を作り、保存時ゲーム停止を要求、ジャンルが現在の Songs 直下であること、現在の tja 相対パス集合と要求集合が大文字小文字無視で一致し重複がないことを確認。UTF-8 BOMなし、改行区切り、非空なら末尾改行付きでバックアップ付き原子的書込。保存を排他する。タイトル表示は先頭最大512行・各行最大16Ki文字、strict UTF-8 を試し不正バイトなら CP932、BOM検出あり。TITLEJA: 非空を優先、なければ最後の TITLE:、#START で停止、なければ拡張子なしファイル名。キー比較は大文字小文字無視。（同:132-249）
- L Services/SongsPathService.cs:17-177,180-420：ゲーム稼働中の変更・復元・補完は禁止。既存ディレクトリを正規化し、Songs 自身またはバックアップ内は指定不可。同じ実効パスなら必須アセット補完のみ。標準 Songs を Info/TaikoDiveLauncher/Songs.original に移し、元の Songs に外部ディレクトリのシンボリックリンクを作る。既にリンクなら元リンクを交換。失敗時はベストエフォートで元に戻しバックアップは削除しない。補完は各直下ジャンルの box.def、CenterText.apt と Image 配下を対象にない場合だけコピー、既存は上書きしない。コピー後の失敗でも外部コピーはロールバックされない。復元はリンクを削除しバックアップを移動、外部の補完ファイルは残す。パス比較は ordinal ignore-case。

## 受入

初見のキーを持つ `Setting.json`、不正 JSON、稼働中のゲーム、重複・欠落・追加された譜面、CP932 と UTF-8 を別々に試す。既存のスコアは書き換えず、元の `.launcher.bak` と書込後の内容を読み戻す。外部 Songs 指定で不足アセットだけ補完、既存の同名ファイルは不変。シンボリックリンク作成失敗時は復元を試み、補完済み外部ファイルは残る。ゲーム実行中の変更・復元・曲順保存は拒否。
