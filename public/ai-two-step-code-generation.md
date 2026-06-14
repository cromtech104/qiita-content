---
title: チケットからDraft PRを自動生成する実装で詰まった2つのこと
tags:
  - Claude
  - Python
  - 生成AI
  - 個人開発
  - GitHubAPI
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

チケット管理システムにissueが来たら、リポジトリのコードを読んでGitHubにDraft PRを自動作成するシステムを作っている。

最初の実装は「チケットの内容とリポジトリのファイルを全部Claudeに渡して差分を生成させる」だった。小さいリポジトリでは動く。ファイルが増えると入力トークンが爆発して現実的じゃなくなった。関係ないファイルを大量に渡しても精度は上がらなかった。

もう一つ詰まったのがJSONパース。「JSON onlyで返して」と指示してもコードフェンスで包んで返してきたり、キー名のスタイルが揺れたりする。本番でそのまま`json.loads()`に渡すとよく落ちた。

この記事はこの2点について。

## ファイルを全部渡さずに2ステップに分ける

2ステップに分けた。

**Step 1**: ファイルの「中身」は渡さず「パス一覧」だけを渡して、チケットに関係しそうなファイルを絞り込む。

**Step 2**: 絞り込んだファイルの中身だけを渡して、修正コードを生成する。

ファイルパスを見るだけでも「このバグは`routes/auth.py`と`models/user.py`あたりだろう」という絞り込みはかなりできる。

```python
async def identify_relevant_files(self, issue: dict, file_tree: list[str]) -> list[str]:
    tree_text = "\n".join(file_tree)
    prompt = f"""## チケット

Summary: {issue.get('summary')}
Description:
{issue.get('description', '(no description)')}

## リポジトリのファイルツリー

{tree_text}

---

このチケットに対応するために必要なファイルを特定してください。

Return JSON only:
{{"files": ["path/to/file1.py", "path/to/file2.py"]}}

Rules:
- チケットに記載された内容に直接関係するファイルのみ選ぶ
- 新規ファイルが必要な場合は想定されるパスを含めてよい
- 最大15ファイル、関連度の高い順
"""
    response = await client.messages.create(
        model=MODEL, max_tokens=512,
        messages=[{"role": "user", "content": prompt}],
    )
    return self._parse_file_list(response.content[0].text)
```

「チケットに直接関係するファイルのみ」と明示しているのがポイント。これがないと「関係するかも」なファイルをどんどん追加してくる。`max_tokens=512`に絞っているのも、パスのリストだけ返せばいいので長い回答を許さないため。

## 修正コードをcommit_groups形式で返させる

Step 2ではファイル内容を渡して修正コードを生成する。差分を`commit_groups`という形式で返させている。

```json
{
  "is_fixable": true,
  "reasoning": "〇〇が原因のため△△を修正する",
  "commit_groups": [
    {
      "message": "feat: 注文履歴機能を追加",
      "files": [
        { "path": "order_history.py", "content": "変更後の完全なファイル内容" }
      ]
    },
    {
      "message": "test: 注文履歴のテストを追加",
      "files": [
        { "path": "test_order_history.py", "content": "テストコードの内容" }
      ]
    }
  ],
  "pr_description": "## 概要\n..."
}
```

1つのcommit_groupが1コミットになる。機能とテストを別コミットにしたり、独立した機能は分けたりするとPRのコミット履歴が読みやすくなる。

`is_fixable: false`も返せるようにしているのが重要で、チケットの記述が曖昧すぎて実装方針が判断できない場合にはPR作成をスキップして、チケットに「情報不足のためスキップしました」とコメントする。無理に生成させると変な差分が作られるだけなので、スキップの判断も明示的に返させている。

## AIのJSONレスポンスが仕様どおりに来ない

ここが一番詰まった。「JSON only」と指示してもコードフェンス（` ```json ... ``` `）で包んで返してくることがある。キー名のスタイルも`is_fixable`だったり`isFixable`だったり揺れる。

まずコードフェンスを除去して、先頭の`{`からJSONを探す。

```python
def _extract_json_object(self, text: str) -> dict | None:
    decoder = json.JSONDecoder()
    cleaned_text = self._strip_code_fence(text.strip())
    for index, char in enumerate(cleaned_text):
        if char != "{":
            continue
        try:
            data, _ = decoder.raw_decode(cleaned_text[index:])
        except json.JSONDecodeError:
            continue
        if isinstance(data, dict):
            return data
    return None

def _strip_code_fence(self, text: str) -> str:
    match = re.fullmatch(r"```(?:json)?\s*(.*?)\s*```", text, re.DOTALL | re.IGNORECASE)
    if match:
        return match.group(1).strip()
    return text
```

`json.loads()`ではなく`raw_decode()`を使っているのは、JSONの後ろに余分な文字があっても途中までパースできるから。

キー名は複数の候補を試す。

```python
IS_FIXABLE_KEYS     = ("is_fixable", "isFixable", "fixable")
REASONING_KEYS      = ("reasoning", "reason", "summary")
COMMIT_GROUP_KEYS   = ("commit_groups", "commitGroups")

def _get_first_present(self, data: dict, keys: tuple, default=None):
    for key in keys:
        if key in data:
            return data[key]
    return default
```

`is_fixable`に`"true"`（文字列）が来ることもあるのでboolへの変換も入れている。

## AIが返すファイルパスはそのまま使わない

AIが返すファイルパスをそのままディスクに書くのは危ない。チケットにユーザーが書いた内容がそのままAIに渡っているので、パストラバーサル（`../../etc/passwd`など）が含まれている可能性がある。

```python
def _normalize_path(self, path) -> str | None:
    if not isinstance(path, str):
        return None

    normalized_path = path.strip().replace("\\", "/")
    if not normalized_path:
        return None
    if normalized_path.startswith(("/", "~")):
        return None
    if re.match(r"^[a-zA-Z][a-zA-Z0-9+.-]*:", normalized_path):  # URI scheme
        return None

    parts = [part for part in normalized_path.split("/") if part]
    if not parts or any(part == ".." for part in parts):
        return None

    return "/".join(parts)
```

絶対パス・ホームディレクトリ・URIスキーム・`..`を全部弾く。AIが生成した値でも、入力にユーザー由来のものが含まれている場合は信頼しない。

---

2ステップへの分割は「全部渡す」が破綻してから気づいた。JSONパースは最初から複数キー候補とコードフェンス除去を入れておいた方が後が楽。
