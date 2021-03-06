---
title: author: description: monikerRange: ms.author: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

---
# <a name="use-grpc-in-browser-apps"></a>ブラウザー アプリでの gRPC の使用

作成者: [James Newton-King](https://twitter.com/jamesnk)

ブラウザーベースのアプリから HTTP/2 gRPC サービスを呼び出すことはできません。 [gRPC-Web](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-WEB.md) は、ブラウザーの JavaScript および Blazor アプリで gRPC サービスを呼び出せるようにするプロトコルです。 この記事では、.NET Core で gRPC-Web を使用する方法について説明します。

## <a name="grpc-web-in-aspnet-core-vs-envoy"></a>ASP.NET Core の gRPC-Web と Envoy

gRPC-Web を ASP.NET Core アプリに追加する方法には、次の 2 つの選択肢があります。

* ASP.NET Core の gRPC HTTP/2 と共に、gRPC-Web をサポートします。 このオプションでは、`Grpc.AspNetCore.Web` パッケージによって提供されるミドルウェアを使用します。
* [Envoy プロキシ](https://www.envoyproxy.io/)の gRPC-Web サポートを使用して、gRPC-Web を gRPC HTTP/2 に変換します。 変換された呼び出しが、ASP.NET Core アプリに転送されます。

それぞれの方法には、長所と短所があります。 アプリの環境で既にプロキシとして Envoy を使用している場合は、gRPC-Web サポートを提供するためにこれを使用するのも、妥当かもしれません。 ASP.NET Core のみを必要とする gRPC-Web の単純なソリューションが必要な場合は、`Grpc.AspNetCore.Web` を選択することをお勧めします。

## <a name="configure-grpc-web-in-aspnet-core"></a>ASP.NET Core での gRPC-Web の構成

ASP.NET Core でホストされている gRPC サービスは、HTTP/2 gRPC と共に gRPC-Web をサポートするように構成できます。 gRPC-Web では、サービスを変更する必要はありません。 唯一の変更点はスタートアップ構成です。

ASP.NET Core gRPC サービスで gRPC-Web を有効にするには:

* [Grpc.AspNetCore.Web](https://www.nuget.org/packages/Grpc.AspNetCore.Web) パッケージへの参照を追加します。
* アプリで gRPC-Web を使用するように構成するには、`UseGrpcWeb` と `EnableGrpcWeb` を *Startup.cs* に追加します。

[!code-csharp[](~/grpc/browser/sample/Startup.cs?name=snippet_1&highlight=10,14)]

上記のコードでは次の操作が行われます。

* GRPC-Web ミドルウェアの `UseGrpcWeb` を、ルーティングの後かつエンドポイントの前に追加します。
* `endpoints.MapGrpcService<GreeterService>()` メソッドで、`EnableGrpcWeb` を使用して gRPC-Web をサポートすることを指定します。 

または、gRPC-Web ミドルウェアを構成して、すべてのサービスで既定で gRPC-Web がサポートされ、`EnableGrpcWeb` が不要になるようにすることもできます。 ミドルウェアを追加するときに `new GrpcWebOptions { DefaultEnabled = true }` を指定します。

[!code-csharp[](~/grpc/browser/sample/AllServicesSupportExample_Startup.cs?name=snippet_1&highlight=12)]

> [!NOTE]
> .NET Core 3.x で [Http.sys によりホストされている](xref:fundamentals/servers/httpsys)場合は、gRPC-Web が失敗する原因となる既知の問題があります。
>
> Http.sys で gRPC-Web を機能させるには、[こちら](https://github.com/grpc/grpc-dotnet/issues/853#issuecomment-610078202)で回避策を参照してください。

### <a name="grpc-web-and-cors"></a>gRPC-Web と CORS

ブラウザーのセキュリティにより、Web ページを提供したドメインと異なるドメインに対して、Web ページが要求を行うことはできません。 この制限は、ブラウザー アプリで gRPC-Web 呼び出しを行う場合に適用されます。 たとえば、`https://www.contoso.com` によって提供されるブラウザー アプリでは、`https://services.contoso.com` でホストされている gRPC-Web サービスの呼び出しがブロックされます。 この制限を緩和するには、クロス オリジン リソース共有 (CORS) を使用できます。

ブラウザー アプリでクロス オリジン gRPC-Web 呼び出しを行えるようにするには、[ASP.NET Core で CORS](xref:security/cors) を設定します。 組み込みの CORS サポートを使用し、<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder.WithExposedHeaders*> で gRPC 固有のヘッダーを公開します。

[!code-csharp[](~/grpc/browser/sample/CORS_Startup.cs?name=snippet_1&highlight=5-11,19,24)]

上記のコードでは次の操作が行われます。

* `AddCors` を呼び出して CORS サービスを追加し、gRPC 固有のヘッダーを公開する CORS ポリシーを構成します。
* `UseCors` を呼び出して、ルーティングの後かつエンドポイントの前に CORS ミドルウェアを追加します。
* `endpoints.MapGrpcService<GreeterService>()` メソッドが `RequiresCors` で CORS をサポートすることを指定します。

## <a name="call-grpc-web-from-the-browser"></a>ブラウザーから gRPC-Web を呼び出す

ブラウザー アプリでは gRPC-Web を使用して gRPC サービスを呼び出すことができます。 ブラウザーから gRPC-Web を使用して gRPC サービスを呼び出す場合、いくつかの要件と制限があります。

* サーバーは、gRPC-Web をサポートするように構成されている必要があります。
* クライアント ストリーミングと双方向ストリーミングの呼び出しはサポートされていません。 サーバー ストリーミングはサポートされています。
* 別のドメインで gRPC サービスを呼び出すには、サーバーで [CORS](xref:security/cors) を構成する必要があります。

### <a name="javascript-grpc-web-client"></a>JavaScript gRPC-Web クライアント

JavaScript gRPC-Web クライアントがあります。 JavaScript から gRPC-Web を使用する方法については、[gRPC-Web を使用した JavaScript クライアント コードの作成](https://github.com/grpc/grpc-web/tree/master/net/grpc/gateway/examples/helloworld#write-client-code)に関するページを参照してください。

### <a name="configure-grpc-web-with-the-net-grpc-client"></a>.NET gRPC クライアントを使用して gRPC-Web を構成する

gRPC-Web 呼び出しを行うように .NET gRPC クライアントを構成できます。 これは、ブラウザーでホストされ、JavaScript コードの同じ HTTP 制限がある [Blazor WebAssembly](xref:blazor/index#blazor-webassembly) アプリに役立ちます。 .NET クライアントによる gRPC-Web の呼び出しは、[HTTP/2 gRPC と同じ](xref:grpc/client)です。 唯一の変更点は、チャネルの作成方法です。

gRPC-Web を使用するには:

* [Grpc.Net.Client.Web](https://www.nuget.org/packages/Grpc.Net.Client.Web) パッケージへの参照を追加します。
* [Grpc.Net.Client](https://www.nuget.org/packages/Grpc.Net.Client) パッケージへの参照が確実に 2.29.0 以上であるようにします。
* `GrpcWebHandler` を使用するようにチャネルを構成します。

[!code-csharp[](~/grpc/browser/sample/Handler.cs?name=snippet_1)]

上記のコードでは次の操作が行われます。

* gRPC-Web を使用するようにチャネルを構成します。
* クライアントを作成し、チャネルを使用して呼び出しを行います。

`GrpcWebHandler` には次の構成オプションがあります。

* **InnerHandler**:gRPC HTTP 要求を行う基になる <xref:System.Net.Http.HttpMessageHandler> (`HttpClientHandler` など)。
* **GrpcWebMode**:gRPC HTTP 要求の `Content-Type` が `application/grpc-web` または `application/grpc-web-text` であるかどうかを指定する列挙型。
    * `GrpcWebMode.GrpcWeb` は、コンテンツをエンコードせずに送信するように構成します。 既定値です。
    * `GrpcWebMode.GrpcWebText` は、コンテンツを base64 でエンコードするように構成します。 ブラウザーでのサーバー ストリーム呼び出しに必要です。
* **HttpVersion**:基になる gRPC HTTP 要求で、[HttpRequestMessage](xref:System.Net.Http.HttpRequestMessage.Version) を設定するために使用される HTTP プロトコル `Version`。 gRPC-Web では特定のバージョンを必要とせず、指定しない限り、既定値はオーバーライドされません。

> [!IMPORTANT]
> 生成された gRPC クライアントには、単項メソッドを呼び出すための同期メソッドと非同期メソッドがあります。 たとえば、`SayHello` は同期であり、`SayHelloAsync` は非同期です。 Blazor WebAssembly アプリで同期メソッドを呼び出すと、アプリが応答しなくなります。 Blazor WebAssembly では、常に非同期メソッドを使用する必要があります。

## <a name="additional-resources"></a>その他の技術情報

* [Web クライアント用 gRPC の GitHub プロジェクト](https://github.com/grpc/grpc-web)
* <xref:security/cors>
