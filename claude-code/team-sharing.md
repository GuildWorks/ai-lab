# Claude Code チーム共有ガイド

チームで Claude Code の設定・ノウハウを共有するための指針です。
「**何を共有でき、何を共有してはいけないか**」を設定の階層構造に沿って整理します。

方針: **推奨テンプレ配布**（強制はせず、各自がコピーして使う）を基本とします。

> 📋 **コピペで使えるサンプルが欲しい人は [settings-examples.md](./settings-examples.md) へ。**
> 本ドキュメントは考え方、サンプル集は実物のテンプレートを提供します。

---

## 1. 設定は4階層ある

Claude Code の設定はこの優先順位で重なります。共有戦略はこの階層に沿って決めます。

| # | 階層 | 場所 | 共有方法 | 中身 |
|---|------|------|----------|------|
| ① | Enterprise（管理者ポリシー） | mac: `/Library/Application Support/ClaudeCode/managed-settings.json` | MDM/Jamf等で配布。**上書き不可** | 組織として禁止事項を強制したい場合 |
| ② | User（個人・**マシン全体**） | `~/.claude/settings.json` | **テンプレとして配布**→各自コピー | 個人の好み・通知・読み取り専用の許可 |
| ③ | Project（共有） | `<repo>/.claude/settings.json` | **git にコミット** | プロジェクト共通ルール |
| ④ | Project（個人） | `<repo>/.claude/settings.local.json` | **gitignore**（共有しない） | 個人の例外許可（クラウドプロファイル等） |

優先順位は **① > ② > ④ > ③**（番号が小さいほど強い／local は共有 project より優先）。

---

## 2. 共有可否の早見表

| 対象 | 共有可否 | 配り方 |
|------|----------|--------|
| プロジェクトの `.claude/settings.json`（permissions / hooks） | ✅ 共有する | git コミット |
| `CLAUDE.md`（規約・非自明コマンド・コーディング標準） | ✅ 共有する | git コミット |
| `.claude/agents` / `commands` / `skills` | ✅ 共有する | git コミット |
| `.mcp.json`（プロジェクトで使う MCP 定義） | ✅ 共有する | git コミット |
| `~/.claude/settings.json`（マシン全体・個人） | ⚠️ テンプレのみ | サンプルを配布→各自コピー |
| PreToolUse 等の hook スクリプト | ⚠️ テンプレのみ | 絶対パス禁止・要レビュー |
| APIキー / トークン / `ANTHROPIC_API_KEY` 等 env | 🚫 共有禁止 | 各自設定 |
| 絶対パス（`/Users/<name>/...`）入りの設定 | 🚫 共有禁止 | `$CLAUDE_PROJECT_DIR` / `$HOME` を使う |
| `settings.local.json` / `projects/` / `history.jsonl` / `sessions/` / cache | 🚫 共有禁止 | 個人ローカルのみ |

---

## 3. ✅ プロジェクト共有設定（`.claude/settings.json`）

git にコミットして全員に配る部分。

### permissions.deny はチーム標準にする価値が高い

事故防止（破壊的コマンド・秘密情報の読み取り）を全員に効かせます。

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(git push --force:*)",
      "Bash(git push -f:*)",
      "Bash(chmod 777:*)",
      "Read(.env)",
      "Read(.env.*)",
      "Read(**/*.pem)",
      "Read(**/*.key)"
    ]
  }
}
```

### permissions.allow はプロジェクト固有の安全コマンドを並べる

許可プロンプトを減らし、全員の体験を揃えます。**そのプロジェクトで頻出かつ安全**なコマンドだけを入れます。

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run lint:*)",
      "Bash(npm run test:*)",
      "Bash(npm run typecheck:*)",
      "Bash(npm run build:*)",
      "Bash(bin/rspec:*)",
      "Bash(bin/rubocop:*)",
      "Bash(git push:*)",
      "Bash(ls:*)",
      "Bash(find:*)"
    ]
  }
}
```

### hooks で保存時 lint/format を全員に強制

レビューコスト削減に直結します。例として、Edit/Write 後にコードフォーマッタを自動実行する hook を仕込めます。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "FILE_PATH=$(echo \"$TOOL_INPUT\" | jq -r '.file_path // empty'); [[ \"$FILE_PATH\" == *.rb ]] && cd \"$CLAUDE_PROJECT_DIR\" && bin/rubocop -A \"$FILE_PATH\" > /dev/null 2>&1; exit 0"
          }
        ]
      }
    ]
  }
}
```

> **重要:** hook スクリプト内は必ず `$CLAUDE_PROJECT_DIR` / `$HOME` を使い、絶対パスを書かないこと（他人の環境で壊れます）。

---

## 4. ⚠️ マシン全体（個人）設定テンプレ（`~/.claude/settings.json`）

各自のマシンにコピーして使う雛形。**強制せず推奨として配布**します。

```json
{
  "permissions": {
    "allow": [
      "Bash(git log:*)", "Bash(git show:*)", "Bash(git diff:*)",
      "Bash(git status:*)", "Bash(git branch:*)", "Bash(git blame:*)",
      "Bash(ls:*)", "Bash(cat:*)", "Bash(head:*)", "Bash(tail:*)",
      "Bash(grep:*)", "Bash(rg:*)", "Bash(fd:*)", "Bash(find:*)",
      "Bash(jq:*)", "Bash(wc:*)", "Bash(which:*)", "Bash(pwd:*)",
      "Bash(gh pr view:*)", "Bash(gh pr diff:*)", "Bash(gh pr list:*)",
      "Bash(gh issue view:*)", "Bash(gh run view:*)", "Bash(gh auth status:*)"
    ],
    "deny": [
      "Bash(sudo:*)"
    ]
  }
}
```

> ポイント: User 階層には「**どのプロジェクトでも安全な読み取り専用コマンド**」だけを置きます。
> プロジェクト固有のビルド/テストコマンドは各リポジトリの `.claude/settings.json`（③）側に置くこと。

### 個人差が出る項目（テンプレで例示するだけ・強制しない）

| キー | 内容 |
|------|------|
| `theme` | 配色（例: `dark-ansi`） |
| `effortLevel` | 思考の深さ（例: `high`） |
| `alwaysThinkingEnabled` | 常時 thinking |
| `enabledPlugins` | 有効化するプラグイン |
| `hooks.Notification` / `hooks.Stop` | 完了/許可待ちの macOS 通知（`osascript`） |

macOS 通知 hook の例（任意）:

```json
{
  "hooks": {
    "Stop": [
      { "matcher": "", "hooks": [
        { "type": "command",
          "command": "osascript -e 'display notification \"タスクが完了しました\" with title \"Claude Code\" sound name \"Hero\"'" }
      ]}
    ]
  }
}
```

---

## 5. 🚫 共有してはいけないもの

- **APIキー・トークン・各種シークレット**（env を含む）
- **絶対パス入りの設定**（`/Users/<name>/...`）。hook では `$CLAUDE_PROJECT_DIR` / `$HOME` を使う
- `~/.claude/` 配下の **個人作業履歴**: `projects/`, `history.jsonl`, `sessions/`, `cache/`, `shell-snapshots/`, `stats-cache.json` など
- 各リポジトリの **`.claude/settings.local.json`**（gitignore 済みであることを確認）

---

## 6. 導入手順（新メンバー向け）

1. Claude Code CLI と GitHub CLI (`gh`) をインストール・認証
2. 上記 **§4 のテンプレ**を `~/.claude/settings.json` にコピー（好みに応じて調整）
3. リポジトリを clone すれば `.claude/settings.json` / `CLAUDE.md` は自動で読み込まれる
4. 必要なプラグイン・MCP を `claude plugins add` でインストール

---

## 7. 設定の置き場所ガイド

新しい情報をどこに置くかの判断基準です。

```
新しい情報を追加したい
  │
  ├─ コードや git から読み取れる？ → 追加しない
  │
  ├─ 全タスクで毎回必要？ → CLAUDE.md
  │
  ├─ 特定ディレクトリの作業で毎回必要？ → サブディレクトリの CLAUDE.md
  │
  ├─ 特定の作業フローで必要？
  │   ├─ ユーザーが起動 → command
  │   ├─ Claude が判断して起動 → agent
  │   └─ agent の作業中に参照 → skill
  │
  ├─ たまに深く参照する？ → docs/ に置いて CLAUDE.md からリンク
  │
  └─ 会話をまたいで覚えておく？ → Memory
```

> **ポイント:** 常時読み込みのコンテキスト（CLAUDE.md）はトークンコストに直結します。
> 「頻度 × 重要度」が高いものだけを CLAUDE.md に置き、それ以外は skill や docs に分離してください。

---

## 8. 運用のベストプラクティス

- **deny ルールはチームで揃える**（破壊的コマンド・秘密情報 Read）。事故防止効果が最も高い
- **allow は「安全で頻出」だけ**を入れる。判断が要るコマンドは allow しない（都度確認させる）
- **CLAUDE.md は200行以下を目安**に。詳細は skill / docs に逃がす
- **hook・設定に絶対パスを書かない**（他人のマシンで壊れる）
- **User 階層 = 横断的に安全なもの / Project 階層 = そのリポジトリ固有 / local 階層 = 個人の例外** という住み分けを守る
- テンプレ更新時はこのドキュメントを更新し、周知する
