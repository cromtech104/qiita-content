---
title: FastAPI(Mangum)をAWS Lambdaで本番運用して踏んだ落とし穴
tags:
  - Python
  - AWS
  - lambda
  - 個人開発
  - FastAPI
private: false
updated_at: '2026-06-28T14:28:41+09:00'
id: 96864b934276c08d0421
organization_url_name: null
slide: false
ignorePublish: false
---

FastAPIをMangumでラップしてLambdaに乗せる構成は、ローカルと同じコードがそのまま動くので最初は快適だった。ただ、ローカルやステージングでは出ず、本番で間欠的にしか再現しない不具合をいくつか踏んだ。原因が分かれば単純なものばかりだが、特定までに時間を使ったので残しておく。

## ウォームコンテナで「event loopが無い」と言われて500になる

一番ハマったのがこれ。HTTPのエンドポイントが、あるときから間欠的に500を返すようになった。CloudWatchを見ると、

```
RuntimeError: There is no current event loop in thread 'MainThread'
```

が出ている。再現条件がしばらく分からなかったが、「重い処理を同じLambdaで非同期に走らせた直後のリクエスト」だけで起きると分かった。

うちの構成では、時間のかかる処理（外部API呼び出しが積み重なる）をHTTPレスポンスのタイムアウトから切り離すため、同じLambda関数を`InvocationType='Event'`で自己invokeしている。その非同期側のエントリで`asyncio.run(...)`を使っていた。

```python
def handler(event, context):
    if is_async_job(event):
        return asyncio.run(run_job(event))  # ← これが問題の起点
    return mangum_handler(event, context)
```

`asyncio.run()`は終了時に内部で`asyncio.set_event_loop(None)`を呼ぶ。Lambdaのコンテナは使い回されるので、同じコンテナに次のHTTPリクエストが来ると、Mangum内部の`asyncio.get_event_loop()`が「現在のループが無い」状態に当たる。Python 3.10以降、ループが無いときの`get_event_loop()`はループを自動生成せず`RuntimeError`を投げるようになったため、これが500として表面化する。ローカルでは「非同期ジョブ→直後に同じプロセスでHTTP」という並びが起きにくいので、まず再現しない。

対策は、HTTPに委譲する前に「使えるループが必ず存在する」ことを保証すること。

```python
def handler(event, context):
    if is_async_job(event):
        return asyncio.run(run_job(event))

    # HTTPに渡す前に current event loop を保証する
    try:
        loop = asyncio.get_event_loop()
        if loop.is_closed():
            asyncio.set_event_loop(asyncio.new_event_loop())
    except RuntimeError:
        asyncio.set_event_loop(asyncio.new_event_loop())

    return mangum_handler(event, context)
```

ポイントは2つある。`get_event_loop()`が`RuntimeError`を投げる「ループが無い」ケースだけでなく、「ループは残っているがすでにcloseされている」ケースもあること。後者では`get_event_loop()`はclosedなループをそのまま返してしまい、それをMangumに渡すと今度は`RuntimeError: Event loop is closed`で500になる。なので`None`（RuntimeError）と`is_closed()`の両方を救って、どちらでも新しいループに差し替える。

そもそも非同期ジョブ側で`asyncio.run()`を使わず手動で作ったループを使い回す手もあるが、ジョブ実行を素直に書きたかったので、HTTP側で必ずループを用意する形にした。

## module-globalで持った非同期クライアントが「Event loop is closed」になる

上と根は同じで、こちらは外部APIクライアント側で起きた。httpxベースの非同期クライアント（自分の場合はLLMのSDK）を、モジュール読み込み時に1個だけ作って使い回していた。

```python
# モジュールトップレベル。importのときに1回だけ生成される
client = AsyncAnthropic(api_key=...)

class Agent:
    async def run(self, ...):
        return await client.messages.create(...)
```

DRYのつもりだったが、非同期ジョブを`asyncio.run()`で実行する構成だと壊れる。`AsyncAnthropic`の内部httpxは、最初に使われたときのイベントループに紐づく。`asyncio.run()`は実行ごとに新しいループを作って終了時に閉じるので、2回目の`asyncio.run()`で同じクライアントを使うと、すでに閉じたループに対して通信しようとして`Event loop is closed`になる。

直し方は、クライアントをグローバルで持たず呼び出しごとに生成・closeすること。

```python
class Agent:
    async def run(self, ...):
        async with AsyncAnthropic(api_key=..., timeout=...) as client:
            return await client.messages.create(...)
```

接続の張り直しコストは気になるが、Lambdaでは1呼び出しの寿命が短く、コネクションを跨いで使い回す前提自体が崩れている。「グローバルに置いて使い回す」最適化は同期クライアントには効くが、イベントループに束縛される非同期クライアントだと逆に壊れる、というのが教訓だった。`timeout`を明示しておくのも忘れがちで、未指定だとLambdaのタイムアウト手前まで掴んだまま戻ってこないことがある。

## VPC内のLambdaからRDSに繋ぐと、再利用した接続が死んでいる

LambdaをVPCに入れてRDSに繋ぐ構成にしたところ、`OperationalError`（接続が切れている系）が間欠的に出るようになった。負荷とは関係なく、「しばらくアクセスが無かった後の最初のリクエスト」で出やすい。

原因は、アイドル状態のTCP接続がNATやRDS側で黙って切られていること。SQLAlchemyのコネクションプールはその死んだ接続をそのまま再利用しようとして失敗する。Lambdaのコンテナが生き残っている間、プールも生き残るので、コンテナの寿命とDB接続の寿命がずれる。

`pool_pre_ping`で再利用前に軽い疎通確認をするのが基本。

```python
engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True,
    pool_recycle=300,  # 秒。RDSのアイドル切断より十分短く取る
)
```

`pool_pre_ping`だけでも大半は防げるが、ping自体がちょうど境界で失敗する余地が残る。なので、一定時間経った接続をプール側から能動的に捨てて張り直す`pool_recycle`も併用している。値はRDS側のアイドルタイムアウトより十分短くしておく。

そもそもLambda + RDSはコネクション管理が相性悪いので、規模が大きくなるならRDS ProxyやData APIを検討する話になる。個人開発の規模では`pre_ping` + `recycle`で十分実用になった。

## Mangumは非HTTPイベントを渡されると例外を投げる

前述の自己invokeでハマったのがこれ。Mangumは「API Gateway / Function URLのHTTPイベント」専用で、それ以外の形のイベント（自分が`InvocationType='Event'`で投げた独自イベントや、SQS・EventBridgeのイベント）を渡すと、イベント形式を推定できずに`RuntimeError`を投げる。

なので、1つのhandlerでHTTPと非同期ジョブの両方を捌くなら、Mangumに渡る前に非HTTPイベントを確実に横取りする必要がある。`requestContext`の有無で判定する例をよく見るが、Function URLとAPI Gatewayでイベント形が違うので、自分のジョブには独自のトップレベルキーを付けて、それを最優先で見るようにした。

```python
def handler(event, context):
    # 自分のジョブは独自キーで判定（イベント形式に依存しない）
    if isinstance(event, dict) and event.get("_my_job"):
        return asyncio.run(run_job(event))
    # それ以外はHTTPとしてMangumへ
    return mangum_handler(event, context)
```

「自分が投げたものを自分で見分ける」キーを決めておくと、将来イベントソースが増えても誤判定しない。

## logger.info()がCloudWatchに出ない

最後は小さいが地味に時間を溶かした話。`logging.getLogger(__name__).info(...)`で書いたログがCloudWatchに出てこなかった。

LambdaのPythonランタイムは、起動時にrootロガーへ専用ハンドラを付けてくれる。ただしrootのログレベルは`WARNING`のまま。アプリ側で`basicConfig`や`setLevel`を呼んでいないと、`INFO`ログはrootの閾値で捨てられて出てこない。`print()`は出るのに`logger.info()`は出ない、という分かりにくい状態になる。

```python
import logging, os

level = os.getenv("LOG_LEVEL", "INFO").upper()
# force=True でLambdaが既に付けたハンドラにも適用する
logging.basicConfig(level=level, force=True)
logging.getLogger().setLevel(level)
# 冗長なboto系は絞る
logging.getLogger("botocore").setLevel(logging.WARNING)
logging.getLogger("boto3").setLevel(logging.WARNING)
```

`force=True`がないと、ランタイムが先にハンドラを付けている分`basicConfig`が何もせず終わるので、ここは必要。

---

振り返ると、event loop関連の2つは「ローカルで再現しない・本番で間欠的に出る」点が共通していて、原因は全部「Lambdaのコンテナ（とプロセス状態）が使い回される」ことに行き着く。リクエストごとにまっさらなプロセスが立ち上がる前提でコードを書いていると足をすくわれる。ウォームコンテナにグローバル状態を残す書き方をしている箇所を疑うと、だいたい当たりだった。

自分はBacklogのチケットからGitHubのDraft PRを自動生成する [keros](https://keros.repocarta.jp) というサービスをこの構成（Lambda + FastAPI + Mangum）で動かしていて、ここに挙げた落とし穴は実際にそこで踏んだもの。同じ構成で運用している人の役に立てば。
