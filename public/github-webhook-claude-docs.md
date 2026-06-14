---
title: GitHub pushをトリガーにClaudeでドキュメントを自動更新する仕組みを作った
tags:
  - Claude
  - Lambda
  - AWS
  - 個人開発
  - 生成AI
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

GitHubリポジトリのコードが変わるたびに、仕様書や設計書を自動で更新する仕組みを個人開発SaaSに組み込んだ。

やっていることはシンプルに見えるが、「リポジトリが大きい」「webhookは重複して来る」「APIコストが爆発する」という3つの問題に順番に詰まった。この記事はその設計メモです。

## 全体のフロー

```
GitHub push
  → GitHub App Webhook (HMAC署名付き)
  → Lambda (FastAPI + Mangum)
  → ファイル取得 (GitHub API)
  → Claude API でドキュメント生成
  → ドキュメントリポジトリへ push
```

PR merge時は同じフローの軽量版（差分ファイルのみ処理）が走る。

## 1. Webhook署名の検証

GitHubはwebhookリクエストに`X-Hub-Signature-256`ヘッダーを付けてくれる。受信側でこれを検証しないと、誰でも偽のpushイベントを送り込める。

```python
import hashlib
import hmac

def verify_github_signature(payload: bytes, signature: str, secret: str) -> bool:
    expected = "sha256=" + hmac.new(
        secret.encode(), payload, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

`hmac.compare_digest`はタイミング攻撃を防ぐための定数時間比較。文字列の`==`で比較すると不一致の位置によって処理時間が変わるため使わない。

署名が一致しない場合は即座に401を返す。これはWebhookを受け付けるAPIの最初の処理にする。

## 2. リポジトリが大きいと詰まる

最初は「ファイルを全部読んで一度にClaudeに渡す」という実装にしていた。小さいリポジトリでは動く。

問題は、コードが数百ファイルを超えたとき。

- GitHubのAPIレート制限（1時間あたり5000リクエスト）に引っかかる
- ファイル内容をすべて連結するとClaudeのコンテキスト制限を超える
- Lambdaのタイムアウト（最大15分）に引っかかる

解決策として、ファイル数で処理を2モードに分けた。

```python
SINGLE_PASS_CHAR_LIMIT = 400_000  # 文字数がこれ以下なら1パス処理
CHUNK_SIZE = 30                   # 2パス処理時のファイル数/チャンク
```

**1パス処理（小〜中規模）**: ファイル内容をすべて連結してClaudeに渡す。シンプルで品質が高い。

**2パス処理（大規模）**:
- Pass 1（軽量モデル）: ファイルツリーだけ渡して「このドキュメントに関係しそうなファイルはどれ？」を判定させる
- Pass 2（高性能モデル）: Pass 1で絞り込んだファイルだけを渡してドキュメントを生成する

Pass 1で全ファイルを読まずにファイルツリー（パス一覧）だけを渡すのがポイント。ファイルパスだけでもかなり絞り込める。

```python
# Pass 1: ファイルツリーから関連ファイルを特定
file_tree = "\n".join(all_file_paths)
prompt = f"""
リポジトリのファイルツリー:
{file_tree}

「API仕様書」を生成するために必要なファイルをリストしてください。
最大15ファイル、関連度の高い順で返してください。
JSON: {{"files": ["path/to/file.py", ...]}}
"""
relevant_files = call_claude_light(prompt)  # Haikuなどの軽量モデル

# Pass 2: 絞り込んだファイルの内容でドキュメントを生成
source = read_files(relevant_files)
doc = call_claude_heavy(source, doc_type="api_spec")  # Sonnetなど
```

ドキュメント種別（アーキテクチャ概要・API仕様・ER図・セットアップガイドなど）ごとにこの処理を繰り返す。

## 3. PR mergeは差分だけ処理する

pushイベントで毎回全ドキュメントを再生成すると、APIコストが跳ね上がる。

PR mergeのときは「変更されたファイル」が明確にわかる（GitHub APIの`/pulls/{id}/files`で取得できる）。変更ファイル数が少なければ、関連するドキュメントだけを差分更新すればいい。

```python
REGEN_FILE_THRESHOLD = 10  # 変更ファイル数がこれ以上 → 全体再生成

changed_files = get_pr_changed_files(pr_number)
if len(changed_files) >= REGEN_FILE_THRESHOLD:
    # 変更が大きい → 全体を再生成
    generate_all_docs(repo)
else:
    # 変更が小さい → 関連ドキュメントだけ更新
    affected_docs = identify_affected_docs(changed_files)
    for doc_type in affected_docs:
        regenerate_doc(repo, doc_type, hint_files=changed_files)
```

pushとPR mergeで処理を分けることで、日常的な小さな変更のコストを大幅に抑えられた。

## 4. Webhookの重複対応

GitHubのWebhookは再送されることがある（ネットワークエラー時など）。同じpushイベントが2回来ても、ドキュメント生成が2回走らないようにする必要がある。

処理済みかどうかをDBで管理し、同一の`X-GitHub-Delivery`（イベントごとのユニークID）が来たらスキップする。

```python
delivery_id = request.headers.get("X-GitHub-Delivery")

if is_already_processed(delivery_id):
    return {"status": "skipped", "reason": "duplicate"}

mark_as_processing(delivery_id)
try:
    run_pipeline(payload)
    mark_as_done(delivery_id)
except Exception as e:
    mark_as_failed(delivery_id)
    raise
```

ただし「処理中」のまま失敗したエントリが残ると詰まるので、一定時間後に再処理可能にするタイムアウトも持たせている。

## 5. Lambdaタイムアウトの回避

ドキュメントが10種類あって、それぞれにClaudeを呼ぶと直列処理では数分かかることがある。

Lambdaの同期呼び出しでは限界があるため、Webhookを受け取るLambdaは「受け取って即座に202を返す」だけにして、実際の生成処理は別のLambdaに非同期で投げる構成にした。

```
Webhook Lambda (同期・タイムアウト30秒)
  → SQS にメッセージを投入して即 202 返却

Generator Lambda (非同期・タイムアウト15分)
  → SQS からメッセージを受け取って処理
```

これによりWebhookの処理は「受け取るだけ」になり、生成処理がどれだけ時間がかかってもタイムアウトしない。

## まとめ

| 問題 | 対処 |
|------|------|
| 大きいリポジトリがコンテキスト制限を超える | ファイル数で1パス/2パスを切り替え |
| 毎回全再生成するとAPIコストが高い | PR mergeは差分ファイルだけ更新 |
| Webhookが重複して来る | delivery_idで冪等処理 |
| Lambdaがタイムアウトする | 受信と生成を非同期に分離 |

設計の肝は「大きいリポジトリをいかにAPIコストを抑えて処理するか」と「webhookの冪等性」の2点だった。

これらの設計をベースに、GitHubリポジトリのドキュメントを自動生成・維持するSaaS（[RepoCarta](https://repocarta.jp)）を作っている。まだ小さいサービスだが、自分でも毎日使っている。
