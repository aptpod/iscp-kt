# Module iscp

# iSCP 2.0 Client Library for Kotlin

iSCP Client for Kotlin は、iSCP version 2を用いたリアルタイムAPIにアクセスするためのクライアントライブラリです。

## Dependencies

- [Protocol Buffers [Core]](https://mvnrepository.com/artifact/com.google.protobuf/protobuf-java)
- [Protocol Buffers [Util]](https://mvnrepository.com/artifact/com.google.protobuf/protobuf-java-util)
- [Java WebSockets](https://mvnrepository.com/artifact/org.java-websocket/Java-WebSocket)

## Installation

```
// build.gradle
...
dependencies {
    ...
    // Install iSCP
    implementation 'com.aptpod.github:iscp:0.11.0'
}
```

```
// settings.gradle
...
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        // Library references for iSCP
        maven { url "https://aptpod.github.io/iscp-kt" }
    }
}
```

## Implementation

### Connect to intdash API

このサンプルではiscp-ktを使ってintdash APIに接続します。

```kotlin
import android.app.Activity
import com.aptpod.iscp.connection.Connection
import com.aptpod.iscp.connection.ConnectionCallbacks
import com.aptpod.iscp.transport.ITransportConfig
import com.aptpod.iscp.transport.WebSocketConfig

class ExampleActivity : Activity() {
    /**
     * 接続するintdashサーバー
     */
    var targetServer: String = "https://example.com"
    /**
     * ノードUUID（ここで指定されたノードとして送受信を行います。
     *
     * intdash APIでノードを生成した際に発行されたノードUUIDを指定します。）
     */
    var nodeId = "00000000-0000-0000-0000-000000000000"
    /**
     * アクセストークン
     *
     * intdash APIで取得したアクセストークンを指定して下さい。
     */
    var accessToken = ""
    /**
     *  コネクション
     */
    var connection: Connection? = null

    fun connect() {
        // 接続情報のセットアップをします。
        var urls = targetServer.split("://")
        var address: String
        var enableTls: Boolean = false
        if (urls.size == 1)
        {
            address = urls[0]
        }
        else
        {
            enableTls = urls[0] == "https"
            address = urls[1]
        }
        // WebSocketを使って接続するように指定します。
        var transportConfig: ITransportConfig = WebSocketConfig(enableTls = enableTls)
        Connection.connectAsync(
            address = address,
            transportConfig = transportConfig,
            tokenSource = {
                // アクセス用のトークンを指定します。接続時に発生するイベントにより使用されます。
                // ここでは固定のトークンを返していますが随時トークンの更新を行う実装にするとトークンの期限切れを考える必要がなくなります。
                accessToken
            },
            nodeId = nodeId,
            completion = { con, ex ->
                if (con == null) {
                    // 接続失敗
                    return@connectAsync
                }
                // 接続成功.
                connection = con
                con.callbacks = this.connectionCallbacks // ConnectionCallbacks
                // 以降、startUpstreamやstartDownstreamなどが実行可能になります。
            }
        )
    }

    val connectionCallbacks : ConnectionCallbacks
        get() = object : ConnectionCallbacks {
            override fun onReconnect(connection: Connection) {
                // Connectionが再オープンされた際にコールされます。
            }

            override fun onDisconnect(connection: Connection) {
                // Connectionがクローズされた際にコールされます。
            }

            override fun onFailWithError(connection: Connection, exception: Exception) {
                // Connection内部で何らかのエラーが発生した際にコールされます。
            }
        }
}
```

### Start Upstream

アップストリームの送信サンプルです。

このサンプルでは、基準時刻のメタデータと、文字列型のデータポイントをiSCPサーバーへ送信しています。

```kotlin
import com.aptpod.iscp.model.BaseTime
import com.aptpod.iscp.model.DataId
import com.aptpod.iscp.model.DataPoint
import com.aptpod.iscp.model.UpstreamChunk
import com.aptpod.iscp.model.UpstreamChunkAck
import com.aptpod.iscp.stream.Upstream
import com.aptpod.iscp.stream.UpstreamCallbacks
import java.time.ZonedDateTime
import java.util.UUID

/**
 * 送信するデータを永続化するかどうか
 */
var upstreamPersist: Boolean = false
/**
 * オープンしたストリーム一覧
 */
var upstreams: MutableList<Upstream> = mutableListOf()

fun ExampleActivity.startUpstream() {
    // セッションIDを払い出します。
    var sessionId = UUID.randomUUID().toString().lowercase()

    // Upstreamをオープンします。
    connection?.openUpstreamAsync(
        sessionId = sessionId,
        persist = upstreamPersist,
        completion = { upstream, ex ->
            if (upstream == null) {
                // オープン失敗。
                return@openUpstreamAsync
            }
            // オープン成功
            upstreams.add(upstream)

            // 送信するデータポイントを保存したい場合や、アップストリームのエラーをハンドリングしたい場合はコールバックを設定します。
            upstream.callbacks = this.upstreamCallbacks // UpstreamCallbacks

            var date = ZonedDateTime.now() // ※マイクロ秒以下をサポートするには修正が必要です。
            var baseTime = (date.toEpochSecond() * 1000_000_000) + date.nano // 基準時刻です。

            // 基準時刻をiSCPサーバーへ送信します。
            connection?.sendBaseTimeAsync(
                baseTime = BaseTime(
                    sessionId = sessionId,
                    name = "manual",
                    priority = 1000,
                    elapsedTime = 0,
                    baseTime = baseTime),
                persist = upstreamPersist,
                completion = { sendBaseTimeEx ->
                    if (sendBaseTimeEx != null) {
                        // 基準時刻の送信に失敗。
                        return@sendBaseTimeAsync
                    }
                    // 基準時刻の送信に成功。

                    // 文字列型のデータポイントをiSCPサーバーへ送信します。
                    var now = ZonedDateTime.now() // ※マイクロ秒以下をサポートするには修正が必要です。
                    upstream.writeDataPoint(
                        dataId = DataId(
                            name = "greeting",
                            type = "string"),
                        dataPoint = DataPoint(
                            elapsedTime = ((now.toEpochSecond() * 1000_000_000) + now.nano) - baseTime, // 基準時刻からの経過時間をデータポイントの経過時間として打刻します。
                            payload = "hello".toByteArray()
                        )
                    )
                }
            )
        }
    )
}

val ExampleActivity.upstreamCallbacks : UpstreamCallbacks
    get() = object : UpstreamCallbacks {
        override fun onGenerateChunk(upstream: Upstream, message: UpstreamChunk) {
            // バッファへ書き込んだデータポイントが実際に送信される直前にコールされます。
        }

        override fun onReceiveAck(upstream: Upstream, message: UpstreamChunkAck) {
            // データポイントの送信後に返却されるACKを受信できた場合にコールされます。
        }

        override fun onFailWithError(upstream: Upstream, error: Exception) {
            // 内部でエラーが発生した場合にコールされます。
        }

        override fun onCloseWithError(upstream: Upstream, error: Exception) {
            // 何らかの理由でストリームがクローズした場合にコールされます。
            // 再度アップストリームをオープンしたい場合は、 `Connection.reopenUpstream()` を使用することにより、ストリームの設定を引き継いで別のストリームを開くことが可能です。
        }

        override fun onResume(upstream: Upstream) {
            // 自動再接続機能が働き、再接続が行われた場合にコールされます。
        }
    }
```

### Start Downstream

アップストリームで送信されたデータをダウンストリームで受信するサンプルです。

このサンプルでは、アップストリーム開始のメタデータ、基準時刻のメタデータ、文字列型のデータポイントを受信しています。

```kotlin
import android.util.Log
import com.aptpod.iscp.model.DataFilter
import com.aptpod.iscp.model.DownstreamChunk
import com.aptpod.iscp.model.DownstreamFilter
import com.aptpod.iscp.model.DownstreamMetadata
import com.aptpod.iscp.stream.Downstream
import com.aptpod.iscp.stream.DownstreamCallbacks
import java.time.Instant
import java.time.ZoneId
import java.time.ZonedDateTime

/**
 * 受信したいデータを送信している送信元ノードのUUID
 *
 * （アップストリームを行っている送信元でConnection.Configで設定したnodeIdを指定してください。）
 */
var targetDownstreamNodeID = "00000000-0000-0000-0000-000000000000"
/**
 * オープンしたダウンストリーム一覧
 */
var downstreams: MutableList<Downstream> = mutableListOf()

fun ExampleActivity.startDownstream() {
    // ダウンストリームをオープンします。
    connection?.openDownstreamAsync(
        downstreamFilters = listOf(
            DownstreamFilter(
                sourceNodeId = targetDownstreamNodeID, // // 送信元ノードのIDを指定します。
                dataFilters = listOf(
                    DataFilter(
                        name = "#", type = "#") // 受信したいデータを名称と型で指定します。この例では、ワイルドカード `#` を使用して全てのデータを取得します。
                )
            )
        ),
        completion = { downstream, ex ->
            if (downstream == null) {
                // オープン失敗。
                return@openDownstreamAsync
            }
            // オープン成功。
            downstreams.add(downstream)
            // 受信データを取り扱うためにデリゲートを設定します。
            downstream.callbacks = this.downstreamCallbacks // DownstreamCallbacks
        }
    )
}

val ExampleActivity.downstreamCallbacks: DownstreamCallbacks
    get() = object : DownstreamCallbacks {
        override fun onReceiveChunk(downstream: Downstream, message: DownstreamChunk) {
            // データポイントを読み込むことができた際にコールされます。
            Log.d(this.javaClass.name, "Received dataPoints sequenceNumber[${message.sequenceNumber}], sessionId[${message.upstreamInfo.sessionId}]")
            for (g in message.dataPointGroups) {
                for (dp in g.dataPoints) {
                    Log.d(this.javaClass.name, "Received a dataPoint dataName[${g.dataId.name}], dataType[${g.dataId.type}], payload[${String(dp.payload)}]")
                }
            }
        }

        override fun onReceiveMetadata(downstream: Downstream, message: DownstreamMetadata) {
            // メタデータを受信した際にコールされます。
            Log.d(this.javaClass.name, "Received a metadata sourceNodeId[${message.sourceNodeId}], metadataType:${message.type}")
            when (message.type) {
                DownstreamMetadata.MetadataType.BASE_TIME -> {
                    var baseTime = message.baseTime!!
                    var date = ZonedDateTime.ofInstant(Instant.ofEpochSecond(baseTime.baseTime / 1000_000_000, baseTime.baseTime % 1000_000_000), ZoneId.systemDefault())
                    Log.d(this.javaClass.name, "Received baseTime[$date], priority[${baseTime.priority}], name[${baseTime.name}]")
                }
                else -> {}
            }
        }

        override fun onFailWithError(downstream: Downstream, error: Exception) {
            // 内部でエラーが発生した場合にコールされます。
        }

        override fun onCloseWithError(downstream: Downstream, error: Exception) {
            // 何らかの理由でストリームがクローズした場合にコールされます。
            // 再度ダウンストリームをオープンしたい場合は、 `Connection.reopenDownstream()` を使用することにより、ストリームの設定を引き継いで別のストリームを開くことが可能です。
        }

        override fun onResume(downstream: Downstream) {
            // 自動再接続機能が働き、再接続が行われた場合にコールされます。
        }
    }
```


### E2E Call

E2E（エンドツーエンド）コールのサンプルです。

コントローラノードが対象ノードに対して指示を出し、対象ノードは受信完了のリプライを行う簡単なサンプルです。

```kotlin
class E2ECallExampleActivity  : Activity() {
    /**
     * 接続するintdashサーバー
     */
    var targetServer: String = "https://example.com"

    /**
     * コントローラーノードのUUID
     */
    var controllerNodeID: String = "00000000-0000-0000-0000-000000000000"
    /**
     * 対象ノードのUUID
     */
    var targetNodeID: String = "11111111-1111-1111-1111-111111111111"

    /**
     * コントローラーノード用のアクセストークン
     *
     * intdash APIで取得したアクセストークンを指定して下さい。
     */
    var accessTokenForController : String = ""
    /**
     * 対象ノード用のアクセストークン
     *
     * intdash APIで取得したアクセストークンを指定して下さい。
     */
    var accessTokenForTarget: String = ""

    /**
     * コントローラーノード用のコネクション
     */
    var connectionForController: Connection? = null
    /**
     * 対象ノード用のコネクション
     */
    var connectionForTarget: Connection? = null
}

//region コントローラーノードからメッセージを送信するサンプルです。このサンプルでは文字列メッセージを対象ノードに対して送信し、対象ノードからのリプライを待ちます。

fun E2ECallExampleActivity.connectForController() {
    // 接続情報のセットアップをします。
    var urls = targetServer.split("://")
    var address: String
    var enableTls: Boolean = false
    if (urls.size == 1)
    {
        address = urls[0]
    }
    else
    {
        enableTls = urls[0] == "https"
        address = urls[1]
    }
    // WebSocketを使って接続するように指定します。
    var transportConfig: ITransportConfig = WebSocketConfig(enableTls = enableTls)
    Connection.connectAsync(
        address = address,
        transportConfig = transportConfig,
        tokenSource = {
            // アクセス用のトークンを指定します。接続時に発生するイベントにより使用されます。
            // ここでは固定のトークンを返していますが随時トークンの更新を行う実装にするとトークンの期限切れを考える必要がなくなります。
            accessTokenForController
        },
        nodeId = controllerNodeID,
        completion = { con, ex ->
            if (con == null) {
                // 接続失敗
                return@connectAsync
            }
            // 接続成功.
            connectionForController = con
        }
    )
}

fun E2ECallExampleActivity.sendCall() {
    // コールを送信し、リプライコールを受信するとコールバックが発生します。
    connectionForController?.sendCallAndWaitReplyCallAsync(
        upstreamCall = UpstreamCall(
            destinationNodeId = targetNodeID,
            name = "greeting",
            type = "string",
            payload = "hello".toByteArray()
        ),
        completion = { downstreamReplyCall, ex ->
            if (ex != null) {
                // コールの送信もしくはリプライの受信に失敗。
                return@sendCallAndWaitReplyCallAsync
            }
            // コールの送信及びリプライの受信に成功。
        }
    )
}

//endregion

//region コントローラーノードからのコールを受け付け、すぐにリプライするサンプルです。

fun E2ECallExampleActivity.connectForTarget() {
    // 接続情報のセットアップをします。
    var urls = targetServer.split("://")
    var address: String
    var enableTls: Boolean = false
    if (urls.size == 1)
    {
        address = urls[0]
    }
    else
    {
        enableTls = urls[0] == "https"
        address = urls[1]
    }
    // WebSocketを使って接続するように指定します。
    var transportConfig: ITransportConfig = WebSocketConfig(enableTls = enableTls)
    Connection.connectAsync(
        address = address,
        transportConfig = transportConfig,
        tokenSource = {
            // アクセス用のトークンを指定します。接続時に発生するイベントにより使用されます。
            // ここでは固定のトークンを返していますが随時トークンの更新を行う実装にするとトークンの期限切れを考える必要がなくなります。
            accessTokenForTarget
        },
        nodeId = targetNodeID,
        completion = { con, ex ->
            if (con == null) {
                // 接続失敗
                return@connectAsync
            }
            // 接続成功.
            this.connectionForTarget = con
            // DownstreamCallの受信を監視するためにコールバックを設定します。
            con.e2ECallCallbacks = this.e2ECallCallbacks // ConnectionE2ECallCallbacks
        }
    )
}

//endregion

val E2ECallExampleActivity.e2ECallCallbacks: ConnectionE2ECallCallbacks
    get() = object : ConnectionE2ECallCallbacks {
        override fun onReceiveCall(connection: Connection, downstreamCall: DownstreamCall) {
            // DownstreamCallを受信した際にコールされます。
            // このサンプルではDownstreamCallを受信したらすぐにリプライコールを送信します。
            connection.sendReplyCallAsync(
                upstreamReplyCall = UpstreamReplyCall(
                    requestCallId = downstreamCall.callId,
                    destinationNodeId = downstreamCall.sourceNodeId,
                    name = "reply_greeting",
                    type = "string",
                    payload = "world".toByteArray()
                ),
                completion = { ex ->
                    if (ex != null) {
                        // リプライコールの送信に失敗。
                        return@sendReplyCallAsync
                    }
                    // リプライコールの送信に成功。
                }
            )
        }

        override fun onReceiveReplyCall(connection: Connection, downstreamReplyCall: DownstreamReplyCall) {
            // DownstreamReplyCallを受信した際にコールされます。
        }
    }
```

## References
- [APIリファレンス](https://docs.intdash.jp/api/intdash-sdk/kotlin/latest/)
  - 過去のバージョンのリファレンスは [こちら](https://docs.intdash.jp/api/intdash-sdk/kotlin-versions)
- [GitHub](https://github.com/aptpod/iscp-kt)
