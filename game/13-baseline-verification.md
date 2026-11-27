# 13｜本体の参照版で実行した検証

2026-11-27（JST）、Windows 11・.NET SDK 10.0.404。固定コミットを隔離ディレクトリに展開し、元の作業ツリーに対してビルドや保存を行わずに検証した。結果は**指定した元実装の観測**であり、未作成の移植版に対する認証ではない。

| 対象 | 実行した試験 | 結果 |
|---|---|---|
| G `fb12417` | `dotnet test tests/TaikoDive.Tests/TaikoDive.Tests.csproj -c Release` | 584 合格、0 失敗、0 スキップ。TRX 集計を照合 |
| D `ff729c1` | `dotnet test tests/TaikoDive.Tests/TaikoDive.Tests.csproj -c Release` | 564 合格、0 失敗、0 スキップ。D の `lib/` を併せて展開した結果 |
| G `fb12417` | Release x64 ビルド、合成曲と隔離素材で起動 | ビルド成功。起動プローブは各 3 回の観測で、`TitleShow=true` → `eDemo`、`false` → `eSongSelect` |
| G `fb12417` | 配布ワークフローの Python テスト、および実 `FontRender` を用いた GDI+ 縦書き回帰（GPU は使わない） | ワークフロー 2 件合格。縦書き回帰 fixed 合格：通常文字を描画し、欠損または空のタイトルは縁あり/なしとも透明 16×16 を返す |

## OBS-01 起動失敗も互換条件になる

- G の隔離起動で `Info/Chara3D.json`（配置済みモデルを指す `modelsPath`）がないと、3D を使うアニメの初期化が `InvalidDataException` で失敗した。**プレイヤー用素材には一般的な 2D フォールバックを想定しない**。同じモデルの必要ファイルを用意する。AI ドンやぷちキャラは別の読込経路。
- 次に通常ジャンルの `box.def` はあるが、そのフォルダーに `CenterText.apt` がない合成ケースを起動した。`SelectSong.tRecovery` → `BoxCenterTextParse.tParseBoxText` で `FileNotFoundException`。従って、登録する通常 BOX には対応ファイルを用意する（合成文字を渡す特殊 BOX と区別）。通常の読み込みは Shift_JIS、ファイルの非空行ごとに文字画像を生成。例では、合成 `CenterText.apt` を追加して起動が進んだ。
- これらは Release 起動でエラーログが出てもプロセス終了コードが 0 だった。スモークは終了コードだけで判定しない。合成ケースでは `TAIKODIVE_E2E_STATUS_PATH` が作られることと内容を確認した。

## OBS-02 中断した試験と未検証範囲

- ゲームの既存 Windows E2E は正規の素材を揃えると起動待ちを通過したが、対象ウィンドウを前面にできずキー送信前に中断。また固定スクリプトの最初のコイン投入は PageDown だが、このコミットの実装は PageUp/Home で投入し PageDown で払い出すため、そのまま適合性の判定に使わない。プローブによる確認は開始シーンまでで、デモ以降の実キー、完奏、結果画面、段位、音と映像の差分比較は未検証。
- 物理コントローラー、音声遅延、Windows 以外、実サービスへの更新、ピクセル単位の一致、開発版の実機全操作はこの作業では試験していない。移植版は読者が `08-conformance.md` の手順で比較する。
