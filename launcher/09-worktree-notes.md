# 09｜Launcher の作業ツリー観測（非規範）

2026年11月27日 JST の局所的な観測。未コミット差分と未追跡文書であり、固定版 L と同一とは検証していない。以下の内容を Launcher の規範仕様に合流しない。

| 作業ツリー | 観測した差分 |
|---|---|
| TaikoDive-Launcher | 追跡済み27ファイルに変更・削除。ホーム、入力・起動・楽曲画面、保存サービス、README、テスト、スモークツール等を変更し、旧入力デバイス画像2枚・そのREADME・`Pages/HomePage.Cards.cs` を削除。未追跡には紹介曲画面・モデル・保存サービス・テスト、演奏活動画面・モデル・サービス・テスト、コントローラー描画、プレビュー画像と文書がある。紹介設定はゲーム側と共有しゲーム実行中の保存拒否、保存時 `.launcher.bak`、未保存移動の確認を記載。演奏活動集計コードはローカル日付換算で365日・月曜始まり53週を生成する。その他の詳細な振る舞いは未検証。 |
| TaikoDive-Launcher-Recovery | `Pages/LauncherSettingsPage.Recovery.cs`、`Pages/LauncherSettingsPage.xaml`、`Tests/LauncherRecoveryUiTests.cs` を変更。差分では設定の再読込・退避して初期化のボタン処理を削除し、保護中の案内文を変更している。保存保護全体が削除されたとは限らない。 |

実機の動作判定は行っていない。本体の実験・開発版は [本体＋Assets の補遺](../game/09-dev-variant.md)。
