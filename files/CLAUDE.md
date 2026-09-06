# CLAUDE.md

このファイルは Claude Code にこのプロジェクトでの作業方法を指示します。

## プロジェクト概要

- Unity {{UNITY_VERSION}} + {{TEMPLATE_NAME}} テンプレートで作成した{{GAME_TYPE}}ゲーム（{{GAME_NAME}}）
- {{GAME_DESCRIPTION}}
- ビルドターゲット: StandaloneWindows64
- **ゲーム仕様・設計: `docs/requirements.md`（要件）と `docs/design.md`（詳細設計）を作成して参照。実装変更時は設計書も更新する**
- バージョン管理: GitHub（画像・動画・音声はコミット除外、詳細は下記）

## 開発セッション開始時の手順

1. `unity skill install claude-code --local` — スキルを最新化
2. `unity open .` — エディター起動（初回やLibrary再生成後はインポートで数分かかる場合あり）
3. `unity status` で `ready` を確認（Pipelineサーバー起動まで待つ）
4. `unity command editor_status` で通信確認（playMode/compiling の状態も見える）
5. MCP確認：`claude mcp list` で `unity-editor-mcp` が **Connected** であることを確認（セッション内なら `/mcp` でも可。エディター起動後に Pipeline コマンドが `mcp__unity-editor-mcp__*` ツールとして公開される）

## MCP連携

- `unity-editor-mcp`（ユーザースコープ、`~/.claude.json`）が `unity mcp` サーバーを登録済み
- エディター起動中は Pipeline コマンドが MCP ツール（`mcp__unity-editor-mcp__*`）として自動公開される
- 設定変更は `unity mcp configure claude-code`、一覧は `claude mcp list` で確認

## よく使うコマンド

Unity CLI（`unity` コマンド）を使用します。

```powershell
unity open .                # プロジェクトを Unity エディターで開く
unity status                # 接続中のエディター状態を確認
unity test .                # EditMode / PlayMode テストを実行
unity projects verify .     # ビルドを壊す整合性問題をチェック
unity build .               # プロジェクトをビルド
```

## コーディング規約

- ゲームplay用スクリプトは `Assets/Scripts/` に配置する
- テストコードは `Assets/Tests/EditMode/`・`Assets/Tests/PlayMode/` に配置する
- エディター拡張は `Assets/Editor/` に配置する
- アセットは `Assets/Art/`（{{GAME_TYPE}}: {{ART_ASSET_TYPES}}）、`Assets/Prefabs/`、`Assets/Audio/` 等に種類ごとに分けて配置する

### 命名規則（Unity公式ガイドライン準拠）

- **PascalCase**: クラス・構造体・列挙型・メソッド・publicフィールド・プロパティ・定数
- **camelCase**: ローカル変数・メソッドパラメータ・privateフィールド（`_camelCase` プレフィックスを使用）
- 変数名は**名詞**、boolは動詞接頭辞（`isDead` / `hasKey`）、boolを返すメソッドは疑問形（`IsGameOver()`）
- メソッド名は**動詞句**で始める（`GetDirection` / `FindTarget`）
- インターフェースは `I` + 形容詞（`IDamageable`）
- 列挙型は単数形の名詞（`WeaponType`）。`[Flags]` 付きのみ複数形
- イベントは動詞句: 直前は現在分詞（`OpeningDoor`）、直後は過去分詞（`DoorOpened`）。発火メソッドは `On` プレフィックス
- **MonoBehaviourは1ファイル1クラス、ファイル名 = クラス名**
- ハンガリアン記法・略語は使わない。冗長な名前も避ける（`Player`クラス内なら `Score` でよく `PlayerScore` は不要）
- 名前空間はPascalCaseでフォルダ構造を反映（`{{GAME_NAME}}.Gameplay` 等）
- `= 0` / `= null` 等の冗長な初期化子は書かない。アクセス修飾子は明示する

## バージョン管理の運用

- 画像・動画・音声（とその .meta）は容量対策で**コミットしない**（.gitignore で除外済み）
- 3Dモデル（fbx等）・フォント・ライブラリ（dll等）は Git LFS で管理
- メディアファイルはgit外で別途バックアップが必要（他マシンでクローンした場合は手動配置）
- `.claude/skills/` はgit管理外。開発開始前に `unity skill install claude-code --local` で最新版を導入してから作業する

## Unityアセット操作の規約

- 新規ファイル作成時に **.meta は手書きしない**。Unityエディターの自動生成を待つ（フォーカスが無い場合は `unity command recompile` で AssetDatabase 更新をトリガー）
- 生成ファイルと .meta は**セットでコミット**する（.meta 欠落は GUID 再生成→参照切れの原因）
- ライブエディターが接続中のときは `.unity` / `.prefab` / `.asset` を直接編集せず Pipeline コマンド（`create_gameobject` 等）で操作する

## トークン節約規約（ガイドライン理解を損なわない範囲で）

- コマンド出力は必要行のみ取得（`| Select-Object -First N` 等で絞る）
- ドキュメント・スキルは必要なセクション/ファイルのみ読む（無闇な全文読み込みをしない）
- 反復ポーリングはバックグラウンド実行に任せ、結果は最終出力のみ確認
- 応答は簡潔に（表・箇条書き優先、装飾的な繰り返しを避ける）
- **節約対象は一時的な出力のみ**。仕様（docs/）・手順・規約の内容そのものを削らない（正確性優先）

## 開発ステップの記録

- 各実装ステップ完了時に、発生した問題と解決を `C:\Unity\templates\lessons-learned.md` の**既存テンプレ形式**（4セクション構成）で「## Step N」セクションへ追記する（既存記述は消去・変更しない）

## エージェント分業（.claude/agents/）

役割別サブエージェントを定義済み。メインセッションが指揮し、ユーザーとの窓口はメインが担う。

| エージェント | 役割 | モデル | 使用場面 |
|---|---|---|---|
| `game-designer` | 要件整理・設計書更新 | glm-5.3[1m] | 新機能・仕様変更・設計レビュー |
| `game-coder` | 実装 | glm-5.3[1m]（セッション継承） | 実装ステップ・リファクタリング |
| `game-tester` | テスト作成・実行・スモークテスト | glm-5.3-flash | テスト追加・実行、失敗の切り分け、プレイモード動作確認 |
| `bug-investigator` | 再現・診断・原因特定（修正しない） | glm-5.3[1m] | 原因不明・影響広範なバグの調査 |
| `bug-fixer` | 修正の実装 | glm-5.3-flash | 調査結果を受けた修正、軽微バグの直接修正 |
| `git-utility` | コミット・プッシュ・ドキュメント雑務 | glm-5.3-flash | コミット・push・lessons-learned追記 |

### 運用ルール

- 典型フロー: 設計 → `game-coder` 実装 → `game-tester` 検証。バグは `bug-investigator` → `bug-fixer`
- **git commit / push はメインセッションのみ**が行う（エージェントは行わない。実務は `git-utility` に委任可）
- エージェントは CLAUDE.md と docs/ を読んだ前提で動く。申し送り（確認事項）を上げてきたらメインが判断する
- ユーザーは「〜のエージェントで（あるいは役割で）〜して」と指示してもよいし、メインに任せてもよい

### フローの軽量化ルール（BallRolling開発実績からの教訓）

- **ステップ粒度は「動くものができる」単位で粗くする**: 設計→実装→確認の儀式は1ステップあたり30分超のコスト。関連機能は1ステップに統合し、ユーザー確認ポイントは「実際に操作できるものができた時」に集約する
- **詳細設計の全面起草は省略可**: 小〜中規模機能は、メインが確定事項（仕様判断・ジオメトリ等の暗黙前提）のみ渡して `game-coder` に一任してよい。`game-designer` の起草は設計判断が複数絡む場合や設計レビュー時に使う
- **バグは規模でフローを選ぶ**: 軽微バグ（原因自明・修正1〜数行）は `bug-fixer` 修正＋スモークテストで完了。`bug-investigator` → `bug-fixer` → `game-tester` の3段は原因不明・影響広範なバグのみ
- **Pipeline半吊りの早期判定**: コマンドが2回連続タイムアウト/Unauthorizedしたら、それ以上待たずに①実行中エージェント停止②Unity再起動③autotick再有効化。ハートビート更新≠コマンド応答（半吊りでもハートビートは進む）
- **実機プレイ確認は省略しない**: スモークテスト合格＝実プレイ合格ではない（呼び出しコンテキストの差で偽陽性が出る実例あり）。ユーザーの実プレイ確認を最終ゲートとする
