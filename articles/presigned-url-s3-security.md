---
title: "Presigned URLで画像アップロードを設計したら セキュリティの多層防御が見えてきた"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "S3", "セキュリティ", "バックエンド"]
published: false
publication_name: "digeon"
---

## 0. はじめに

駆け出しエンジニアとしての備忘録としてこの記事を書いています！！未熟な理解のところも多いと思いますのでご容赦願います。
今回は学習として、画像アップロード処理をどのように詳細に設計していくかを考えていきました。その際に、新しい概念に次々と出会ったのでこの記事ではPresigned URLによるセキュリティについて記事に取り上げようと思います。

## 1. なぜPresigned URLを選んだのか（設計判断）

もともと画像アップロード処理について考えると、ブラウザ→サーバー→S3というようにデータが行き来しているのではないか、と思っていました。

これは典型的なプロキシ型の処理です。

:::message
**プロキシ型とは**
ブラウザが送った画像バイナリをサーバーが一度受け取り、そのままS3へ転送する方式です。サーバーが「中継役（プロキシ）」となるため、同じバイナリがブラウザ→サーバー→S3と2回ネットワークを流れます。
:::

しかしこの方式には構造的な無駄があります。バックエンドが画像のバイナリを中継するため、同じデータがブラウザ→バックエンド→S3と2回転送されます。同時アップロードが増えるほどバックエンドのCPUと帯域を消費し、バックエンド自体が処理のボトルネックになってしまいます。（CPU＝料金所での審査処理、帯域＝道幅の広さ、で理解してます）

そこで今回学習のために取り入れたのが **Presigned URL方式** です。バックエンドがS3への署名付きURLを発行し、ブラウザはそのURLを使ってS3に直接アップロードします。バックエンドが扱うのはメタデータ・署名付きURL・S3キーのみで、画像のバイナリ本体には一切触れません。帯域もCPUも節約でき、同時アップロードが増えてもバックエンドの負荷はほぼ変わらないのが利点として挙げられます。

また、Presigned URL方式にはSSRF対策上の利点もあります。

:::message
**SSRFとは**
悪意あるユーザーが指定したURLをサーバーが内部でfetchする設計があると、そのサーバーを踏み台にして本来閲覧できないIAM認証情報などを盗みとられる攻撃です。ユーザー指定URLをサーバーがfetchする設計では①URL検証②ネットワーク隔離③IMDSv2強制の三層で防ぐ必要がありますが、Presigned URL方式はそもそもユーザー指定のURLをfetchする仕組みを持たないため、引き金となる機能が存在せずシンプルに防ぐことができます。

最初SSRFと聞いたとき何を言っているのかわかりませんでした。セキュリティの攻撃名称ってむずかしいアルファベットが多くて困ります。（XSSとか...）
:::

:::message
**IMDSv2とは**
EC2インスタンスが自身のメタデータ（IAMロールの一時認証情報など）を取得するためのAWS内部APIです。旧バージョン（IMDSv1）は `http://169.254.169.254/` へのシンプルなGETリクエストだけでアクセスできたため、SSRF攻撃でサーバーにそのURLをfetchさせるだけでIAM認証情報を盗めてしまいました。

IMDSv2（v2）ではセッショントークン方式に変わり、メタデータ取得の前にPUTリクエストでトークンを先に発行する必要があります。SSRFはGETリクエストを誘発する攻撃であるためPUTが発行できず、IMDSv2を強制することでメタデータへの到達を防ぐことができます。
:::


なお、プロキシ型が常に悪いわけではありません。サーバーを経由することでアップロード前にウイルススキャンや画像リサイズといったサーバーサイド処理を挟めること、監査ログをサーバー側で確実に記録できることが利点として挙げられます。要件によっては意図してこの方式を選ぶケースもあるらしいです。


参考: [Amazon S3 署名付きURLについて](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/using-presigned-url.html) / [OWASP: SSRF](https://owasp.org/Top10/2021/ja/A10_2021-Server-Side_Request_Forgery_%28SSRF%29/)

## 2. 三者の役割分担

ブラウザ・App Server・S3の三者が登場する中で、それぞれが何の役割を担っているのだろうと思いました。ブラウザはHTTPリクエストとアップロード処理のロジックを担っているのはなんとなくわかるのですが、この際App Serverは何しているんだろう？
そもそもS3って何なのか、ただのデータの倉庫なのかという部分から気になることが止まりませんでした。
調べてみると、三者の役割分担は以下のようになっているようです。

| 役割 | 主な責務 |
| --- | --- |
| **ブラウザ** | ファイル選択・S3への直接PUT・進捗表示 |
| **App Server** | JWT認証/認可・Presigned URL発行・メタデータのDB永続化 |
| **S3** | バイナリ保管・署名の再計算と照合によるリクエスト検証 |

**ブラウザ** はユーザー操作の窓口です。ファイル選択やドラッグ&ドロップを受け付け、App ServerへのHTTPリクエスト（POST・PATCH・GET・DELETE）と、S3へのバイナリの直接送信を担います。

**App Server** はコントロールタワーの役割を担います。JWTによる認証・認可、Presigned URLの発行、DBへのメタデータの永続化を行います。画像のバイナリ本体には一切触れず、「誰が・何を・どこに置くか」の管理だけを担当します。

**S3** はバイナリデータの保管庫です。ブラウザから直接PUTされたファイルを受け取り、URLに埋め込まれた署名をAWS認証情報（secret access keyまたは一時認証情報）でSigV4再計算・照合することで、App Serverが正規に発行したリクエストかどうかを検証します。

![三者の役割分担](/images/upload_swimlane_plain.png)

参考: [Securing Amazon S3 presigned URLs for serverless applications](https://aws.amazon.com/blogs/compute/securing-amazon-s3-presigned-urls-for-serverless-applications)

## 3. Secret access keyによる署名の仕組み

調べていると、secret access keyというワードが目につきました。どうやら認証・認可のためのブラウザ⇔App Server間のJWT認証と同じように、ブラウザ⇔S3間でもsecret access keyによる署名によって特定のS3操作を期限付きで認可していることがわかります。
以下、調べた内容となっています。

Presigned URLの中核にあるのは **HMAC-SHA256** という署名アルゴリズムです。App ServerはPresigned URLを生成するとき、AWS認証情報（secret access keyまたは一時認証情報）を使って「HTTPメソッド・対象のS3キー・有効期限・発行日時」を材料にHMAC-SHA256でハッシュ値を計算します。この値が署名本体であり、URLのクエリパラメータとして材料とともに埋め込まれます。

```
https://s3.amazonaws.com/app-images-prod/uploads/user-1/abc.jpg
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Expires=300
  &X-Amz-Signature=a3f9b2c1...  ← 署名本体
```

ブラウザはこのURLをそのまま使ってS3へのPUTリクエストを送ります。S3はリクエストを受け取ると、URLから材料を取り出し、同じAWS認証情報でSigV4署名を再計算して、値が一致すれば「App Serverが正規に発行したリクエスト」としてアクセスを許可します。

この仕組みの重要な点は、secret access key（またはIAMロールが発行した一時認証情報）がApp ServerとAWSの間だけに存在することです。ブラウザはその認証情報を持たないためURLを改ざんしても正しい署名を作れず、対象キーを一文字でも書き換えれば再計算した署名が食い違い、S3に即座に拒否されます。

（署名については高校の情報の授業で触れていたので、アハ体験でした...）

参考: [署名バージョン4 - AWS実装について](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_sigv.html)

## 4. セキュリティの多層防御

セキュリティは自分はもともと一つや二つくらいで集約して実行されているものだと思っていました。そもそもセキュリティを脅かす問題がいくつあるか、どのような種類があるかの理解も限られてたものだったのでこの多層防御という概念はとても新鮮でした。各セキュリティごとに守る対象や意図が異なっていて大きな学びです。

ファイルアップロードのセキュリティは一か所で担保できるものではなく、今回の設計では5層の防御を重ねています。

### 第1層：フロントエンドのバリデーション

Content-Typeとファイルサイズをブラウザ上でチェックし、明らかにおかしいリクエストをUIの段階で弾きます。

ただし、ブラウザの検証は開発者ツールで簡単に改ざんできるため、これだけでは不十分です。フロントエンドのバリデーションは「容量が大きすぎるファイルをサーバーに到達する前にエラーにする」というUX向上が主目的であり、セキュリティとしてはほとんど期待できません。
（バリデーションの意図にも違いがあるんだ）

### 第2層：App Serverでのバリデーション

JWTのuser_idとDBのuser_idを照合して「誰がどのリソースを操作できるか」をビジネスロジックで判定します。同時にContent-Type・拡張子・サイズをサーバー側で再検証します。フロントエンドを迂回した直接リクエストもここで弾くことができます。

### 第3層：バケットポリシーでのバリデーション

Presigned URL生成時にContent-Typeを署名対象に含めることで、指定外のContent-TypeでのリクエストをS3の署名検証段階で拒否します。Presigned POSTを使う場合はポリシー条件にContent-Typeを明示的に列挙できます。

:::message
**Content-Typeホワイトリストの実装について**
S3バケットポリシーの条件キー（`s3:x-amz-storage-class` など）にはContent-Typeを直接フィルタするキーが存在しません。今回の要件におけるPresigned PUT URL方式でのContent-Type制御は、App ServerがURL生成時に `ContentType` パラメータを指定することで実現させようとしています。S3はアップロード時に署名と一致するContent-Typeヘッダーを要求し、不一致のリクエストを署名検証の段階で拒否します。

なお、Presigned POSTを使う方式では、S3 POST Policyの条件として `["eq", "$Content-Type", "image/jpeg"]` のようにContent-Typeを明示的に列挙することが公式に定義されています。

参考: [バケットポリシーの条件キー例](https://docs.aws.amazon.com/AmazonS3/latest/userguide/amazon-s3-policy-keys.html) / [Amazon S3 POST Policy](https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-HTTPPOSTConstructPolicy.html)
:::

:::message alert
**第1〜3層の共通の限界**
第1層から第3層はいずれも「ブラウザが申告したContent-Typeの文字列」を検証しています。悪意あるユーザーがContent-Typeを `image/png` と偽装しながら実行ファイルを送った場合、この3層はまとめてすり抜けられてしまいます。
:::

### 第4層：IAMロールによるAWSインフラレベルの認可

「このAWSサービスはS3を操作してよいか」をAWSが判定します。App Serverを経由しない不正なアクセスをインフラの層で遮断します。

### 第5層：Lambdaによるマジックバイト検証

S3に保存されたファイルのバイナリ本体を直接読み、先頭数バイトの **マジックバイト** でファイルの実形式を確かめます。

- JPEGなら必ず `FF D8 FF` で始まる
- PNGなら必ず `89 50 4E 47` で始まる

Content-Typeを偽装したファイルや、ポリグロット形式（複数のフォーマットとして解釈できるファイル）を検出するのがこの層の役割としています。

ちなみに、以下Claudeによる補足となっています。

> *Claude による補足*
> 申告された文字列を一切信用せずファイルの実態で判定する点で前の層より強力ですが、マジックバイト検証だけで完全な防御を保証できるわけではありません。デコード・再エンコードによる検証や、検証が完了するまで配信しない設計と組み合わせることで、セキュリティ性に優れた設計を実現する...
セキュリティって奥が深いですね。

---

各層は独立して動作し、前の層をすり抜けたものを次の層が捕まえる構造になっています。どれか一層が突破されてもほかの層が機能するのが多層防御の本質だと感じました。

参考: [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)

## 5. 設計中に気づいた見落とし

設計を進める中で、重要な見落としに気づきました。アップロード（PUT）についてはPresigned URL方式を採用して設計していましたが、**ダウンロード（GET）の公開範囲** について明確に定義していなかったのです。

「画像は誰でも見られるのか、本人だけなのか」という前提が要件に明記されていなければ、アーキテクチャの選択自体が変わります。

| 公開範囲 | 推奨アーキテクチャ |
| --- | --- |
| 誰でも閲覧可 | CloudFront経由で配信 |
| 本人のみ | GETのたびにApp ServerでJWT検証 → GET用Presigned URLを発行 |

画像アップロードという一見シンプルな機能でも「誰が・いつ・どのリソースを・どのように扱えるか」を要件の段階で明確にしなければ、設計の手戻りが発生します。実際に今回手戻りが発生し、一機能だけでもその大変さを感じました。

:::message
この教訓は医療・金融といったセキュリティ要件の厳しいサービスでは特に重要です。機密性の高いデータを扱う場合は、VPC内のプライベートネットワーク・Presigned URLによるアクセス制御・Lambdaでの実体検証を組み合わせた設計が求められるため、「どのレベルのセキュリティが必要か」は実装を始める前に確定させるべき最重要事項の一つだと実感しました。
:::

## 6. さいごに

今回、最も理解に苦しんだのは **IAMとHMAC（署名）の違い** です。

- **IAMロール** ：バックエンドがS3にPUTする権限を持つ根拠。AWSはそのロールに基づいて一時的な認証情報を自動で発行する。効くのは「AWSを操作する主体（バックエンド）」に対してであって、アプリのエンドユーザーには効かない。
- **署名（HMAC）** ：IAMロールの権限を材料にして作成される。バックエンドはPUT・キー・期限・サイズ条件をもとにHMAC計算を実行し、その結果をURLに埋め込む。URLを持つ人は期限内に使えるが、許可される操作の範囲は発行元のIAM権限に制限される。

整理するとこうなります。

> **IAMロール** は「URLを発行するバックエンドの権限の源」
> **署名** は「その権限から切り出した、特定操作だけを許す使い捨ての許可証」
> 作るのはバックエンド、検証するのはS3

セキュリティやエラーハンドリングは様々なケースが想定され、詳細設計で必要十分を追い求めることの難しさを感じました。特に状態遷移についてはこの場合まだまだ考慮の余地があり、安全性を優先するか速度を優先するかで遷移する状態が変わってきます。また、エラーハンドリングについてはここに取り上げていない内容で定義しないといけないこと、判断しないといけないことが山のようにあります。圧倒されそうです...
AIはそれらしい設計をたくさん出してくれますが、その妥当性を自身で判断できるようになれるよう、これからも積極的に学んでいこうと思います。

## 参考資料（まとめたもの）

- [Amazon S3 ドキュメント](https://docs.aws.amazon.com/ja_jp/s3/?icmpid=docs_homepage_featuredsvcs)
- [Amazon S3 署名付きURL](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/using-presigned-url.html)
- [Securing Amazon S3 presigned URLs for serverless applications](https://aws.amazon.com/blogs/compute/securing-amazon-s3-presigned-urls-for-serverless-applications)
- [署名バージョン4 - AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_sigv.html)
- [Amazon S3 POST Policy（Content-Type条件キーの公式定義）](https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-HTTPPOSTConstructPolicy.html)
- [OWASP: Server-Side Request Forgery (SSRF)](https://owasp.org/Top10/2021/ja/A10_2021-Server-Side_Request_Forgery_%28SSRF%29/)
- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
