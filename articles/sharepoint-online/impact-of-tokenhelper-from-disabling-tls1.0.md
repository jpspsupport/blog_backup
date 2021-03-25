---
title: TLS 1.0 や 1.1 廃止による TokenHelper でアプリ専用トークン取得動作への影響について
date: 2021-03-25
tags:
  - SharePoint Online
---

Microsoft 365 においては、メッセージ センターや各公開情報に記載の通り、TLS 1.0 と 1.1 を廃止し、新しい暗号化技術を基準とすることで、お客様データのセキュリティ保護を向上する動きを拡大しております。
全世界のお客様を想定すると一部に限定されることを想定してはおりますが、特に旧バージョンのオペレーティング システムや .NET Framework (4.5 以下) 基盤をご利用の場合は、ご自身のシステムが影響を受けないか今一度ご確認ください。

一般的なプロトコルとして TLS ではトランスポート層において、クライアントとサーバーで対応バージョン等の情報を交換し、ハンドシェイクを実施します。この通信が確立できた場合はアプリケーション層の通信を開始する流れです。
TLS 1.0 や 1.1 が廃止が適応されると、クライアントから TLS 1.0 を要求した際に、サーバーがこれを拒否し、トランスポート層で通信が遮断されます。

このような遮断を受けた時、.NET アプリケーション側で確認する典型的なエラーは以下となります。

### 典型的な例外の例

`System.Net.WebException: The underlying connection was closed: An unexpected error occurred on a send. --> System.IO.IOException: Unable to read data from the transport connection: An existing connection was forcibly closed by the remote host. --> System.Net.Sockets.SocketException: An existing connection was forcibly closed by the remote host.`

HttpWebRequest.GetResponse メソッドや、Client Side Object Model (CSOM) などでは、上記のようなエラーで、通信が遮断される動作が想定されます。
なお、TokenHelper などでアプリ専用トークンをご利用の場合は、少し異なったエラーが発生します。

### TokenHelper アプリ専用トークン取得時の例

`Microsoft.IdentityModel.SecurityTokenService.RequestFailedException: Token request failed. ---> System.Net.WebException: The remote server returned an error: (401) Unauthorized.at System.Net.HttpWebRequest.GetResponse() at Microsoft.IdentityModel.S2S.Protocols.OAuth2.OAuth2WebRequest.GetResponse() at Microsoft.IdentityModel.S2S.Protocols.OAuth2.OAuth2S2SClient.Issue(String securityTokenServiceUrl OAuth2AccessTokenRequest oauth2Request) --- End of inner exception stack trace --- at Microsoft.IdentityModel.S2S.Protocols.OAuth2.OAuth2S2SClient.Issue(String securityTokenServiceUrl OAuth2AccessTokenRequest oauth2Request) at SampleNameSpace.TokenHelper.GetAppOnlyAccessToken(String targetPrincipalName String targetHost String targetRealm)`

上記エラーが発生したとしても、対処方法としては TLS 1.2 を有効化する対応を実施ください。

### 対処策
公開情報ご確認の上、上記のようなエラーが発生した場合は TLS 1.0 および 1.1 無効化による影響を疑い、対処を実施ください。

タイトル : TLS 1.2 を有効にする方法
アドレス : https://docs.microsoft.com/ja-jp/mem/configmgr/core/plan-design/security/enable-tls-1-2

なお、Azure App Service 上のアプリで .NET Framework 4.5 以下をご使用の場合は、対象のフレームワークバージョンを 4.6 以上に上げることが推奨です。
それが難しい場合は、以下のコードを一行アプリケーションの初期化段階に加えることをお勧めします。

```csharp
ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;
```

### 詳細説明

TokenHelper では、Realm を取得し、ClientID, ClientSecret および Realm を使用して、アクセストークンを取得する流れとなります。

```csharp
    //Get the realm for the URL
    string realm = TokenHelper.GetRealmFromTargetUrl(new Uri(siteUrl));

    //Get the access token for the URL.  
    string accessToken = TokenHelper.GetAppOnlyAccessToken(TokenHelper.SharePointPrincipal, new Uri(siteUrl).Authority, realm).AccessToken;

    //Create a client context object based on the retrieved access token
    using (ClientContext cc = TokenHelper.GetClientContextWithAccessToken(siteUrl, accessToken))
    {
        cc.Load(cc.Web, p => p.Title);
        cc.ExecuteQuery();
        Console.WriteLine(cc.Web.Title);
    }
```

エラーが発生する流れとして、まずは TLS によりトランスポート層で拒否されることにより、TokenHelper.GetReamFromTargetUrl メソッド内で WebException (内部例外は IOException と SocketException) が返されます。
しかし、該当メソッドでは WebException をメソッド外にスローすることはありません。該当リクエストは HTTP 応答 401 が返ることを想定したコードであるため、内部で例外を補足し、HTTP 応答ヘッダー WWW-Authenticate の値から Realm の情報を読み取ろうとします。

```csharp
public static string GetRealmFromTargetUrl(Uri targetApplicationUri)
{
	WebRequest request = WebRequest.Create(targetApplicationUri + "/_vti_bin/client.svc");
	request.Headers.Add("Authorization: Bearer ");

	try
	{
		using (request.GetResponse())
		{
		}
	}
	catch (WebException e)
	{
		if (e.Response == null)
		{
			return null;
		}

		string bearerResponseHeader = e.Response.Headers["WWW-Authenticate"];
		if (string.IsNullOrEmpty(bearerResponseHeader))
		{
			return null;
		}

		const string bearer = "Bearer realm=\"";
		int bearerIndex = bearerResponseHeader.IndexOf(bearer, StringComparison.Ordinal);
		if (bearerIndex < 0)
		{
			return null;
		}

		int realmIndex = bearerIndex + bearer.Length;

		if (bearerResponseHeader.Length >= realmIndex + 36)
		{
			string targetRealm = bearerResponseHeader.Substring(realmIndex, 36);

			Guid realmGuid;

			if (Guid.TryParse(targetRealm, out realmGuid))
			{
				return targetRealm;
			}
		}
	}
	return null;
}
```

しかし、トランスポート層で通信が拒否されます。アプリケーション層での通信は実行されないため、SharePoint Online サーバーから HTTP 応答ヘッダー WWW-Authenticate などの値は返されません。このことにより Realm が null として返されます。

その結果、次の GetAppOnlyAccessToken メソッドに targetRealm が null が渡されます。
ACS に渡されるクライアント ID は、ClientID@Realm の形式で渡されます。この値が不正な形式 (ClientID@) となるため、HTTP 応答コード 401 と共に RequestFailedException が返されます。


## 参考情報
タイトル : Microsoft 365 の TLS 1.0 と 1.1 を無効にする
アドレス : https://docs.microsoft.com/ja-jp/microsoft-365/compliance/tls-1.0-and-1.1-deprecation-for-office-365?view=o365-worldwide

タイトル : Office 365 および Office 365 GCC での TLS 1.2 の準備
アドレス : https://docs.microsoft.com/ja-jp/microsoft-365/compliance/prepare-tls-1.2-in-office-365?view=o365-worldwide


今回の投稿は以上です。
