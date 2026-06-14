---
title: GitHubのpushをトリガーにClaudeでドキュメントを自動更新する仕組みを個人で作った話
tags:
  - AWS
  - lambda
  - 個人開発
  - 生成AI
  - Claude
private: false
updated_at: '2026-06-14T21:00:10+09:00'
id: 716e37636e720a47a4ff
organization_url_name: null
slide: false
ignorePublish: false
---

GitHubのpushをトリガーに、リポジトリのドキュメントをClaudeで自動更新するSaaSを個人で作っている。

「WebhookでpushのイベントをLambdaで受け取って、Claudeに渡せばいいだけ」と最初は思っていた。実際に作ると、署名検証、大きいリポジトリへの対応、Webhookの重複処理、Lambdaのタイムアウト回避など、ちゃんと考えないといけないところがいくつかあった。詰まったところをまとめておく。

## Webhook署名の検証は最初にやる

GitHubは`X-Hub-Signature-256`ヘッダーに署名を付けてWebhookを送ってくる。これを検証しないと、誰でも偽のpushイベントを送り込める。

```python
import hashlib
import hmac

def verify_github_signature(payload: bytes, signature: str, secret: str) -> bool:
    expected = "sha256=" + hmac.new(
        secret.encode(), payload, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

`==`で比較しないのはタイミング攻撃の対策で、`hmac.compare_digest`は常に一定時間で比較する。これはWebhookエンドポイントで最初にやること。署名が合わなければ即401を返す。

## リポジトリが大きいとそのまま全部渡せない

最初は「全ファイルを読んでClaudeに渡せばいい」という実装にしていた。小さいリポジトリでは動く。ファイル数が増えると詰まる。

- GitHubのAPIレート制限（1時間5000リクエスト）に引っかかる
- 全ファイルを連結するとClaudeのコンテキストを超える
- Lambdaがタイムアウトする（最大15分）

なのでファイル数で処理を2パターンに分けた。

```python
SINGLE_PASS_CHAR_LIMIT = 400_000  # これ以下なら1パスで処理
CHUNK_SIZE = 30                   # 2パス処理時のファイル数/チャンク
```

**小〜中規模（1パス）**: ファイル内容を全部連結してClaudeへ。シンプルで品質も高い。

**大規模（2パス）**:
- Pass 1（軽量モデル）: ファイルのパス一覧だけを渡して「このドキュメントに関係するファイルはどれ？」を聞く
- Pass 2（高性能モデル）: 絞り込んだファイルの中身だけを渡してドキュメントを生成する

```python
# Pass 1: ファイルツリー（パス一覧）だけ渡して絞り込む
file_tree = "\n".join(all_file_paths)
prompt = f"""
ファイルツリー:
{file_tree}

「API仕様書」の生成に必要なファイルをリストしてください。
最大15ファイル。
JSON: {{"files": ["path/to/file.py", ...]}}
"""
relevant_files = call_claude_light(prompt)

# Pass 2: 絞り込んだファイルの中身でドキュメントを生成
source = read_files(relevant_files)
doc = call_claude_heavy(source, doc_type="api_spec")
```

ファイルパスを見るだけでもかなり絞れる（`routes/auth.py`とか`models/user.py`とか）。全部読ませるよりずっと安い。

## PR mergeは差分だけ更新する

pushのたびに全ドキュメントを再生成していると、APIのコストが跳ね上がる。PR mergeのときは変更ファイルが明確にわかるので、それを使う。

```python
REGEN_FILE_THRESHOLD = 10  # 変更ファイル数がこれ以上 → 全体再生成

changed_files = get_pr_changed_files(pr_number)
if len(changed_files) >= REGEN_FILE_THRESHOLD:
    generate_all_docs(repo)
else:
    affected_docs = identify_affected_docs(changed_files)
    for doc_type in affected_docs:
        regenerate_doc(repo, doc_type, hint_files=changed_files)
```

日常の小さな変更はdiff更新、大きな変更は全体再生成という切り替え。これで通常のコスト感がかなり違う。

## 同じWebhookが2回来ることがある

GitHubのWebhookはネットワークエラーなどで再送されることがある。同じpushが2回来ても生成処理が2回走らないようにする必要があって、`X-GitHub-Delivery`（イベントごとのユニークID）をDBに記録して管理した。

```python
delivery_id = request.headers.get("X-GitHub-Delivery")

if is_already_processed(delivery_id):
    return {"status": "skipped"}

mark_as_processing(delivery_id)
try:
    run_pipeline(payload)
    mark_as_done(delivery_id)
except Exception as e:
    mark_as_failed(delivery_id)
    raise
```

「処理中」のまま失敗したエントリが残ると次の処理が走れなくなるので、一定時間後に再処理可能にするタイムアウトも持たせている。

## 受信と生成を分離する

ドキュメントが10種類あって、それぞれClaudeを呼ぶと数分かかることがある。LambdaはWebhookの受信もやっていて、同期処理にすると余裕でタイムアウトする。

なのでWebhookを受け取るLambdaはSQSに積んで即202を返すだけにして、実際の生成処理は別Lambdaにした。

```
Webhook Lambda（30秒でタイムアウト）
  → SQS に積んで即 202 返却

Generator Lambda（15分まで）
  → SQS からメッセージを取り出して処理
```

GitHubから見ると即座にレスポンスが返ってくるし、生成処理がどれだけかかっても詰まらない。

---

署名検証と冪等性だけは最初からやっておかないと後で直すのが面倒。コンテキスト設計はリポジトリの規模次第で全然変わってくるので、小さいリポジトリから始めて壊れたら考えるくらいでも良かった。
