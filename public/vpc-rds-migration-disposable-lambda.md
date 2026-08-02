---
title: VPCの中にしかないRDSにマイグレーションを流すのに、使い捨てのLambdaを立てた
tags:
  - AWS
  - RDS
  - lambda
  - PostgreSQL
  - 個人開発
private: false
updated_at: '2026-08-02T17:58:57+09:00'
id: 0e930c0e7e3b43dd6622
organization_url_name: null
slide: false
ignorePublish: false
---

本番のRDSをプライベートサブネットに置いてて、VPCの外からは繋がらないようにしてる。セキュリティ的にはそれでいいんだけど、いざスキーマのマイグレーション（`ALTER TABLE` とか）を流そうとすると、手元の `psql` からRDSに届かなくて詰まる。

よくある逃げ道は、踏み台のEC2を1台立ててSSHで入るとか、RDSを一時的にパブリックに開けるとか。でも踏み台は消し忘れるし常時課金だし、RDSを公開するのは論外。VPNをこのためだけに用意するのも重い。

で、結局「同じVPCの中に使い捨てのLambdaを立てて、そこからマイグレーションSQLを流して、終わったら消す」に落ち着いた。踏み台みたいに残らないし、RDSは閉じたまま。

## やってること

流れはこう。

1. マイグレーションを流すコード（SQLを `psycopg` で実行するだけのhandler）をzipにする
2. VPCに繋がる一時的なIAMロールを作る
3. RDSと同じサブネット／SGでLambdaを作る
4. 1回invokeして結果を見る
5. Lambdaとロールを消す

### IAMロール

VPCにアタッチするLambdaには、ENIを作る権限（`AWSLambdaVPCAccessExecutionRole`）が要る。これが無いとVPCに繋げなくて、関数がずっとPendingのままになる。あとマイグレーション用の管理者DBパスワードをSecrets Managerから読む権限も足す。

```bash
aws iam create-role --role-name migrate-temp \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy --role-name migrate-temp \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
# + Secrets Manager の GetSecretValue を inline policy で付ける
```

### LambdaをVPCの中に立てる

RDSに届くサブネットとセキュリティグループを指定する。ここを間違えるとタイムアウトする。RDS側のSGが、Lambdaに付けたSGからの5432を許可してるかも確認しておく。

```bash
aws lambda create-function --function-name migrate-temp \
  --runtime python3.12 --handler migrate.handler \
  --role arn:aws:iam::123456789012:role/migrate-temp \
  --zip-file fileb://migrate.zip \
  --timeout 120 \
  --vpc-config SubnetIds=subnet-aaa,subnet-bbb,SecurityGroupIds=sg-xxx
```

### 流して、消す

```bash
aws lambda invoke --function-name migrate-temp \
  --payload '{"migrations":["2026-07-01_add_column"]}' out.json
cat out.json   # 適用結果を確認

aws lambda delete-function --function-name migrate-temp
# ロールは inline policy を先に消してから
aws iam delete-role --role-name migrate-temp
```

## ハマったところ

- **DDL権限**：アプリが普段使ってるDBユーザーだと `ALTER TABLE` できないことが多い。マイグレーション用にマスターのパスワードを別のシークレットに置いて、それで繋いだ。アプリの実行ユーザーにDDL権限を持たせないほうが安全だし。
- **VPCアタッチで待たされる**：`AWSLambdaVPCAccessExecutionRole` を付け忘れるとENIが作れず、関数がずっとPendingになる。最初これで悩んだ。
- **共有RDSの取り違え**：1つのRDSに複数プロダクトを同居させてると、`search_path` を対象スキーマに固定しないと隣に流しかねない。使い捨てとはいえ、流す先は絞る。
- **冪等に書く**：`IF NOT EXISTS` / `DROP ... IF EXISTS` で書いておくと、途中で失敗して再invokeしても事故らない。

踏み台を常設するほどでもない、たまのマイグレーションにはこれが軽い。関数もロールも作って消すだけなので、後に何も残らないのがいい。

---

ちなみにこれ、Backlogのチケットから GitHub の Draft PR を作る Keros というのを個人で運用しててやってる話です。RDSはVPC内・アプリはLambdaなので、スキーマ変更のたびにこの使い捨てLambdaを回してます。
https://keros.repocarta.jp/example
