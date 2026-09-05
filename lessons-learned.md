# Lessons Learned — Unity ゲーム開発（Claude Code + Unity CLI）

他プロジェクトに活かすための問題と解決の記録。各実装ステップ完了時に「## Step N」セクションへ追記する。

**記録形式**: `### 1. 発生した現象 (Problem)` / `### 2. 原因 (Root Cause)` / `### 3. 解決策・実装パターン (Solution & Code Pattern)` / `### 4. 次回への教訓 (Key Takeaway)`

---

## Step 1: 迷路ロジック + EditModeテスト

### 【事例】bool配列の初期化漏れで迷路が「全部開いた盤面」になる（テストが検出）
#### 1. 発生した現象 (Problem)
迷路生成ロジック実装後のテストで「行き止まりが0個」「別シードでも同一結果」の2つが失敗。生成された迷路は全方向壁なしの平坦な盤面だった。
#### 2. 原因 (Root Cause)
`Cell`構造体（壁の有無をbool4つで保持）の配列が C# の既定で **false初期化** されるため、全方向「壁なし」状態から生成していた。掘る処理は壁を false にするだけなので、何も変化しなかった。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
コンストラクタで明示的に全壁 true で初期化する。

```csharp
// 「あり」から「なし」を掘るタイプのデータは初期状態を明示する
for (var i = 0; i < Cells.Length; i++)
    Cells[i] = new Cell { North = true, South = true, East = true, West = true };
```
#### 4. 次回への教訓 (Key Takeaway)
bool配列はfalse初期化。「壁ありから掘る」設計では初期化を必ず明示する。ロジックをテストファーストに書くと、この種のバグを生成直後に検出できる。

### 【事例】テストの前提が仕様とずれていた（出口が行き止まりに含まれる）
#### 1. 発生した現象 (Problem)
「入り口・出口は行き止まりに含まれない」という自作テストが失敗。出口セル (9,9) が行き止まり（壁3方向）として検出された。
#### 2. 原因 (Root Cause)
完全迷路（全域木）では出口セルも構造上の行き止まり（葉）になり得る。除外すべきは「アイテムを置く候補」であって、構造判定そのものではない——テストの前提が仕様理解とずれていた。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
テストを緩和（行き止まり存在検証のみに）。除外要件は設計書に移して実装側で対応:

> アイテム配置: 行き止まり座標から**入り口・出口を除いて**乱数で1箇所選択（design.md）
#### 4. 次回への教訓 (Key Takeaway)
テスト失敗は「実装バグ」と「テストの前提誤り」の両方があり得る。どちらの判断をしたかを設計書に残し、後続実装に反映する。

### 【事例】再コンパイル直後のテスト実行で「テスト0件」「Network error」「Thread was being aborted」
#### 1. 発生した現象 (Problem)
`recompile` 直後に `run_tests` を実行すると `Total:0`（テスト未検出）。また `Network error: An error occurred while sending the request`、Consoleには赤いエラー `Failed to handle /api/exec request: Thread was being aborted.`（発生源は Unity.Pipeline 内部）も記録された。
#### 2. 原因 (Root Cause)
コンパイル中・ドメインリロード中はテストアセンブリが未完成で、Pipelineサーバーも応答不能。リロードと重なったコマンドはスレッドごと安全に中断される（Unity.Pipeline パッケージの仕様）。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
完了確認→待ち→実行の手順を必ず踏む:

```bash
unity command recompile
# recompile_status が "completed" になるまでポーリング（5秒間隔）
sleep 8   # ドメインリロード完了待ち（editor_status の domainReloadInProgress:false を確認）
unity command run_tests --mode editor
```
#### 4. 次回への教訓 (Key Takeaway)
ドメインリロード直後の接続エラー・未検出は**想定内**（unity-pipelineスキルに同旨の記載あり）。`Unity.Pipeline.*` 発生源で直後の操作が成功しているエラーは無視してよい。

## Step 2: EditorWindowによる迷路生成

### 【事例】capture_scene_view では Main Camera の調整結果を確認できない
#### 1. 発生した現象 (Problem)
スクリプトで Main Camera を真上向きに調整した後 `capture_scene_view` でスクリーンショットを撮ったが、斜め角度の画像になり調整が反映されていないように見えた。
#### 2. 原因 (Root Cause)
`capture_scene_view` は**エディターのシーンビューカメラ**（ユーザー操作対象のビュー）を撮影する。Main Camera とは無関係。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
検証目的で使い分ける:
- Main Camera（ゲーム画面）→ `unity command capture_game_view`
- エディターの作業ビュー → `unity command capture_scene_view`
#### 4. 次回への教訓 (Key Takeaway)
カメラ制御を実装したら `capture_game_view` で検証する。両キャプチャの使い分けを最初から把握しておく。

### 【事例】トップダウンカメラで座標(0,0)が「左下」に映り、設計書の「左上」と不一致
#### 1. 発生した現象 (Problem)
生成確認のスクリーンショットで、入り口(0,0)の緑マーカーが画面左下に表示された。設計書には「入り口=左上」と記載していた。
#### 2. 原因 (Root Cause)
カメラを真下向き（`Quaternion.Euler(90, 0, 0)`）にすると、ワールドZ+方向が**画面上**に投影される。(0,0)が画面左下になるのは座標系として正しい。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
実装（座標系として正しい投影）を優先し、requirements.md の記載を「左下=入り口、右上=出口」に修正した。
#### 4. 次回への教訓 (Key Takeaway)
画面上の位置を仕様に書くときは、カメラ角度とワールド座標の対応（Z+が画面上、等）まで設計段階で確定する。

### 【事例】Editor asmdef からロジックアセンブリへ参照追加が必要
#### 1. 発生した現象 (Problem)
Editor フォルダのツールから `MazeGenerator`（ランタイム側アセンブリ BallRolling 内）を呼ぶコードが、参照未設定ではコンパイルできない。
#### 2. 原因 (Root Cause)
`Assets/Editor/BallRolling.Editor.asmdef` の references にロジックアセンブリが含まれていなかった。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
asmdef の references に `"BallRolling"` を追加してから実装。エディターツール→ゲームロジック参照はこの1行で解消する。
#### 4. 次回への教訓 (Key Takeaway)
エディターツールでゲームロジックを再利用する構成では、Editor asmdef への参照追加を先に行う（asmdefだけ置いた状態のWarning「no scripts associated with it」はスクリプト配置後に自動解消する一時的なもの）。

---

## 一般（環境構築時に記録済みの事例）

### 【事例】Unityエディタ拡張でのオブジェクト生成とUndoの不整合
#### 1. 発生した現象 (Problem)
AIがエディタウィンドウからオブジェクトを生成するコードを実行した際、画面上には生成されるが、Ctrl+Z（Undo）を押しても消えず、シーンを保存して開き直すと生成したオブジェクトが消失する。
#### 2. 原因 (Root Cause)
`Instantiate` や `new GameObject()` をそのまま使用していたため、Unityのエディタ側のシリアライズ管理（シーンの変更検知）に登録されていなかった。また、Undoシステムへの登録が行われていなかった。
#### 3. 解決策・実装パターン (Solution & Code Pattern)
エディタ拡張内でオブジェクトを生成・操作する場合は、必ず `Undo.RegisterCreatedObjectUndo` を使用し、プレハブを配置する場合は `PrefabUtility.InstantiatePrefab` を使用する。

```csharp
// 修正後のベストプラクティス
GameObject go = new GameObject("SpawnedObject");
// 生成直後に必ずUndoに登録する（これにより変更検知とUndoが有効になる）
Undo.RegisterCreatedObjectUndo(go, "Create SpawnedObject");
```
#### 4. 次回への教訓 (Key Takeaway)
Unityのエディタ拡張でシーン上の何かを変更・生成する場合は、ランタイム用のコードをそのまま使わず、必ず `UnityEditor` 名前空間の `Undo` や `PrefabUtility` を経由させること。
