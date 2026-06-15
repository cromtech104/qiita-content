---
title: LLMにコード修正を生成させるとき、ファイルを全部渡してはいけない
tags:
  - Python
  - 個人開発
  - GitHubAPI
  - 生成AI
  - Claude
private: false
updated_at: '2026-06-14T21:00:09+09:00'
id: 86ac82d3807990af0b7a
organization_url_name: null
slide: false
ignorePublish: false
---

チケットの内容を読んでリポジトリのコードを修正し、GitHub Draft PRを自動作成するシステムを作っていた。最初の実装はシンプルで「チケットとリポジトリのファイルを全部LLMに渡してコードを出させる」だった。

小さいリポジトリでは動く。ファイルが増えてくると入力トークンが爆発して現実的じゃなくなった。無関係なファイルを大量に渡しても精度は上がらなかった。

もう一つ詰まったのがJSONパース。「JSON onlyで返して」と指示してもコードフェンスで包んで返してきたり、キー名の書き方が揺れたりする。本番でそのまま`json.loads()`に渡すとよく壊れた。

## ファイルのパス一覧だけ渡して絞り込んでから中身を渡す

2ステップに分けた。

**Step 1**: ファイルの「中身」は渡さず「パス一覧」だけを渡して、チケットに関係しそうなファイルを絞り込む。

**Step 2**: 絞り込んだファイルの中身だけを渡して、修正コードを生成する。

`routes/auth.py`や`models/user.py`などファイルパスを見るだけで「このバグはここあたりだろう」という判断はかなりできる。

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

## 差分をcommit_groups形式で返させると構造化しやすい

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

1つのcommit_groupが1コミットになる。機能とテストを別コミットにするなど、PRのコミット履歴を意味のある単位に分けられる。

`is_fixable: false`を返せるようにしているのが重要で、チケットの記述が曖昧すぎて実装方針が判断できない場合はPR作成をスキップしてチケットに「情報不足のためスキップしました」とコメントする。無理に生成させると意味不明な差分が作られるだけなので、スキップの判断も明示的に返させた。

## 「JSON only」と言ってもコードフェンスで包んで返してくる

ここが一番詰まった。`json.loads()`に渡したらエラー、というのを何度か経験してから対策した。

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

## キー名のスタイルがis_fixableだったりisFixableだったり揺れる

同じモデルでも試行によって`is_fixable`になったり`isFixable`になったりする。複数の候補を順番に試す。

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

`is_fixable`に文字列`"true"`が来ることもあるのでboolへの変換も入れている。

## LLMが返すファイルパスはそのまま信頼しない

AIが返すファイルパスをそのままディスクに書くのは危ない。チケットにユーザーが書いた内容がそのままLLMに渡っているので、パストラバーサル（`../../etc/passwd`など）が含まれている可能性がある。

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

絶対パス・ホームディレクトリ・URIスキーム・`..`を全部弾く。LLMが生成した値でも、入力にユーザー由来のものが含まれていれば信頼しない。

---

2ステップへの分割は「全部渡す」が破綻してから気づいた。JSONパースは最初から複数キー候補とコードフェンス除去を入れておけば後で悩まずに済む。
