JWTトークン認証の Prosody プラグイン
==================

このプラグインは、[RFC7519] で説明されているJWTトークンに基づいてクライアント接続を検証する Prosody 認証プロバイダーを実装します。
lib-jitsi-meet で外部の認証形式を使用できます。  
ユーザーが認証に成功したら、RFCの説明に従ってJWTトークンを生成し、それをクライアントアプリに渡します。有効なトークンに接続すると、jitsi-meet システムによって認証されたと見なされます。

構成時にクライアントを識別する *アプリケーションID* と、サーバーとJWTトークンジェネレーターの両方で共有される *secret* を提供する必要があります。 RFCの説明にあるとおり、生成されたトークンを認証できるHMACハッシュ値を計算するために *secret* が使用されます。 トークンジェネレーターの実装に使用できる既存のライブラリは多数あります。 詳細はこちら: [http://jwt.io/#libraries-io]

JWTトークン認証は現在、BOSH接続でのみ機能します。

[RFC7519]: https://tools.ietf.org/html/rfc7519
[http://jwt.io/#libraries-io]: http://jwt.io/#libraries-io

### トークンの構造

トークン認証では、次の JWT claims を使用します。

- 'iss' サーバーに接続するクライアントアプリを識別する *アプリケーションID* を指定します。 トークンを生成する前に、サービスプロバイダーに確認する必要があります。
- 'room' トークンを割り当てる部屋の名前を指定します。ここは完全なアドレスではなく、例えば `conference1@muc.server.net` というMUC (Multi-User Chat) があると想定した場合は、`conference1` を指定します。または、「*」を指定して、ドメイン内のすべての部屋にアクセスできます。
- 'exp' RFCで定義されているトークンの有効期限（タイムスタンプ）です。
- 'sub' このトークンで認証するときに使用されるドメインの名前を指定します。例えば `conference1@muc.server.net` というMUC (Multi-User Chat) があると想定した場合は、`server.net` を指定します。
- 'aud' アプリケーションを識別する値です。トークンを使用しているサービスを表します。トークンを生成する前に、サービスプロバイダーに確認する必要があります。

`secret` はHMACハッシュ値を計算し、HS256トークンを検証するために使用されます。  
または、トークンは秘密鍵で署名し、RS256トークンを使用して公開鍵サーバーを介して認証されます。このモードでは、JWTの「kid」ヘッダーを公開鍵の名前に設定する必要があります。バックエンドサーバーは、事前構成された公開鍵サーバーから鍵をフェッチして確認するように構成する必要があります。

### トークン識別子

認証で使用される基本的な claims に加えて、トークン認証ではJWTペイロード内の「コンテキスト」フィールドにユーザー表示情報を提供することもできます。

- 'group' ユーザーが属するグループを指定する文字列です。レポート / 分析での使用を目的としています。
- 'user' 現在のユーザーの表示情報を含むオブジェクトです。
  - 'id' ユーザー識別子の文字列です。レポート / 分析での使用を目的としています。
  - 'name' ユーザーの表示名です。
  - 'email' ユーザーのメールアドレスです。
  - 'avatar' ユーザーのアバターのURLです。
- 'callee' 1対1のビデオ通話を開始するときの表示情報を含むオプションのオブジェクトです。2番目のユーザーが参加する前に、最初のユーザーにオーバーレイを表示するために使用されます。
  - 'id' ユーザー識別子の文字列です。レポート / 分析での使用を目的としています。
  - 'name' 1対1のビデオ通話でのユーザーの表示名です。
  - 'avatar' 1対1のビデオ通話でのアバターのURLです。

#### アクセストークン識別子 / コンテキスト

lib-jitsi-meet のデータにアクセスするには、設定で Prosody モジュール `mod_presence_identity`を有効にする必要があります。

```lua
VirtualHost "jitmeet.example.com"
    modules_enabled = { "presence_identity" }
```

これでトークンのデータが JitsiParticipant クラスのIDとして使用できるようになりました。例えば `USER_JOINED` イベントなどに渡されます。

注: トークンの値は常に有効な値である必要があります。例えばアバターの値を「null」とした場合、エラーがスローされます。

### トークンの例

#### ヘッダー（RS256公開鍵検証を使用）

```
{
  "kid": "jitsi/custom_key_name",
  "typ": "JWT",
  "alg": "RS256"
}
```

#### Payload

```
{
  "context": {
    "user": {
      "avatar": "https:/gravatar.com/avatar/abc123",
      "name": "John Doe",
      "email": "jdoe@example.com",
      "id": "abcd:a1b2c3-d4e5f6-0abc1-23de-abcdef01fedcba"
    },
    "group": "a123-123-456-789"
  },
  "aud": "jitsi",
  "iss": "my_client",
  "sub": "meet.jit.si",
  "room": "*",
  "exp": 1500006923
}
```

### トークンの検証

JWTトークンは次の2つの場所でチェックされています。

- ユーザーが BOSH を介して Prosody に接続したとき。トークン値は「トークン」クエリパラメータまたは BOSH URL として渡されます。ユーザーはXMPP匿名認証方式を使用します。
- ユーザーがルームを作成、または参加するとき。ルームの名前と claims で指定した名前を比較します。これにより、認証されていないユーザーが盗んだトークンを悪用して、システムに新しい会議室を割り当てることが防止できます。管理ユーザーは、Jicofo などで使用される有効なトークンを提供する必要はありません。

### Lib-jitsi-meet オプション

JWT認証が *lib-jitsi-meet* で使用される場合、トークンは *JitsiConference* コンストラクターに渡されます。

```
var token = {アプリケーションにより提供されるトークン}

JitsiMeetJS.init(initOptions).then(function(){
    connection = new JitsiMeetJS.JitsiConnection(APP_ID, token, options);
    ...
    connection.connect();
});

```

### Jitsi-meet オプション

トークンを使用して jitsi-meet 会議を開始するには、トークンをURLパラメータとして指定する必要があります。

```
https://example.com/angrywhalesgrowhigh?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

現在は、会議に参加するすべてのユーザーがトークンを使用する必要があります。
部屋を作成した後、別の匿名ドメインを使用して変更することは可能ですが、それはまだテストされていません。

### トークンプラグインのインストール

トークン認証は、Debianパッケージのインストールを使用して自動的に統合できます。 jitsi-meet をインストールした上で `jitsi-meet-tokens` をインストールします。
これを自動的に構成するには、Prosody 設定ファイルが付属する jitsi-meet のバージョン779以降が必要です。

```
apt-get install jitsi-meet-tokens
```

"Prosody のパッチ" セクションに進み、設定を完了します。

### Prosody のパッチ

JWTトークン認証には、prosody-trunk のバージョンが747以降であることが必要です。

[こちら]から最新の prosody-trunk パッケージがダウンロードできます。ダウンロードしたら次のコマンドでインストールします。

```
sudo dpkg -i prosody-trunk_1nightly747-1~trusty_amd64.deb
```

*/etc/prosody/prosody.cfg.lua* の最後に以下の行が記述されていることを確認します。これにより Jitsi Meet の設定が読み込まれています。Prosody のデフォルト設定が変わる可能性があるため確認が必要です。

```
Include "conf.d/*.cfg.lua"
```

クライアントからサーバーへの暗号化の設定を `false` に変更してください。`true` のままだとトークン認証は機能しません。

```
c2s_require_encryption=false
```

[こちら]: http://packages.prosody.im/debian/pool/main/p/prosody-trunk/

### 手動プラグイン設定

次の3つの手順で Prosody の設定を変更します。

\1. *plugin_paths* に jitsi meet Prosody プラグインのパスを指定します。これは、プラグインが jitsi-meet-token パッケージのインストール時にコピーされる場所です。これは、グローバル設定セクションに含まれている必要があります（ホストの設定ファイルの先頭にある可能性があります）。

```lua
plugin_paths = { "/usr/share/jitsi-meet/prosody-plugins/" }
```

また、オプションでキー認証のグローバル設定を指定します。これらのオプションは両方とも、デフォルトで「*」パラメーターに設定されています。

```lua
asap_accepted_issuers = { "jitsi", "some-other-issuer" }
asap_accepted_audiences = { "jitsi", "some-other-audience" }
```

\2. ドメイン設定の中で、認証方式を "token" に変更し、アプリケーションID、secret、およびオプションでトークンの有効期間を指定します。

```lua
VirtualHost "jitmeet.example.com"
    authentication = "token";
    app_id = "example_app_id";             -- application identifier
    app_secret = "example_app_secret";     -- application secret known only to your token
    									   -- generator and the plugin
    allow_empty_token = false;             -- tokens are verified only if they are supplied by the client
```

共有の secret を使用する代わりに、asap_key_server をベースURLに設定して、JWTトークンヘッダーの「kid」フィールドの公開鍵を見つけることができます。

```lua
VirtualHost "jitmeet.example.com"
    authentication = "token";
    app_id = "example_app_id";                                  -- application identifier
    asap_key_server = "https://keyserver.example.com/asap";     -- URL for public keyserver storing keys by kid
    allow_empty_token = false;                                  -- tokens are verified only if they are supplied
```


\3. MUCコンポーネントの設定セクションでトークン検証プラグインを有効にします。

```lua
Component "conference.jitmeet.example.com" "muc"
    modules_enabled = { "token_verification" }
```
