# Claude Code 設定サンプル集

実運用ベースの **コピペで使える `settings.json` / `settings.local.json` のサンプル**です。
考え方の全体像は [team-sharing.md](./team-sharing.md) を参照してください。

> **使い方**: 下のサンプルをコピーして、`<your-name>` などのプレースホルダを自分の環境に合わせて置き換えるだけです。

---

## どこに何を置くか（30秒で分かる版）

| ファイル | 置く場所 | git管理 | 入れるもの |
|---|---|---|---|
| ユーザー設定 | `~/.claude/settings.json` | ❌ 各自 | どのプロジェクトでも安全な読取コマンド・通知 |
| プロジェクト共有 | `<repo>/.claude/settings.json` | ✅ コミット | 共通の禁止ルール＋そのリポの安全コマンド |
| プロジェクト個人 | `<repo>/.claude/settings.local.json` | ❌ gitignore | 絶対パス・個人env・plugin ON/OFF |

**鉄則**
- 🚫 **シークレット**（パスワード・トークン・APIキー）は共有ファイルに書かない → `settings.local.json` へ
- 🚫 **絶対パス**（`/Users/...`）は共有ファイルに書かない → `settings.local.json` か、`**/` パターンで一般化
- ✅ **deny は揃える**（破壊的コマンド・秘密情報の読取禁止）。事故防止効果が最大

---

## ① ユーザー設定 `~/.claude/settings.json`（各自のマシン全体）

「どのプロジェクトでも安全な読取専用コマンド」だけを許可します。これを入れておくと許可ダイアログが激減します。

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

好みで足せる任意項目（強制しない）:

```jsonc
{
  "theme": "dark-ansi",           // 配色
  "effortLevel": "high",          // 思考の深さ
  "alwaysThinkingEnabled": true,  // 常時 thinking
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

## ② プロジェクト共有 `settings.json`（git にコミット）

「共通の禁止ルール」＋「そのリポジトリの安全コマンド」を入れます。**全員に同じ安全策が効く**のが利点です。

```json
{
  "enableAllProjectMcpServers": true,
  "permissions": {
    "allow": [
      "Bash(git push:*)",
      "Bash(git fetch:*)",
      "Bash(git commit:*)"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(git push --force:*)",
      "Bash(git push -f:*)",
      "Bash(chmod 777:*)",
      "Bash(sudo:*)",
      "Read(.env)",
      "Read(.env.*)",
      "Read(**/*.pem)",
      "Read(**/*.key)",
      "Read(**/credentials.yml.enc)",
      "Read(**/master.key)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "CMD=$(echo \"$TOOL_INPUT\" | jq -r '.command // empty'); if echo \"$CMD\" | grep -Eq 'git push .*(origin )?(main|master)\\b'; then echo '{\"block\": true, \"reason\": \"mainへの直接pushは禁止。PRを作成してください。\"}'; exit 2; fi; exit 0"
          }
        ]
      }
    ]
  }
}
```

### `allow` に足すプロジェクト固有コマンド（言語別）

そのリポでよく使う **安全な** コマンドだけを追加します。

```jsonc
// Rails（bin/ 運用）
"Bash(bin/rspec:*)", "Bash(bin/rubocop:*)", "Bash(bin/rails:*)", "Bash(bin/rake:*)", "Bash(bin/brakeman:*)"

// Rails（bundle 運用）
"Bash(bundle exec rspec:*)", "Bash(bundle exec rubocop:*)"

// フロントエンド（npm）
"Bash(npm run lint:*)", "Bash(npm run test:*)", "Bash(npm run typecheck:*)",
"Bash(npm run build:*)", "Bash(npm run codegen:*)"

// モノレポ（サブディレクトリ運用）
"Bash(cd back && bin/rspec:*)", "Bash(cd front && npm run test:*)"
```

### 保存時に自動フォーマットする hook（任意・おすすめ）

Edit/Write のたびに整形され、レビュー指摘が減ります。**絶対パスを書かず `$CLAUDE_PROJECT_DIR` を使う**のがポイント。

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

---

## ③ プロジェクト個人 `settings.local.json`（gitignore・各自）

絶対パス・個人の env・plugin の ON/OFF はここに書きます。**git には乗せません。**

```json
{
  "enableAllProjectMcpServers": true,
  "permissions": {
    "allow": [
      "Bash(gh pr view:*)",
      "Bash(gh pr diff:*)",
      "Read(/Users/<your-name>/.rbenv/versions/**)",
      "Read(/opt/homebrew/bin/**)"
    ]
  },
  "env": {
    "MYSQL_HOST": "localhost",
    "MYSQL_USERNAME": "root",
    "MYSQL_PASSWORD": "your-local-password"
  }
}
```

> **注意点**
> - 絶対パスは先頭スラッシュ **1つ**（`/Users/...`）。`//Users/...` のように 2 つにするとマッチしません。
> - `MYSQL_PASSWORD` などの接続情報は **この local ファイルだけ**に。共有 `settings.json` には絶対に書かないこと。
> - `.gitignore` に `.claude/settings.local.json` が入っているか必ず確認。

---

## 導入チェックリスト

- [ ] `~/.claude/settings.json` に ① のユーザー設定をコピーした
- [ ] 担当リポの `.claude/settings.json` に ② の共有設定（deny＋PreToolUse）が入っている
- [ ] 個人固有の許可・env は `.claude/settings.local.json`（③）に分離した
- [ ] 共有ファイルに **シークレット・絶対パスが混入していない**
- [ ] `.gitignore` に `.claude/settings.local.json` がある
- [ ] 絶対パスの先頭スラッシュが 1 つになっている

---

## よくある間違い

| 症状 | 原因 | 対処 |
|---|---|---|
| local の Read 許可が効かない | 先頭スラッシュが `//`（2つ） | `/Users/...` に直す |
| 他人の環境で hook が壊れる | 絶対パスをハードコード | `$CLAUDE_PROJECT_DIR` / `$HOME` を使う |
| 秘密情報が git に乗った | `env` を共有 `settings.json` に書いた | `settings.local.json` に移し、履歴から除去 |
| 許可ダイアログが多すぎる | 安全コマンドが allow されていない | ① のユーザー設定 or ② の allow に追加 |
| `gh` 経由で意図せず書き込み | `gh:*` で全許可している | `gh pr view` 等の read 系に絞る |
