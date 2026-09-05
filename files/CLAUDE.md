# CLAUDE.md

このファイルは Claude Code にこのプロジェクトでの作業方法を指示します。

## プロジェクト概要

- Unity 6（6000.6.0f1）+ Universal 3D (URP) テンプレートで作成した玉転がしゲーム
- ビルドターゲット: StandaloneWindows64
- **ゲーム仕様・設計: `docs/requirements.md`（要件）と `docs/design.md`（詳細設計）を必ず参照。実装変更時は設計書も更新する**
- バージョン管理: 未導入（git 導入時にルートの `.gitignore` を使用）

## よく使うコマンド

Unity CLI（`unity` コマンド）を使用します。

```powershell
unity open .                # プロジェクトを Unity エディターで開く
unity status                # 接続中のエディター状態を確認
unity test .                # EditMode / PlayMode テストを実行
unity projects verify .     # ビルドを壊す整合性問題をチェック
unity build .               # プロジェクトをビルド
```

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

## コーディング規約

- ゲームplay用スクリプトは `Assets/Scripts/` に配置する
- テストコードは `Assets/Tests/EditMode/`・`Assets/Tests/PlayMode/` に配置する
- エディター拡張は `Assets/Editor/` に配置する

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
- 名前空間はPascalCaseでフォルダ構造を反映（`BallRolling.Gameplay` 等）
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
