# claude-code-notebooklm

Claude Code で [notebooklm-py](https://github.com/teng-lin/notebooklm-py) を使うためのスキルです。  
Google NotebookLM のノートブック作成・ソース追加・ポッドキャスト生成などをClaude Codeから直接操作できます。

## 必須要件

- Python 3.10 以上
- Google Account（NotebookLM の利用に必要）
- Claude Code

## インストール

パッケージマネージャーは [uv](https://docs.astral.sh/uv/) を推奨します。依存関係の管理がシンプルになります。  
`notebooklm-py` は複数プロジェクトをまたいで使えるようグローバルにインストールします。

```bash
# 1. notebooklm-py をグローバルにインストール
uv tool install "notebooklm-py"

# 2. ブラウザをインストール
playwright install chromium
```

### スキルの配置

このリポジトリをクローンし、`notebooklm` フォルダを Claude Code のスキルディレクトリに配置します。

```bash
git clone https://github.com/water-public/claude-code-notebooklm.git
```

クローン後、`notebooklm` フォルダを以下のパスにコピーします。

**Windows:**
```
C:\Users\<ユーザー名>\.claude\skills\notebooklm\
```

**Mac / Linux:**
```
~/.claude/skills/notebooklm/
```

配置後の構造:
```
.claude/
└── skills/
    └── notebooklm/
        └── SKILL.md
```

## 初期設定（Google ログイン）

```bash
notebooklm login
```

ブラウザが開くので Google アカウントでログインします。完了後、スキルが使えるようになります。

動作確認:
```bash
notebooklm status
```

## 使い方

Claude Code で `/notebooklm` を呼び出すか、意図を伝えるだけで起動します。

**例:**
```
/notebooklm を用いて x86 architecture と Arm architecture の比較レポートを作りたい。
パフォーマンス、コスト等の複合視点で比較し、トレーニング・推論・エージェントの
それぞれのフェーズでの役割も整理したレポートを作成して。
```

## 対応機能

- ノートブックの作成・管理
- ソース追加（URL、YouTube、PDF、音声・動画・画像ファイル）
- Web リサーチ（fast / deep モード）
- チャット（Q&A）
- ポッドキャスト・動画・スライド・インフォグラフィック・レポート・クイズ・フラッシュカード・マインドマップの生成
- 各種フォーマットでのダウンロード

詳細なコマンドリファレンスはスキルファイル [`notebooklm/SKILL.md`](notebooklm/SKILL.md) を参照してください。

## オリジナルからの変更点（Windows 対応）

本スキルは [teng-lin/notebooklm-py の SKILL.md](https://github.com/teng-lin/notebooklm-py/blob/main/SKILL.md) をベースに、**Windows（cp932 エンコーディング環境）での実運用で発見した問題への対処**を追加したものです。

### 1. クエリは英語必須・結果を日本語に翻訳

Windows の cp932 ターミナルでは、日本語クエリが `UnicodeEncodeError` を引き起こします。  
スキルはすべての `notebooklm ask` クエリを英語で発行し、取得後に日本語へ翻訳するルールを追加しています。

### 2. `notebooklm ask` の安全なパターン（`--json` リダイレクト必須）

`--json` なしの plain text 出力に含まれる em-dash（`—`）等の非cp932文字がターミナルをクラッシュさせます。  
またフルの `--json` 出力（200〜600KB）は Read ツールの上限を超えるため、以下の2ステップパターンを追加しています：

```bash
# ① --json でファイルに書き出す
notebooklm ask "query" --json > C:/temp/output.json

# ② answer フィールドだけを UTF-8 で抽出
python -c "
import json
with open('C:/temp/output.json', encoding='utf-8') as f:
    d = json.load(f)
with open('C:/temp/output_answer.txt', 'w', encoding='utf-8') as f:
    f.write(d.get('answer', ''))
print('saved')
"
# ③ output_answer.txt を Read ツールで読む
```

### 3. 重複ソースの一括削除パターン

`notebooklm source delete` はデフォルトで対話プロンプトが出るため、bashループで止まります。  
`--yes` フラグ + Python subprocess を使った一括削除パターンを追加しています。

### 4. `source add-research` は `--no-wait` 必須

`--no-wait` なしで実行すると、リサーチ完了後の表示処理でcp932クラッシュが発生します。  
クラッシュは**表示処理**で起きるため、インポート自体は完了している場合があり、その後 `research wait --import-all` を再実行すると**同じソースが重複インポート**されます。

追加した対処パターン：
- `--no-wait` を必須化し、表示とインポートを分離
- `research wait --import-all` は1タスクにつき**1回だけ**実行するルールを明記
- 失敗後のフォールバック（`source list` で確認 → 未インポート分を手動 `source add`）

### 5. エラーハンドリングの拡充

オリジナルにない以下のエラーケースを追記：

| 追加したエラー | 対処内容 |
|---|---|
| `UnicodeEncodeError: cp932` on `source add-research` | `--no-wait --json` パターンで回避 |
| `source wait` が exit code 1 で失敗 | 即削除して続行（リトライしない） |
| `research wait --import-all` が exit code 1 で失敗 | 再実行禁止・手動で未インポート分を追加 |
| Source `status: error`（アクセス拒否・404） | 即削除して続行 |
| ドメイン制限によるソース追加失敗 | `https://pure.md/` をURLの前置きで回避（Yahoo Finance 等で有効） |

---

## 依存関係

- [notebooklm-py](https://github.com/teng-lin/notebooklm-py) — NotebookLM の CLI ツール（本スキルはこのCLIをClaude Codeから利用するためのラッパーです）
