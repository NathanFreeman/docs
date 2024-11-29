以下是文本中非代码块部分翻译成日语的内容：

**
* 安装
  * [Swooleのインストール](environment.md)
  * [拡張の衝突](getting_started/extension.md)

* [簡単な例](start/start_server.md)
  * [TCPサーバー](start/start_tcp_server.md)
  * [UDPサーバー](start/start_udp_server.md)
  * [HTTPサーバー](start/start_http_server.md)
  * [WebSocketサーバー](start/start_ws_server.md)
  * [MQTT（IoT）サーバー](start/start_mqtt.md)
  * [非同期タスクの実行](start/start_task.md)
  * [キューアブレーションの初探](start/coroutine.md)

* [サーバー（非同期スタイル）](server/init.md)
  * [TCP/UDPサーバー](server/tcp_init.md)
    * [方法](server/methods.md)
    * [プロパティ](server/properties.md)
    * [設定](server/setting.md)
    * [コールバックイベント](server/events.md)
  * [HTTPサーバー](http_server.md)
  * [WebSocketサーバー](websocket_server.md)
  * [Redisサーバー](redis_server.md)
  * [複数のポートの聞き取り](server/port.md)

* [サーバー（キューアブレーションスタイル）](server/co_init.md)
  * [TCPサーバー](coroutine/server.md)
  * [HTTPサーバー](coroutine/http_server.md)
  * [WebSocketサーバー](coroutine/ws_server.md)

* [クライアント](client_init.md)
  * [同期ブロッキングクライアント](client.md)
  * [非同期コールバッククライアント](client_async.md)
  * [キューアブレーションクライアント](coroutine_client/init.md)
    * [TCP/UDPクライアント](coroutine_client/client.md)
    * [Socketクライアント](coroutine_client/socket.md)
    * [HTTP/WebSocketクライアント](coroutine_client/http_client.md)
    * [HTTP2クライアント](coroutine_client/http2_client.md)
    * [PostgreSQLクライアント](coroutine_client/postgresql.md)
    * [FastCGIクライアント](coroutine_client/fastcgi.md)
    * [MySQLクライアント](coroutine_client/mysql.md)
    * [Redisクライアント](coroutine_client/redis.md)

* [キューアブレーション（Coroutine）](coroutine.md)
  * [ワンクリックキューアブレーション](runtime.md)
  * [核心API](coroutine/coroutine.md)
  * [キューアブレーションコンテナ](coroutine/scheduler.md)
  * [システムAPI](coroutine/system.md)
  * [プロセスAPI](coroutine/proc_open.md)
  * [Channel](coroutine/channel.md)
  * [WaitGroup](coroutine/wait_group.md)
  * [Barrier](coroutine/barrier.md)
  * [並行呼び出し](coroutine/multi_call.md)
  * [接続プール](coroutine/conn_pool.md)
  * [Library](library.md)
  * [キューアブレーションのデバッグ](coroutine/gdb.md)
  * [プログラミング注意](coroutine/notice.md)

* ファイルの非同期操作
  * [実装](file/engine.md)
  * [設定](file/setting.md)

* チェックリスト管理 (Thread)
  * [チェックリストの作成](thread/thread.md)
  * [チェックリストプール](thread/pool.md)
  * [方法とプロパティ](thread/info)
  * [並行マップ](thread/map.md)
  * [並行リスト](thread/arraylist.md)
  * [並行队列](thread/queue.md)
  * [同期バリア](thread/barrier.md)
  * [データタイプ](thread/transfer.md)

* プロセス管理 (Process)
  * [プロセスの作成](process/process.md)
  * [プロセスプール](process/process_pool.md)
  * [プロセスマネージャー](process/process_manager.md)
  * [高パフォーマンスの共有メモリ](memory/table.md)

* 并行管理
  * [ロック](memory/lock.md)
  * [原子計数](memory/atomic.md)
  * [イベントループ](event.md)
  * [タイマー](timer.md)

* その他
  * [常量](consts.md)
  * [エラーコード](other/errno.md)
  * [ini設定](other/config.md)
  * [雑項関数](functions.md)
  * [ツールの使用](other/tools.md)
  * [関数の别名まとめ](other/alias.md)
  * [エラーレポートの提出](other/issue.md)
  * [カーネルパラメータの調整](other/sysctl.md)
  * [Linuxシグナルリスト](other/signal.md)
  * [オンライン交流](other/discussion.md)
  * [ドキュメント貢献者](CONTRIBUTING.md)
  * [Swooleプロジェクトへの寄付](other/donate.md)
  * [ユーザーとケーススタディ](case.md)

* 常見問題
  * [インストール問題](question/install.md)
  * [使用問題](question/use.md)
  * [Swooleについて](question/swoole.md)

* バージョン管理
  * [サポート計画](version/supported.md)
  * [向下不兼容の変更](version/bc.md)
  * [バージョン更新履歴](version/log.md)

* Swoole学習
  * [基礎知識](learn.md)
  * [プログラミング注意](getting_started/notice.md)
  * [その他の知識](learn_other.md)
  * [Swoole記事](blog_list.md)
**