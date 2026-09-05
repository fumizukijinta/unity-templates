# 3Dゲーム開発環境セットアップ手順書（Claude Code 実行用）

この手順書は、新しい**3D** Unityゲームプロジェクトの開発環境を構築するための完全な手順です。Claude Code が読んで実行します。テンプレート雛形は `C:\Unity\templates\files\` にあります。

## 0. 入力の確認（ユーザーに質問する）

開始前にユーザーへ確認する:

| 項目 | 例 | 用途 |
|---|---|---|
| ゲーム名（PascalCase） | `MyNewGame` | フォルダ名・リポジトリ名・CLAUDE.md |
| ゲームの説明（1行） | `3Dアクションパズル` | README・CLAUDE.md |
| GitHubリポジトリ公開範囲 | private（推奨）/ public | リポジトリ作成 |

## 1. 前提の確認（失敗したら止めて報告）

```powershell
unity --version                  # Unity CLI が動作する
unity auth status                # サインイン済み
unity editors list               # Unity 6系（6000.x）がインストール済み、パスを控える
gh auth status                   # GitHub サインイン済み（未ログインなら unity mcp 同様の手順を案内）
claude mcp list                  # unity-editor-mcp が Connected（未登録なら unity mcp configure claude-code）
```

## 2. プロジェクト作成

```powershell
# <GameName> を実際のゲーム名に、--editor-version はインストール済み Unity 6 系に置換
unity projects new <GameName> --path C:\Unity\project --editor-version 6000.6.0f1 --template com.unity.template.urp-blank
```

- **注意**: `unity projects new` は既存パスを拒否する。親（`C:\Unity\project`）は既存でも、その下の新規フォルダ名なら作成できる
- 作成されたパス: `C:\Unity\project\<GameName>`

## 3. 雛形ファイルの配置

```powershell
# カレントディレクトリを新プロジェクトへ移動してから
cd C:\Unity\project\<GameName>
Copy-Item C:\Unity\templates\files\.gitignore .
Copy-Item C:\Unity\templates\files\.gitattributes .
Copy-Item C:\Unity\templates\files\.claude .claude -Recurse
Copy-Item C:\Unity\templates\files\CLAUDE.md .
Copy-Item C:\Unity\templates\files\README.md .
```

プレースホルダーを置換する（CLAUDE.md と README.md の両方）:

| プレースホルダー | 置換値 |
|---|---|
| `{{GAME_NAME}}` | ゲーム名 |
| `{{GAME_DESCRIPTION}}` | ゲームの説明 |
| `{{GAME_TYPE}}` | `3D` |
| `{{TEMPLATE_NAME}}` | `Universal 3D (URP)` |
| `{{UNITY_VERSION}}` | 実際のバージョン（例: 6000.6.0f1） |
| `{{ART_ASSET_TYPES}}` | `Models・Materials・Textures` |

## 4. git の初期化

```powershell
git init -b main
git lfs install
git add -A
git commit -m "Initial commit: Unity <version> URP project scaffold"
```

## 5. GitHub リポジトリ作成とpush

```powershell
gh repo create <GameName> --private --source . --remote origin --push --description "<説明>"
```

## 6. Pipeline パッケージ導入（重要: エディター起動の前に実行する）

```powershell
unity pipeline install --project-path .
```

**なぜ先に**: manifest.json に直接書かれるため、エディター起動時にロードされる。起動中に install するとエディターがバックグラウンドで変更を検知できず、フォーカスを当てるまで反映されないことがある。

## 7. エディター起動と確認

```powershell
unity open .                              # 起動（初回はLibrary生成で数分）
unity status                              # "ready" になるまで待つ（ポーリング）
unity command editor_status               # 通信確認
unity command set_autotick --enable true  # autotick確認（既定で有効なはず）
```

- `unity status` が接続不能なら `unity pipeline list` で Safe Mode（コンパイルエラー）を確認
- コマンドが固まる場合はモーダルダイアログの可能性（`editor_status` の `dialog` フィールド）→ ユーザーに伝える

## 8. スキル導入

```powershell
unity skill install claude-code --local
```

## 9. 疎通テスト

```powershell
# Sphere生成→検索→削除でライブ操作を確認
unity command create_gameobject --name Ball --primitive sphere
unity command find_gameobjects --name Ball
unity command delete_gameobject --target /Ball
# GitHubにプッシュ（.metaの自動生成分を含む）
git add -A; git commit -m "Add scaffold files with meta"; git push
```

## 10. 完了報告

以下を表で報告する: プロジェクトパス / Unityバージョン / テンプレート / リポジトリURL / Pipelineポート / MCP状態 / 疎通テスト結果

## トラブルシューティング

| 症状 | 対処 |
|---|---|
| `unity projects new` が「パスは既に存在します」 | 別フォルダ名にするか、既存が空なら一時親ディレクトリに作成→移動 |
| `unity status` に繋がらない | `unity pipeline list` で Safe Mode / サーバー状態を確認。C#エラーなら修正して再起動 |
| コマンドがタイムアウト | `editor_status` で `blocked_by_dialog` を確認。ダイアログは人間が閉じる必要 |
| gh未認証 | `! gh auth login` をユーザーに実行してもらう（ブラウザ認証） |
