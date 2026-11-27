# 27｜Launcher の構成上の制約と移植案

対象は Launcher L の C#/.NET 10 と WinUI。以下は移植の選択肢であって、元実装の仕様変更やベンチマークではない。本体と Assets のメモリ管理・描画・音声時計の分析は [本体の構成分析](../game/27-architecture-tradeoffs.md)。

## 維持する部分と分離する部分

Windows での互換性を優先するなら、WinUI を維持し、`launcher.json` の保存保護、ゲーム用データの検証・変換、更新の検証ロジックをビューから分離する。設定と更新の失敗状態を WinUI の可視状態に対応させる。プロセス、ウィンドウ配置、IME、フォント列挙、コントローラーの検出だけを OS 依存の境界に置く。

## 別 UI 基盤の候補

| 案 | 見込める改善 | 移植負担 | 採用条件 |
|---|---|---|---|
| **ランチャーだけ分離：WinUI → Avalonia** | ゲーム本体とは別に、C#でWindows/macOS/Linux向け設定・曲・履歴のUIを作る候補。[1] | XAMLやWinUIコントロールをそのまま移植する前提にしない。デバイス割当、フォント、プロセス検出、自己更新は各OS対応。Windows版描画との一致と操作性の同等性は別々に判定。 | ゲームとランチャーを異なる時期に移植する場合。Windows版しか動かないゲームを、ランチャーの移植だけで他OS対応と呼ばない。 |

## 比較と採否

Windows 版を残し、別 OS 版は表示・IME・ウィンドウ制御、子プロセス、更新の各アダプターを段階的に差し替える。調査箇所：`L/TaikoDiveLauncher.csproj`、`L/Services/ControllerInputMonitor.cs`、`L/Services/GameProcessService.cs`。

同じ合成データでキーの保存値、未知項目・バックアップ、履歴の暦境界、更新署名の検証結果を厳密比較する。表示は DPI・入力フォーカス・狭い縁の実ヒット・テーマを別途確認する。Windows 以外のプロセス／更新／入力を未実装のまま Windows 版と同等としない。候補実装と性能の測定は行っていない。

## Sources

[1] https://docs.avaloniaui.net/docs/supported-platforms
