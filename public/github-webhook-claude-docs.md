---
title: GitHub Webhook + LambdaでLLM処理を動かすときに考えたこと
tags:
  - AWS
  - lambda
  - 個人開発
  - 生成AI
  - Claude
private: false
updated_at: '2026-06-15T12:00:00+09:00'
id: 716e37636e720a47a4ff
organization_url_name: null
slide: false
ignorePublish: false
---

GitHub Webhookを受け取って、LLMに何か処理させて、結果をどこかに書き出す、というパターンのシステムを作った。コンセプト自体は単純で最初はすぐ動いた。プロダクションとして運用するにはいくつか考えないといけないことがあったので、実装した内容を残しておく。

## Webhook署名を検証しないと誰でもリクエストを投げ放題になる

GitHubは`X-Hub-Signature-256`ヘッダーにHMAC-SHA256の署名を付けてWebhookを送ってくる。これを検証しないと、外部から偽のpushイベントを投げてLLM処理を無限に起動できてしまう。

```python
import hashlib
import hmac

def verify_github_signature(payload: bytes, signature: str, secret: str) -> bool:
    expected = "sha256=" + hmac.new(
        secret.encode(), payload, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

`==`ではなく`hmac.compare_digest`で比較するのはタイミング攻撃への対策。Webhookエンドポイントに到達したら最初にここを通して、合わなければ即401を返す。

## リポジトリのファイルを全部LLMに渡せるのは小規模だけ

最初は「全ファイルを読んでLLMに渡せばいい」という実装にしていた。ファイルが増えると3つの壁にぶつかる。

- GitHubのAPIレート制限（1時間5000リクエスト）に引っかかる
- 全ファイルを連結するとコンテキストウィンドウを超える
- Lambdaがタイムアウトする（最大15分）

ファイル総量で処理を2パターンに分けた。

```python
SINGLE_PASS_CHAR_LIMIT = 400_000  # これ以下なら1パスで処理
CHUNK_SIZE = 30                   # 2パス処理時のファイル数/チャンク
```

**小〜中規模（1パス）**: ファイル内容を全部連結してLLMへ渡す。シンプルで品質も高い。

**大規模（2パス）**:
- Pass 1: ファイルの「パス一覧だけ」を渡して「この処理に関係するファイルはどれ？」を聞く
- Pass 2: 絞り込んだファイルの中身だけを渡して実際の処理をする

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

# Pass 2: 絞り込んだファイルの中身だけで処理
source = read_files(relevant_files)
result = call_claude_heavy(source, task="api_spec")
```

ファイルパスだけでも「このバグは`routes/auth.py`と`models/user.py`あたりだろう」くらいの絞り込みはできる。全部読ませるよりトークンコストが大幅に下がる。

## pushのたびに全量処理するとAPIコストが跳ね上がる

PR mergeのときは`commits`の差分から変更ファイルが明確にわかる。それを使って変更のあったドキュメントだけ再生成するようにした。

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

日常的な小さな変更はdiff更新、大きなリファクタリングは全体再生成、という切り替え。通常のAPIコストが体感でかなり変わる。

## GitHubのWebhookは同じイベントを複数回送ってくることがある

ネットワークエラーでGitHubがWebhookを再送することがある。同じpushイベントが2回来ても処理が2回走らないよう、`X-GitHub-Delivery`ヘッダー（イベントごとのユニークID）をDBに記録して冪等性を担保した。

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

「処理中」状態のまま失敗したエントリが残ると次の処理が走れなくなるので、一定時間後に再処理可能にするタイムアウトも設けた。

## LLM処理とWebhook受信を同じLambdaでやるとタイムアウトする

ドキュメントの種類が増えてくると、LLMの呼び出しが積み重なって数分かかることがある。Webhookを受け取るLambdaで同期処理すると余裕でタイムアウトする（Lambdaのタイムアウトはデフォルト3秒、最大15分だが、GitHubがWebhookのタイムアウトとみなす時間はもっと短い）。

Webhook受信LambdaはSQSにメッセージを積んで即202を返すだけにして、実際の処理は別のLambdaに分けた。

```
Webhook Lambda（数秒で即返却）
  → SQS にキューイング → 即 202 返却

Generator Lambda（最大15分まで使える）
  → SQS からメッセージを取り出して処理
```

GitHubから見ると即座にレスポンスが返るし、処理側はタイムアウトを気にせず動ける。

---

署名検証と冪等性は最初からやっておかないと後で直すのが面倒。コンテキスト設計はリポジトリの規模が変わると大きく変わるので、最初は1パスで動かして、遅くなったら2パス化するくらいでよかった。
