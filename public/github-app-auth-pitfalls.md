---
title: GitHub Appの認証、思ったより罠が多かった
tags:
  - Python
  - GitHub
  - 認証
  - 個人開発
  - GithubApp
private: false
updated_at: '2026-06-04T00:06:17+09:00'
id: c2323f3a3175f59dffd3
organization_url_name: null
slide: false
ignorePublish: false
---

GitHub Appを使ってリポジトリにpushする機能を実装した。OAuthと似たようなものだろうと思っていたが、全然違った。ドキュメントが分散していて全体像を掴むまでに時間がかかったので、詰まったところを残しておく。

## 認証が2段階ある

まずここで混乱した。

GitHub Appの認証は「App自体の認証」と「リポジトリへのアクセス」が別になっている。

```
RSA秘密鍵で署名
  ↓
App JWT（有効10分）
  ↓ POST /app/installations/{id}/access_tokens
Installation Access Token（有効1時間）
  ↓
GitHub API
```

OAuth Appならアクセストークン1本でGitHub APIを叩けるが、GitHub Appはまず「俺がそのAppだ」というJWTを作り、それを使ってインストール先のトークンを取得するという2段階が必要になる。公式の説明を読んでいると「App JWT」と「Installation Access Token」が別のページに書いてあって、最初はこの2つが同じものだと思い込んでいた。

## JWTのiatを現在時刻にすると弾かれる

App JWTを生成するとき、`iat`を`int(time.time())`にしていたら認証エラーになった。

```python
# これで弾かれた
jwt_payload = {
    "iat": int(time.time()),
    "exp": int(time.time()) + 540,
    "iss": app_id,
}
```

GitHubのサーバーとの間にわずかな時刻のズレがあると、`iat`が「未来の時刻」として扱われて弾かれる。対策は60秒前にずらすことで、公式ドキュメントにも書いてあるが見落としていた。

```python
now = int(time.time())
jwt_payload = {
    "iat": now - 60,   # クロックスキュー対策
    "exp": now + 540,  # 上限が10分なので余裕を持って9分
    "iss": app_id,
}
```

## Installation Access Tokenを毎回取得するとレート制限に当たる

Installation Access Tokenは1時間で失効する。毎回APIを叩いて取得していたら、しばらくしてレート制限のエラーが出てきた。当然キャッシュが必要で、ただし「失効ギリギリまで使う」のも危ない。取得してすぐ失効するケースがあるので、5分前に再取得するようにした。

```python
_token_cache: dict[int, tuple[str, float]] = {}

def _get_installation_token(installation_id: int) -> str:
    cached = _token_cache.get(installation_id)
    if cached and time.monotonic() < cached[1] - 300:  # 5分前まで使う
        return cached[0]

    app_jwt = _generate_app_jwt()
    resp = requests.post(
        f"https://api.github.com/app/installations/{installation_id}/access_tokens",
        headers={
            "Authorization": f"Bearer {app_jwt}",
            "Accept": "application/vnd.github+json",
            "X-GitHub-Api-Version": "2022-11-28",
        },
        timeout=10,
    )
    resp.raise_for_status()

    data = resp.json()
    token = data["token"]

    # expires_atがレスポンスに含まれているのでそれを使う
    # 形式は "2024-01-01T00:00:00Z"
    expires_at_str = data.get("expires_at", "")
    try:
        dt = datetime.fromisoformat(expires_at_str.replace("Z", "+00:00"))
        ttl = (dt - datetime.now(timezone.utc)).total_seconds()
        expires_monotonic = time.monotonic() + ttl
    except Exception:
        expires_monotonic = time.monotonic() + 3600

    _token_cache[installation_id] = (token, expires_monotonic)
    return token
```

`expires_at`はレスポンスに入っているので、自前で計算するよりズレない。

## Webhookは全イベントが1つのエンドポイントに届く

pushだけ受け取るエンドポイントを作れると思っていたが、そうではなかった。インストール・push・PR・Marketplaceの購入など、App宛ての全イベントが同じURLに届く。`X-GitHub-Event`ヘッダーで種別を判定して自前でルーティングする必要がある。

```python
@app.post("/webhooks/github")
async def github_webhook(
    request: Request,
    x_github_event: str = Header(None),
    x_hub_signature_256: str = Header(None),
):
    payload = await request.body()

    # 署名検証を先にやる
    secret = get_secret("github_app_webhook_secret")
    if not verify_signature(payload, x_hub_signature_256, secret):
        raise HTTPException(status_code=401)

    body = json.loads(payload)

    if x_github_event == "ping":
        return {"status": "ok"}        # App登録直後の疎通確認、即返す

    if x_github_event == "installation":
        handle_installation(body)

    elif x_github_event == "push":
        handle_push(body)

    elif x_github_event == "pull_request":
        handle_pull_request(body)

    return {"message": "ok"}
```

`ping`イベントはApp設定を保存したときにGitHubから飛んでくる疎通確認で、これを即座に200で返さないとApp登録が完了しない。署名検証をしてから返すようにしていたら、秘密鍵の設定ミスで署名検証が失敗してApp登録できないという状況にハマった。`ping`だけは先に返す。

## Webhook署名検証は`hmac.compare_digest`を使う

署名の検証を`==`でやっていたが、タイミング攻撃の余地があるので`hmac.compare_digest`を使う。

```python
def verify_signature(payload: bytes, signature_header: str, secret: str) -> bool:
    if not signature_header or not signature_header.startswith("sha256="):
        return False

    expected = hmac.new(
        secret.encode("utf-8"),
        payload,
        hashlib.sha256,
    ).hexdigest()

    actual = signature_header[len("sha256="):]
    return hmac.compare_digest(expected, actual)
```

`==`は文字列の一致箇所が増えるほど処理時間が長くなり、タイミングで情報が漏れる。`compare_digest`は常に一定時間で比較する。

## ボット自身のpushでWebhookが無限ループする

GitHub Appがドキュメントを更新してpushすると、そのpushに対してWebhookが届く。そのままドキュメント生成を走らせると永遠にループする。

```python
if x_github_event == "push":
    pusher_name = body.get("pusher", {}).get("name", "")
    if pusher_name.endswith("[bot]"):
        return {"message": "ignored"}

    # ここからドキュメント生成処理
```

GitHub ActionsのコミットはpusherがL`github-actions[bot]`になる。自前のbotアカウントでpushするなら、そのアカウント名で判定する。

---

全体的に「OAuthと同じような感覚でやると詰まる」という印象だった。公式ドキュメントは[Authenticating as a GitHub App installation](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation)と[Receiving webhooks with a GitHub App](https://docs.github.com/en/apps/creating-github-apps/writing-code-for-a-github-app/building-a-github-app-that-responds-to-webhook-events)を最初に読んでおくと全体像が掴めてよかった。
