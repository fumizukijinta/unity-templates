# {{GAME_NAME}}

Unity 6 で開発する{{GAME_TYPE}}ゲーム。

## 開発環境

| 項目 | 値 |
|---|---|
| エンジン | Unity {{UNITY_VERSION}} |
| レンダーパイプライン | {{TEMPLATE_NAME}} |
| ビルドターゲット | StandaloneWindows64 |
| テスト | Unity Test Framework |

## セットアップ

```powershell
# Unity CLI でプロジェクトを開く
unity open .

# エディターが未インストールの場合
unity install {{UNITY_VERSION}}
```

## テスト

Unity Test Framework を使用します（EditMode / PlayMode 両対応）。

```powershell
# すべてのテストを実行
unity test .

# EditMode のみ実行
unity test . --test-mode EditMode
```

Claude Code を使用している場合は `/test` カスタムコマンド（`.claude/commands/test.md`）も利用できます。

## ディレクトリ構造

```
{{GAME_NAME}}/
├── Assets/            # ゲームアセット・スクリプト
│   └── Scripts/       # ゲームplay用スクリプト
├── Packages/          # パッケージマニフェスト
├── ProjectSettings/   # プロジェクト設定
├── .claude/           # Claude Code 設定（テスト自動化コマンドなど）
├── CLAUDE.md          # Claude Code 向けプロジェクトガイド
├── .gitignore         # バージョン管理用の除外設定
└── README.md          # このファイル
```
