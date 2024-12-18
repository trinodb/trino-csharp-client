# Trino C# Client

## Introduction

A streaming C# Trino client library with ADO.NET interfaces. The priorties for this library are:

* Response performance (fast return of rows for both short and long running queries).
* Type support in .NET (with complex types, date time precision) and System.Data types.
* Authentication which is customizable and flexible (to support arbitrary custom token refresh, or unique cloud requirements).
* Full ADO.NET implementation.
* Streaming with read-ahead into customizable buffer size to allow for no-wait client paging.
* Session support, parameterized query support.
* No dependencies outside of .NET core except Newtonsoft.Json and Microsoft.Extensions.Logging. Authentication in separate library.
* Alignment with Trino Java client, similar class structure.

## Resources

Project introduction at Trino Summit 2024 by George Fischer

[![YouTube](https://img.youtube.com/vi/x2rF6IEjFK0/0.jpg)](https://www.youtube.com/watch?v=x2rF6IEjFK0)


## Libraries

These are the libraries that make up the client:

|Library|Description|Compatibility|Notes|
|-|-|-|-|
|Trino.Client|Trino .NET SDK|.NET Standard 2.0|Core client library providing direct access to Trino protocols. Handles session management, query execution, and result streaming.|
|Trino.Data.ADO|Trino ADO.NET client|.NET Standard 2.0|ADO.NET wrapper implementing DbConnection, DbCommand, and IDataReader. Provides familiar database access patterns and schema discovery functionality.|
|Trino.Client.Auth|Authentication providers|.NET Standard 2.0|Suthentication implementations in a separate package to avoid dependency conflicts.|

> [!NOTE]
> .NET standard 2.0 provides compatibility with .NET Framework 4.7.2 which is widely used.
> Design note: IAsyncEnumerable is not available in .NET Standard 2.0 but due to asynchronous read-ahead you do not need to await reading every row.

## Building

### Command Line

1. Install .NET SDK latest version from <https://dotnet.microsoft.com/en-us/download>
2. From the root folder of the project, run `msbuild TrinoDriver.sln`

To make the nuget packages:

```cmd
nuget pack Trino.Client\Trino.Client.nuspec -Version 1.0.0
nuget pack Trino.Data.ADO\Trino.Data.ADO.nuspec -Version 1.0.0
nuget pack Trino.Client.Auth\Trino.Client.Auth.nuspec -Version 1.0.0
```

### Visual Studio

In Visual Studio, open TrinoDriver.sln and build.

### Visual Studio Code

1. Install .NET SDK latest version from <https://dotnet.microsoft.com/en-us/download>
2. Install the C# extension for VS Code
3. Open the project folder in VS Code
4. Select a build configuration using the .NET: Select Project command
5. Build using Terminal > Run Build Task or `dotnet build TrinoDriver.sln`


## Usage Examples

The client library provides two ways to interact with Trino:

1. ADO.NET Interface (Recommended for .NET applications) - Provides standardized database access through familiar ADO.NET abstractions (DbConnection, DbCommand, IDataReader). Use this if you want code that's consistent with other database providers or need to work with existing ADO.NET-based tools and frameworks. See the [ADO.NET Connection](#adonet-connection) section for detailed configuration options.

2. Trino SDK - Direct access to Trino functionality through a lightweight client implementation. This approach offers more fine-grained control and aligns closely with other Trino client implementations.

### Quick Start Using ADO.NET

```csharp
TrinoConnectionProperties properties = new TrinoConnectionProperties()
{
    Catalog = "tpch",
    Server = new Uri("https://trino.myhost.net/"),
    Auth = new TrinoJWTAuth(token: "...")
};

using (TrinoConnection connection = new TrinoConnection(properties))
{
    using (IDbCommand command = new TrinoCommand(connection, "SELECT * FROM tpch.tiny.customer LIMIT 5"))
    using (IDataReader reader = command.ExecuteReader())
    {
        while (reader.Read())
        {
            for (int i = 0; i < reader.FieldCount; i++)
            {
                Console.WriteLine($"{reader.GetName(i)} -> {reader.GetDataTypeName(i)} : {reader.GetValue(i)}");
            }
        }
    }
}

### Quick Start Using Trino SDK

#### Basic Query Execution

```csharp
// Create a session with default settings
ClientSession session = new ClientSession(
    sessionProperties: new ClientSessionProperties()
    {
        Server = new Uri("http://localhost:8080/")
    });

// Execute query and read results into a DataTable
DataTable dt = await(
    await RecordExecutor.Execute(
        session: session,
        statement: "select * from tpch.tiny.customer"
        ).ConfigureAwait(false)
    ).BuildDataTableAsync().ConfigureAwait(false);
Console.WriteLine($"Read {dt.Rows.Count} rows");
```

#### Row-by-Row Processing

```csharp
RecordExecutor records = await RecordExecutor.Execute(
        session: session,
        statement: "select * from tpch.tiny.customer"
        ).ConfigureAwait(false);

// Results stream automatically with configurable buffer
foreach (var row in records)
{
    Console.WriteLine(string.Join(",", row));
}
```

#### Advanced SDK Features

For more complex scenarios, the SDK provides direct access to Trino features:

```csharp
ITrinoAuth auth = new TrinoJWTAuth() { AccessToken = "token" };

ClientSession session = new TrinoConnectionProperties()
{
    Catalog = "tpch",
    Server = new Uri("https://trino.myhost.net/"),
    Auth = auth,
    SessionProperties = new Dictionary<string, string>() 
    { 
        { "dictionary_aggregation", "false" } 
    }
}.GetSession();

RecordExecutor records = await RecordExecutor.Execute(
    logger: null, // optional detailed logging
    queryStatusNotifications: [(stats, error) => 
    { 
        Console.WriteLine($"Processed rows: {stats.processedRows}"); 
    }], // Output statistics to console as query runs
    session: session,
    statement: "select * from tpch.tiny.customer where custkey = ?",
    queryParameters: new List<QueryParameter>() { new(751) },
    bufferSize: 1024 * 1024 * 5, // 5 MB
    isQuery: true,
    CancellationToken.None).ConfigureAwait(false);

DataTable dt = await records.BuildDataTableAsync().ConfigureAwait(false);
Console.WriteLine("Data table created with rows: " + dt.Rows.Count);
```

## ADO.NET Connection

The ADO.NET implementation provides standard DbConnection, DbCommand, and DataReader interfaces for Trino. This allows you to use Trino with existing ADO.NET-based tools and patterns.

### Connection Properties

When establishing an ADO.NET connection you declare your connection using the `TrinoConnectionProperties` object which can be serialized into a query string and can be set using a query string. Example:

```csharp
TrinoConnectionProperties properties = new TrinoConnectionProperties()
{
    Server = new Uri("https://trino.myhost.net/"),
    Auth = new TrinoJWTAuth(token: "...")
};
```

or

```csharp
TrinoConnectionProperties properties = new TrinoConnectionProperties()
{
    Server = new Uri("http://localhost:8080/")
};
```

### Connection String

The Trino ADO.NET library supports connection strings. The connection string is expressed as semicolon delimetered assignments. The server name needs to be broken down into `host`, `port`, and `enablessl` properties.

> [!NOTE] Any auth class used in the connections string must be in a loaded assembly. If you have a custom authenticator it must be in a loaded assembly in order to be discovered.

Example connection using a connection string:

```csharp
using (TrinoConnection tc = new TrinoConnection())
{
    tc.ConnectionString = $"catalog=delta;schema=nyc;host={demoClusterHost};auth=TrinoJWTAuth;AccessToken={token}";
    using (IDbCommand trinoCommand = new TrinoCommand(tc, all_types, TimeSpan.MaxValue, null, logger))
    {
        IDataReader idr = trinoCommand.ExecuteReader();
        while (idr.Read())
        {
            // consume data
        }
    }
}
```

### Session Management

The connection maintains session properties that affect query execution. These can be set through properties or SQL commands:

```csharp
using (TrinoConnection connection = new TrinoConnection(properties))
{
    // Direct property setting
    connection.ConnectionSession.Properties.Properties["query_max_memory"] = "1GB";
    
    // Using SQL commands
    using (IDbCommand command = new TrinoCommand(connection, "SET SESSION query_max_memory = '2GB'"))
    {
        command.ExecuteNonQuery();
    }
}
```

### Connection String/TrinoConnectionProperties

|Property|Description|Example Connection Property|Example Connection String|
|-|-|-|-|
|AdditionalHeaders|Key value pairs to be sent to the Trino endpoint.|`new Dictionary<string, string>() {{"key", "value"}}`|n/a|
|AllowHostNameCNMismatch|Allows the hostname not to match the Common Name (CN) or Subject Alternative Name (SAN) listed on the certificate|false|`AllowHostNameCNMismatch=false`|
|AllowSelfSignedServerCert|The server’s SSL/TLS certificate does not need to be issued by a trusted Certificate Authority (CA) and can be self-signed instead.|false|`AllowSelfSignedServerCert=false`|
|Auth|Authentication method. Must implement ITrinoAuth. See [Authentication](#authentication).|new TrinoAzureDefaultAuth(scope: "https://1c9dee158a58484e87f84fb40997b520/.default")|`auth=TrinoJWTAuth;accessToken=token;`|
|Catalog|Trino default catalog|`catalog=tpch`|
|ClientTags|A list of tags associated with the client|`new List<string>() {"tag1", "tag2"}`|`clienttags=tag1,tag2`|
|CompressionDisabled|Disables compressed communication between Trino and the client.|false|`compressiondisabled=true`|
|Host|Trino cluster hostname.|trino.myhost.net|`host=host.trino.com`|
|Port|Trino cluster port. Default 443.|443|`port=443`|
|Path|Trino cluster path. Default unset.|my/host/path|`path=my/host/path`|
|EnableSsl|If https should be enabled. Default https.|https|`enableSsl=true`|
|Roles|Authorization roles to use for catalogs, specified as a list of key-value pairs for the catalog and role. For example, catalog1:roleA;catalog2:roleB sets roleA for catalog1 and roleB for catalog2.|`new Dictionary<string, ClientSelectedRole>() {{"catalog1", new ClientSelectedRole(Type.ROLE, "roleA") }}`|n/a|
|Schema|Trino default schema|sf1|`schema=sf1`|
|SessionProperties|Trino session properties initial values.|`new Dictionary<string, string>() { { "query_cache_enabled", "true" }, { "query.cache.ttl", "1h" } }`|`properties=property1:true,property2:100`|
|Source|Identifier of the client request to be associated with the query|my_service|`source=myclient`|
|TestConnection|Connection is tested when Open() is called. Slows down the query.|false|`testconnection=true`|
|TimeZone|Timezone of Trino client.|`TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time")`|`timezone=UTC`|
|TraceToken|Token for tracing service integration flow|f1ec1fc3-60f2-48a2-95e8-663c29146351|`tracetoken=f1ec1fc3-60f2-48a2-95e8-663c29146351`|
|Server|Alternate method of providing host, port, and enableSsl|`new Uri("https://trino.myhost.net");`|n/a|
|User|The user connecting to the cluster.|user_name|`user=user_name`|

## Authentication

Authentication is handled through implementations of `ITrinoAuth`. Each provider must implement:

* `AuthorizeAndValidate()`: Initializes or refreshes authorization
* `AddCredentialToRequest()`: Adds credentials to each HTTP request

Built-in providers include:

#### JWT Bearer Token
Simple token-based authentication:
```csharp
Auth = new TrinoJWTAuth(token: "your-token-here")
```

You can implement token refresh:
```csharp
TrinoJWTAuth auth = new TrinoJWTAuth(getInitialToken());
using (TrinoConnection connection = new TrinoConnection(properties))
{
    // Setup automatic refresh every 45 minutes
    Task.Run(async () =>
    {
        while (true)
        {
            await Task.Delay(TimeSpan.FromMinutes(45));
            auth.AccessToken = getNewToken();
        }
    });
}
```

#### Azure Default Credentials

Uses Azure Identity with automatic token management:

```csharp
Auth = new TrinoAzureDefaultAuth(scope: "https://myapplication/.default")
```

#### OAuth Client Credentials

OAuth 2.0 client credentials flow:

```csharp
Auth = new TrinoOauthClientSecretAuth(
    tokenEndpoint: "login.microsoftonline.com",
    clientId: "client-id",
    clientSecret: "client-secret",
    scope: "https://myscope/")
```

#### Custom Authentication

You can implement custom authentication by implementing ITrinoAuth:

```csharp
public class CustomAuth : ITrinoAuth
{
    public void AuthorizeAndValidate()
    {
        // Initialize or refresh authorization
        // Called when connection is opened
    }

    public void AddCredentialToRequest(HttpRequestMessage request)
    {
        // Add authentication headers or other credentials
        // Called for each request to Trino
        request.Headers.Add("Authorization", "Custom " + GetCredential());
    }
}
```

Using the custom auth provider:

```csharp
properties.Auth = new CustomAuth();
```

When using connection strings, any auth class must be in a loaded assembly to be discovered.

### Session Properties

Session properties exist for the duration of the Trino connection. These can be set directly in a SQL command such as `USE tpch.sf1` or `SET SESSION ...`, but you can set and consume these properties directly on the connection.

Example setting session properties:

```csharp
using (TrinoConnection connection = new TrinoConnection(properties))
{
    connection.ConnectionSession.Properties.Properties.Add("query_cache_enabled", "false");
    connection.ConnectionSession.Properties.Source = "Client name";

    ...
}
```

Example reading session properties that are set as part of a query:

```csharp
using (TrinoConnection connection = new TrinoConnection(properties))
{
    using (IDbCommand trinoCommand = new TrinoCommand(connection, "USE tpch.sf10"))
    {
        trinoCommand.ExecuteNonQuery();
    }
    using (IDbCommand trinoCommand = new TrinoCommand(connection, "SET SESSION hive.insert_existing_partitions_behavior = 'OVERWRITE'"))
    {
        trinoCommand.ExecuteNonQuery();
    }
    Console.WriteLine($"Catalog: {connection.ConnectionSession.Properties.Catalog}, Schema: {connection.ConnectionSession.Properties.Schema}");
    Console.WriteLine(string.Join(", ", connection.ConnectionSession.Properties.Properties.Keys.ToArray()));
```

Output:

```info
Catalog: tpch, Schema: sf10
hive.insert_existing_partitions_behavior
```

|Session Property|Notes|
|-|-|
|AdditionalHeaders|Key-value pairs of additional HTTP headers to send with requests|
|AuthorizationUser|The authorized user for the session|
|Catalog|Default catalog for unqualified table names. Can also be set via connection properties|
|ClientInfo|Additional information about the client application|
|ClientRequestTimeout|Timeout duration for client requests|
|ClientTags|Set of tags associated with the client session|
|CompressionDisabled|Whether HTTP compression is disabled for responses|
|ExtraCredentials|Dictionary of additional credentials used by the query|
|Locale|Locale setting for the session|
|Path|SQL path for the session, useful for catalog/schema context|
|PreparedStatements|Dictionary of prepared statements created using SQL `PREPARE ...` syntax|
|Principal|Principal identity for the session|
|Properties|Dictionary of Trino session properties that control query behavior|
|ResourceEstimates|Dictionary of resource estimates for query planning|
|Roles|Dictionary mapping catalogs to their selected roles|
|Schema|Default schema for unqualified table names. Can also be set via connection properties|
|Server|URI of the Trino server|
|Source|Identifier for the client application making the request|
|TimeZone|Timezone for query processing (string format, compatible with Java's ZoneId)|
|TraceToken|Token used for query tracing and debugging|
|TransactionId|ID of the current transaction if in a transaction|
|User|User identity for the session (may be overridden by server)|

### Streaming

Results are consumed from the IDataReader interface. As the rows are consumed Trino will asynchronously produce results which are then asynchronously pulled by the client. The default buffer size is 50MB (defined as MaxTargetResultSizeMB * 10), which means the client will pull in 50MB of data before blocking until those rows are consumed from the client.

To customize the buffer size, provide the byte size of the buffer when creating the reader. If you provide a value of 0, the client will read one page ahead and wait until that page is consumed.

```csharp
IDataReader idr = trinoCommand.ExecuteReader(1024 * 1024 * 100); // 100 mb
```
