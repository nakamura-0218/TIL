# TIL
ECS要点整理
運用設計
⓵ログ設計
■ サービス概要
1.CloudWatch Logs
・シンプルな運用や大量にログが出力されないケースにおいてはCloudWatch Logsのみでログ収集が可能
・保持期間を設定できるため、必要に応じて蓄積されたログのメンテナンス処理を自動化
・CloudWatchサブスクリプションフィルターを利用することで、指定した文字列が含まれているログのみを抽出可能
⇒抽出したログをLambda経由でSNSやQ Developer in chat applicationsと連携しSlackなどに通知

2.FireLens(Fluent Bit)　サイドカーコンテナ構成
・Data Firehoseと連携してAWSサービス(S3、Redshift、Opensearch Service)や、サードパーティ(Splunk、Snowflake)へのログ転送が可能
・S3とCloudWatch Logs同時に転送可能

■ ログ管理手法
〇ログ長期保管の目的としてアプリケーションログをS3へ、障害検知用途としてエラーログはCloudWatch Logsへ転送する場合
1.CloudWatch Logsに転送後、S3へエクスポートする
⇒CloudWatch Logsに全てのログが転送されることによるコスト増大、S3へのエクスポート完了までに最大12時間かかる

2.CLoudWatch Logsのサブスクリプションフィルター機能を利用し、Data Firehoseを経由させS3へ継続的にログを保存
⇒1.と同様に、CloudWathc Logsにログが全て取り込まれることと、Data Firehoseの料金も加算されるため最もコストがかかってしまう

3.Fluent Bitを利用し、CloudWatch LogsとS3へ同時出力させる
⇒カスタム定義を利用することで障害検知に必要なログはCloudWatch Logsへ転送し、それ以外のログはS3へというように柔軟にログ管理が可能
　特定のログをフィルタリングして指定した出力先のみにログを出力することができる

■ 設計判断基準
・ログが大量に出力されないシンプルな運用
⇒CloudWatch Logsを選定
・ログが大量に出力される場合や、コストを最適化したい場合
⇒ログ出力先を柔軟に指定できるFireLens(Fluent Bit)を選定


⓶メトリクス設計
■ メトリクス取得手法
1.標準CloudWatchメトリクス
取得粒度：ECSクラスター、ECSサービス
主なメトリクス：
- CPUUtilization
- MemoryUtilization
特徴：追加コスト無し、最小限の監視

2.Container Insights
取得粒度：ECSクラスター、ECSサービス、ECSタスク
特徴：
- タスク単位でモニタリングが可能（コンテナ単位ではなく平均値）
- ディスクやネットワークに関するメトリクスも収集可能
- コンテナ数増加に伴うコスト影響が小さい

3.Container Insights enhanced observability
取得粒度：ECSクラスター、ECSサービス、ECSタスク、コンテナ
特徴：
- コンテナ単位で詳細なリソース・パフォーマンス分析が可能
- メトリクス数増加に比例してコスト増加

■ 設計判断基準
・最低限のリソース監視で十分な場合(主に開発環境)
⇒標準CloudWatchメトリクスを選定
・タスク単位での状態把握や異常検知が必要な場合(主に検証環境、本番環境)
⇒Container Insightsを選定
・コンテナ単位での詳細分析やトラブルシュートが必要な場合(主に検証環境、本番環境)
⇒Container Insights enhanced observabilityを選定

※注意事項
サイドカーコンテナが複数必要な場合、スケールアウトによりECSタスク数がスケールすると、アプリケーションコンテナだけではなく、サイドカーコンテナもそれに応じて増加する。
ECSタスクが増えるほど、消費するリソースも増え、Container Insights enhanced observabilityを有効化している場合、サイドカーコンテナ分のカスタムメトリクス取得が発生する。
スケールアウトが多数発生する場合、サイドカーコンテナをECSサービスとして切り離すことで、必要なリソース量に制限してスケールさせることが可能。


