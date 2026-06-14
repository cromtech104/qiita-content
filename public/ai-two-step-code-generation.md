---
title: AIにコード修正を生成させる2ステップpromptingと、JSONレスポンスを堅牢にパースする実装
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

チケット管理システムのissueを受け取り、対象リポジトリを読んでGitHubにDraft PRを自動作成するシステムを作った。

「AIにコードを書かせる」部分で詰まった点が2つある。**ファイル全部を渡すとコストとコンテキストが爆発すること**、**AIのJSONレスポンスが仕様どおりに来ない**ことだ。この記事ではその設計と実装を書く。

## やること

```
チケット（バグ・要望）
  → Step 1: ファイルツリー → 関連ファイルを特定
  → Step 2: 関連ファイルの内容 → 修正コードを生成
  → GitHub Draft PR作成
```

## なぜ2ステップか

最初は「チケットとリポジトリのファイルを全部渡してコードを生成させる」という実装にした。動いたが、問題が2つあった。

**コストが高い**: ファイル数が増えると入力トークンが爆発する。関係のないファイルを大量に渡しても精度は上がらない。

**コンテキスト制限**: 大きなリポジトリでは全ファイルを連結するとコンテキスト制限を超える。

解決策として2ステップに分けた。Step 1ではファイルの「中身」は渡さず「ファイルパスの一覧」だけを渡す。それだけでも「このチケットに関係しそうなファイル」はかなり絞れる。

## Step 1: ファイルツリーから関連ファイルを特定

```python
async def identify_relevant_files(
    self, issue: dict, file_tree: list[str]
) -> list[str]:
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

ポイントはプロンプトに「**チケットに記載された内容に直接関係するファイルのみ選ぶ**」と明示していること。これがないと「関係するかもしれない」ファイルを大量に返してくる。

max_tokensを512に絞っているのも意図的。ファイルパスのリストだけ返せばいいので、長い回答を許さない。

## Step 2: 関連ファイルの内容から修正コードを生成

Step 1で絞り込んだファイルだけを読み込んで、修正コードを生成する。

```python
async def generate_fix(
    self, issue: dict, file_contents: dict[str, str]
) -> FixResult:
    files_text = "\n\n".join(
        f"### {path}\n```\n{content}\n```"
        for path, content in file_contents.items()
    )
    prompt = f"""## チケット

Summary: {issue.get('summary')}
Description: {issue.get('description', '(no description)')}

## 関連ファイルの内容

{files_text}

---

Respond in JSON only:
{{
  "is_fixable": true,
  "reasoning": "実装方針の説明",
  "commit_groups": [
    {{
      "message": "feat: add order history feature",
      "files": [
        {{
          "path": "order_history.py",
          "content": "変更後の完全なファイル内容"
        }}
      ]
    }}
  ],
  "pr_description": "## 概要\\n..."
}}
"""
    response = await client.messages.create(
        model=MODEL, max_tokens=8096,
        messages=[{"role": "user", "content": prompt}],
    )
    return self._parse_fix_result(response.content[0].text)
```

### commit_groupsで論理的にコミットを分割させる

生成する差分を`commit_groups`という形式にしている。1つのcommit_groupが1コミットに対応する。

これにより：
- 機能追加とテストを別コミットにできる
- 独立した機能は別コミットに分けられる
- PRのコミット履歴が読みやすくなる

### is_fixable: やらない判断を明示する

`is_fixable: false`を返せる設計にしている。

チケットの記述が曖昧すぎて何を実装すべきか判断できない場合、AIは無理にコードを生成しなくていい。`is_fixable: false`と`reasoning`（スキップ理由）だけ返す。

呼び出し元はこれを見てPR作成をスキップし、チケット管理システムに「情報不足のためスキップしました」とコメントを残す。

## JSONレスポンスのパース

ここが一番詰まった。

AIは「JSON only」と指示してもコードフェンス（` ```json ... ``` `）で包んで返すことがある。また、キー名が仕様と微妙に違う（`is_fixable`ではなく`isFixable`など）ことがある。

### コードフェンスを除去する

```python
def _strip_code_fence(self, text: str) -> str:
    match = re.fullmatch(r"```(?:json)?\s*(.*?)\s*```", text, re.DOTALL | re.IGNORECASE)
    if match:
        return match.group(1).strip()
    return text
```

### 先頭の`{`からJSONをスキャンする

レスポンスの先頭に余分な文字列がついていることがある。先頭の`{`を探してそこからパースを試みる。

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
```

`json.loads()`ではなく`raw_decode()`を使うのは、JSONの後ろに余分な文字列があっても途中までパースできるため。

### 複数のキー名候補を受け付ける

```python
IS_FIXABLE_KEYS     = ("is_fixable", "isFixable", "fixable")
REASONING_KEYS      = ("reasoning", "reason", "summary")
PR_DESCRIPTION_KEYS = ("pr_description", "prDescription")
COMMIT_GROUP_KEYS   = ("commit_groups", "commitGroups")

def _get_first_present(self, data: dict, keys: tuple, default=None):
    for key in keys:
        if key in data:
            return data[key]
    return default
```

### booleanの文字列対応

```python
def _coerce_bool(self, value) -> bool | object:
    if isinstance(value, bool):
        return value
    if isinstance(value, str):
        normalized = value.strip().lower()
        if normalized in {"true", "yes", "1"}:
            return True
        if normalized in {"false", "no", "0"}:
            return False
    return value
```

## ファイルパスの検証

AIが生成したファイルパスをそのままディスクに書くのは危険。パストラバーサル（`../../etc/passwd`など）を防ぐ検証を入れる。

```python
def _normalize_path(self, path) -> str | None:
    if not isinstance(path, str):
        return None

    normalized_path = path.strip().replace("\\", "/")
    if not normalized_path:
        return None
    # 絶対パス・ホームディレクトリを拒否
    if normalized_path.startswith(("/", "~")):
        return None
    # URI scheme を拒否 (file://, http:// など)
    if re.match(r"^[a-zA-Z][a-zA-Z0-9+.-]*:", normalized_path):
        return None

    parts = [part for part in normalized_path.split("/") if part]
    # .. によるディレクトリ遡上を拒否
    if not parts or any(part == ".." for part in parts):
        return None

    return "/".join(parts)
```

AIが生成するパスをそのまま信頼しないのは、入力がユーザーのチケット（外部入力）を含んでいるから。チケットに`../../../../etc/passwd`のようなパスが書かれていた場合のことを考えると、検証は必須。

## まとめ

| 問題 | 対処 |
|------|------|
| 全ファイルを渡すとコスト爆発 | 2ステップ（ツリー→絞り込み→生成） |
| AIがJSONにコードフェンスをつける | 正規表現で除去 |
| キー名がスタイルによって揺れる | 複数の候補キーをフォールバックで試す |
| JSONの前後に余分な文字列がある | `raw_decode`で先頭`{`からスキャン |
| AIが書いたパスをそのまま使うと危険 | パストラバーサル検証を必ず入れる |
| 対応できないチケットに無理に対応させる | `is_fixable: false`で明示的にスキップ |

これらの設計をベースに、チケット管理システムのissueを起点にGitHub Draft PRを自動作成するサービスを作っている。
