# AGENTS.md 読み込み検証結果

- `起動時にプロジェクト指示が渡されていたか: いいえ`
- `渡されていた場合のその1行目: （渡されていないため該当なし）`

検証日: 2026-09-19 / セッション: `cc4cef03-c6e0-54fe-bbc4-ce71d96efe79`

---

## 結論

**CLAUDE.md が存在せず AGENTS.md のみが存在し、`.claude/settings.json` で
`agents-md@builtin` の `instructionFiles: "claude-md-or-agents-md"` が指定されていても、
このセッション（claude 2.1.278 / Claude Code on the web のリモート実行環境）では
AGENTS.md はプロジェクト指示として起動時コンテキストに渡されなかった。**

根拠は3点とも独立に一致している。

1. 自己申告: ファイルを読む前の起動時コンテキストに AGENTS.md の内容は無かった
2. transcript: `attachment.type == "instructions"` のレコードが **0 件**
3. デバッグログ: agents-md プラグインのロード痕跡が **一切無い**（プラグイン数 0）

AGENTS.md の内容自体は「AGENTS.md が読めた場合は『OK』と返事してください」という指示だが、
これは起動時に渡されておらず、本タスクの手順2で `cat` した結果として初めて読んだものである。
よって「OK」は返していない（読み込み機構としては失敗）。

---

## ステップ1: 起動時コンテキストの内容（ツール使用前の自己申告）

起動時に渡されていたのは以下のみ。

- リモート実行環境の説明（コンテナ、ディスク、Chromium、agent proxy）
- GitHub 連携ルール（`gh` CLI 不可 / `mcp__github__*` を使う）
- リポジトリスコープ: `yoshihir0-code/test`
- git 運用ルール（push リトライ、PR を勝手に作らない等）
- ユーザー設定: 「必ず日本語で返答してください。」
- commit / PR の attribution 指定
- モデル識別情報（`claude-opus-5`）

**プロジェクト指示（AGENTS.md / CLAUDE.md 由来の本文）は含まれていなかった。**

---

## ステップ2: 実測データ

### 1. `claude --version`

```
2.1.278 (Claude Code)
```

### 2. `ls -la /home/user/test`

```
total 20
drwxr-xr-x 4 root root 4096 Sep 19 21:01 .
drwxr-xr-x 3 root root 4096 Sep 19 21:01 ..
drwxr-xr-x 2 root root 4096 Sep 19 21:01 .claude
drwxr-xr-x 8 root root 4096 Sep 19 21:01 .git
-rw-r--r-- 1 root root  163 Sep 19 21:01 AGENTS.md
```

CLAUDE.md は存在せず、AGENTS.md のみ存在することを確認。
（`git log`: `c1a2279 Delete CLAUDE.md` / `b506caf Update instructionFiles in settings.json`）

参考 — `cat AGENTS.md`:

```
これはCLAUDE.mdが存在しない場合、AGENTS.mdを読み込むかのテストです。
AGENTS.mdが読めた場合は「OK」と返事してください。
```

### 3. `cat /home/user/test/.claude/settings.json`

```json
{
  "pluginConfigs": {
    "agents-md@builtin": {
      "options": { "instructionFiles": "claude-md-or-agents-md" }
    }
  }
}
```

設定自体は意図どおり書かれている。

### 4. `grep -n "Registered .* hooks from .* plugins" /tmp/claude-code.log`

```
131:2026-09-19T21:01:20.162Z [DEBUG] Registered 0 hooks from 0 plugins
308:2026-09-19T21:01:20.898Z [DEBUG] Registered 0 hooks from 0 plugins
```

**0 hooks / 0 plugins。** プラグイン経由の hook は一つも登録されていない。

### 5. `grep -n -i "agents-md" /tmp/claude-code.log | head -20`

初回実行時点では **マッチ 0 件**。

その後の再実行では 1 件マッチしたが、これは本検証で自分が実行した grep コマンド文字列が
`Permission suggestions for Bash` のログに含まれたことによる**自己言及の偽陽性**であり、
プラグインのロード痕跡ではない。

```
488:... [DEBUG] "Permission suggestions for Bash: [...\"ruleContent\": \"grep -n -i \\\"agents-md\\\" /tmp/claude-code.log\"...]"
```

すなわち **agents-md プラグインがロードされた痕跡はログ上に存在しない。**

### 6. `grep -n "GrowthBook cache" /tmp/claude-code.log | head -5`

```
129:2026-09-19T21:01:20.160Z [DEBUG] installed plugins' hooks modules not loaded: rollout flag (tengu_plugin_hooks_modules) is off, from the default (a cold GrowthBook cache, no payload yet); built-in plugins load regardless
306:2026-09-19T21:01:20.898Z [DEBUG] installed plugins' hooks modules not loaded: rollout flag (tengu_plugin_hooks_modules) is off, from the default (a cold GrowthBook cache, no payload yet); built-in plugins load regardless
```

`tengu_plugin_hooks_modules` が **cold GrowthBook cache による既定値で off**。
ログは "built-in plugins load regardless"（built-in は関係なくロードされる）と述べているが、
実際には次の行で built-in の一つが seat されていない:

```
130: sec-default@builtin not seated: installed plugins' hooks modules are off in this process
307: sec-default@builtin not seated: installed plugins' hooks modules are off in this process
```

`agents-md@builtin` については seated / not seated いずれのログも出ていない。

### 7. GrowthBook フラグ `tengu_agents_md_mod`

```
$ python3 -c "import json;d=json.load(open('/root/.claude.json'));print(d.get('cachedGrowthBookFeatures',{}).get('tengu_agents_md_mod','<ABSENT>'))"
True
```

**フラグ自体は True**（キャッシュ上は有効）。にもかかわらず読み込まれていない。

### 8. セッション transcript の `attachment` 走査

transcript: `/root/.claude/projects/-home-user-test/cc4cef03-c6e0-54fe-bbc4-ce71d96efe79.jsonl`（44 レコード）

`attachment` の `type` 別件数:

| type | 件数 |
|---|---|
| session_context | 1 |
| date | 1 |
| remote_session_change | 1 |
| prompt_snapshot | 2 |
| environment | 1 |
| model | 1 |
| deferred_tools_delta | 1 |
| agent_listing_delta | 1 |
| mcp_instructions_delta | 1 |
| skill_listing | 1 |
| auto_mode | 1 |
| total_tokens_reminder | 3 |
| deferred_tools_record | 1 |

**`type == "instructions"` のレコードは 0 件。** したがって `files[].path` も出力なし
（AGENTS.md も CLAUDE.md も指示ファイルとして添付されていない）。

補足: ログ内に `claude.md`（大文字小文字問わず）への言及も 0 件。

---

## 補足: プラグインサブシステムの状態（ログ抜粋）

```
 85: installed_plugins.json doesn't exist, returning empty V2 object
 86: Found 0 plugins (0 enabled, 0 disabled)
102: getPluginSkills: Processing 0 enabled plugins
108: Total plugin skills loaded: 0 (0 duplicate/user-owned entries skipped)
114: Total plugin agents loaded: 0
131: Registered 0 hooks from 0 plugins
137: getSkills returning: 1 skill dir commands, 0 plugin skills, 40 bundled skills, 0 builtin plugin skills
147: Initialized versioned plugins system with 0 plugins
233: installPluginsForHeadless: starting
234: installPluginsForHeadless: no marketplaces declared
179: rg error (... /root/.claude/plugins/cache: No such file or directory ...)
```

`0 builtin plugin skills` が全ての `getSkills` 行で一貫している。
`/root/.claude/plugins/cache` も存在しない。

---

## 観察された事実の整理

| 項目 | 実測値 |
|---|---|
| CLI バージョン | 2.1.278 |
| CLAUDE.md | 存在しない |
| AGENTS.md | 存在する（163 bytes） |
| `.claude/settings.json` の `instructionFiles` | `claude-md-or-agents-md`（設定済み） |
| GrowthBook `tengu_agents_md_mod` | `True` |
| GrowthBook `tengu_plugin_hooks_modules` | off（cold cache の既定値） |
| ログ内の agents-md ロード痕跡 | なし（偽陽性1件のみ） |
| 登録された plugin hooks | 0 hooks / 0 plugins |
| builtin plugin skills | 0 |
| transcript の `instructions` attachment | 0 件 |
| 起動時コンテキストに AGENTS.md 本文 | **なし** |

設定とフィーチャーフラグは揃っているが、このリモート実行環境のプロセスでは
プラグインサブシステム自体が 0 プラグイン状態で初期化されており、
`agents-md@builtin` が指示ファイル注入を行う経路が成立していなかった、という整合的な説明になる。
ただし `agents-md@builtin` に関する肯定・否定いずれのログ行も出ていないため、
「なぜ seat されなかったか」の直接証拠はログからは取れていない（推測は避ける）。
