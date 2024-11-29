#サポート計画

| ブランチ | PHPバージョン | 開始日時 |アクティブサポート終了日時 | セキュリティメンテナンス終了日時 |
|-----------------------------------------------------------------|-----------|------------|------------|------------|
| [v4.8.x](https://github.com/swoole/swoole-src/tree/4.8.x)  | 7.2 - 8.2 | 2021-10-14 | 2023-10-14 | 2024-06-30 |
| [v5.0.x](https://github.com/swoole/swoole-src/tree/5.0.x)       | 8.0 - 8.2 | 2022-01-20 | 2023-01-20 | 2023-07-20 |
| [v5.1.x](https://github.com/swoole/swoole-src/tree/master)      | 8.0 - 8.2 | 2023-09-30 | 2025-09-30 | 2026-09-30 |
| [v6.0.x](https://github.com/swoole/swoole-src/tree/master)      | 8.1 - 8.3 | 2024-06-23 | 2026-06-23 | 2027-06-23 |

| アクティブサポート | 官方開発チームからの積極的なサポートがあり、報告されたエラーやセキュリティ問題は直ちに修正され、通常のプロセスに従って正式なバージョンがリリースされます。 |
| -------- | ---------------------------------------------------------------------------------------------- |
| セキュリティメンテナンス | 緊急のセキュリティ問題の修正のみをサポートし、必要に応じて正式なバージョンがリリースされます |
##もうサポートされていないバージョン

これらのバージョンはもはや公式にサポートされていません。これらのバージョンを使用しているユーザーはできるだけ早くアップグレードする必要があります。なぜなら、未解決のセキュリティ脆弱性に遭遇する可能性があるからです。- `v1.x` (2012-7-1 ~ 2018-05-14)- `v2.x` (2016-12-30 ~ 2018-05-23)- `v3.x` （廃止）- `v4.0.x`, `v4.1.x`, `v4.2.x`, `v4.3.x` (2018-06-14 ~ 2019-12-31)- `v4.4.x`（2019-04-15 ~ 2020-04-30）- `v4.5.x` (2019-12-20 ~ 2021-01-06)
- `v4.6.x`, `v4.7.x`（2021-01-06 ~ 2021-12-31）
##バージョン特徴- `v1.x`：非同期回调モード。- `v2.x`：`setjmp/longjmp`に基づく単スタックのコロナールで、基盤は依然として非同期回调を実現し、回调イベントがトリガーされた後にPHP呼び出しスタックを切り替えます。- `v4.0-v4.3`：`boost context asm`に基づく二重スタックのコロナールで、カーネルは全面的に協程化され、`EventLoop`に基づく協程スケジュール器を実現しました。- `v4.4-v4.8`：`runtime coroutine hook`を実現し、PHPの組み込み同期ブロッキング関数を自動的に協程の非同期非ブロッキングモードに置き換え、Swoole协程がほとんどのPHPライブラリと互換性を持つようにしました。- `v5.0`：全面的に協程化され、非協程モジュールが削除されました。型安全化され、多くの歴史的な負荷が取り除かれました。新しい`swoole-cli`実行モードが提供されています。- `v5.1`：`pdo_pgsql`、`pdo_oci`、`pdo_odbc`、`pdo_sqlite`の協程化をサポートし、Http\Serverのパフォーマンスを向上させました。
- `v6.0`：マルチスレッドモードをサポートします。