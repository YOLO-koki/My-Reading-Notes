# Nginxまわり

- nginxのアクセスログはベンチやabコマンド回すごとに出力先ファイルを切り替え、一回のアクセスログ集計・分析につき、1ファイルを利用する形式にする
  - abコマンドやベンチでアプリ実行
  - これにより、Webサーバー（nginx）のアクセスログが溜まる
  - そのアクセスログを`alp`コマンドで集計してデータを分析し、ボトルネックを特定する
  - ログが出力されたファイルを退避させる（ex. mv logs/access.log logs/access.log.old）
  - もう一度abコマンドやベンチでアプリ実行

- ログファイルのローテーションを行うシェルスクリプトの例

```shell
#!/bin/sh

mv /var/log/nginx/access.log /var/nginx/access.log. `date +%Y%m%d-%H%M%S`
nginx -s reopen
```

## リバースプロキシについて

- C10K問題
  - 大量のリクエストが飛んできたとき、プロセス同士またはスレッド同士でのコンテキストスイッチが必要になってしまうことによってメモリをが不足し、CPUが忙しくなって結果的にパフォーマスが落ちてしまう問題
- C10K問題を回避する例
  - goroutineを使う
  - Nginxの、ノンブロッキングI/Oと多重I/O
    - ノンブロッキングI/O
      - ブロッキング…データの読み込みや到着、書き込み等が完了するまでのプログラムの待ち時間のこと
      - ノンブロッキングI/Oはこの待ち時間の発生を回避できる代わりに、エラーが発生する
        - そのため、これを活用する際は適切なエラー処理（リトライなど）が必要
    - 多重I/O
      - ファイルディスクリプタ…OSやプログラムが、書き込みや読み込みを行うための対象ファイルのパイプ（I/O）を識別するための整数
      - 複数のファイルディスクリプタを同時に渡すことができ、それによりいずれかのファイルディスクリプタがI/O可能になった時に通知を受け取ることができる
- Nginxをリバースプロキシとして利用する時のポイント
  - 静的ファイルの設定
    - 静的ファイルの配信をアプリケーションサーバーが行っている場合、Webサーバー側（Nginx等）に設定を入れることでアプリケーションサーバーへの負荷を軽減できる
  - gzipを使って圧縮したレスポンスを返すこと

# MySQLまわり

- スロークエリログ出力設定(以下、例)

```
# どこかにあるmy.cnf（なければ作ればOK）

[mysql]
slow_query_log = 1                                  # スロークエリを有効化
slow_query_log_file = /var/log/mysql/mysql-slow.log # スロークエリログの出力先ファイル名
long_query_time = 0                                 # 指定した秒数以上かかったクエリのみログ出力

```

- `long_query_time` について補足
  - `0`にすると全てのクエリのログが出力されるが、そのメリデメ
    - メリット
      - 1回の実行が高速でも、（無意味に）大量に発行されているクエリの特定に有効
    - デメリット
      - 全てのクエリが出力されるため、出力されたログから特に時間がかかっているクエリの特定が困難
- 設定の反映にはインスタンス内でのMySQL（ローカルならdockerコンテナ自体）の再起動が必要

- ログ出力内容の確認ポイント
  - mysqldumpslowコマンドで、ログ内容を確認する場合（ex. `mysqldumpslow /var/log/mysql/mysql-slow.log`）
    - なんのクエリに、どれぐらい時間がかかっているのかの確認
      - 例えば、一回の実行に0.05sかかるクエリが30秒間で480回呼び出されると、22秒を消費してしまう
    - 時間がかかっているクエリについては生ログを確認すること
      - インスタンスやホストマシン上にマウントされたログファイル（mysql-slow.log）の内容を`mysqldumpslow`等のログ集計ツール経由ではなく、`cat`コマンドで生ログを出力し、`Rows_sent`と`Rows_examined`の比率を確認するのが良い
        - 例えば、`Rows_sent`が3で、`Rows_examined`が100000なら、内部で10万行を処理し、クライアントには3行返却した、という意味（←効率が悪いことがわかる）
          - 原因はだいたいインデックスがPKにしか貼られていないこと
- インデックスの有無を確認する時は`explain`を使ったクエリをMySQL上で実行する（以下、例）

```sql
mysql> set exlain_format = TRADITIONAL;
mysql> EXPLAIN SELECT * FROM `comments` WHERE `post_id` = 9995 ORDER BY `created_at` DESC LIMIT 10;
+----+-------------+----------+------------+------+---------------+------+---------+------+-------+----------+-----------------------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                       |
+----+-------------+----------+------------+------+---------------+------+---------+------+-------+----------+-----------------------------+
|  1 | SIMPLE      | comments | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 99288 |    10.00 | Using where; Using filesort |
+----+-------------+----------+------------+------+---------------+------+---------+------+-------+----------+-----------------------------+
1 row in set, 1 warning (0.004 sec)

mysql> alter table comments add index post_id_idx(post_id);
Query OK, 0 rows affected (0.110 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> EXPLAIN SELECT * FROM `comments` WHERE `post_id` = 9995 ORDER BY `created_at` DESC LIMIT 10;
+----+-------------+----------+------------+------+---------------+-------------+---------+-------+------+----------+----------------+
| id | select_type | table    | partitions | type | possible_keys | key         | key_len | ref   | rows | filtered | Extra          |
+----+-------------+----------+------------+------+---------------+-------------+---------+-------+------+----------+----------------+
|  1 | SIMPLE      | comments | NULL       | ref  | post_id_idx   | post_id_idx | 4       | const |    6 |   100.00 | Using filesort |
+----+-------------+----------+------------+------+---------------+-------------+---------+-------+------+----------+----------------+
1 row in set, 1 warning (0.001 sec)
```

- 負荷試験を再実行（ベンチまたはabコマンド、アプリを動かすとき）時には、予めスロークエリログを削除しておき、前回の実行時のログと混ざらないようにすること
  - ログファイル（mysql-slow.log）を削除したり名前を変えた時は`mysqladmin flush-logs`を実行してファイルが更新されていることをMySQLに伝える必要あり

一連のコマンド例↓

```shell
rm /var/log/mysql/mysql-slow.log
mysqladmin flush-   logs # 「mysqladmin -uroot -proot flush-logs」というコマンドの必要である可能性もある
```

## N+1問題の解決法

1. キャッシュ活用
   - Redisやmemcached等のミドルウェアを用いて、同じIDでのクエリリクエストであればキャッシュから取り出し、そうでなければDBにリクエストするよう調整する方法
2. SQLの(INNNER)JOIN
   - JOINを使って、2つ以上のテーブル情報を1つのクエリで取得する方法
3. SQLのIN句
   - 予めIDをリストで整形しておき、それらをIN句に渡すことで、合計2回のクエリに抑える方法
     - 注意
       - IN句の長さが大きすぎると、クエリのサイズが大きくなりすぎてエラーになったり、狙ったインデックスが使われなくなることもある

## その他の効率化

1. 必要なカラムだけ取得する
   - `SELECT *`と書かず、必要なカラムだけ取得する
2. プリペアドステートメントを無効にする
   - Go言語のドライバである`go-sql-driver/mysql`ではデフォルトでONになっているので、ここをOFFにすることで無意味なキャッシュ処理を無くし通信の回数を減らせることがある
3. キャッシュの活用
   - MySQLには、読み込んだデータやインデックスをメモリ上に確保する機能（InnoDB Buffer Pool）があ理、データベース専用にメモリを確保してくれる
     - 設定値は`innodb_buffer_pool_size`という値で、MySQL8.0では128MBがデフォルトになってる。
     - この値を調整すれば、必要に応じてストレージではなくメモリからデータを読み込んでくれるので、高速化が見込める
       - データベース専用のサーバーがある場合は、物理メモリの80%を割り当てると良い
       - 注意：OS自体にもディスクキャッシュという一度ファイルから呼び出したデータをメモリにキャッシュとして確保する機構があるので、これと重複しないよう調整する必要がある
         - 対処法：データを読み書きする際に、`O_DIRECT`というフラグを有効にする
           - MySQLでは、`innodb_flush_method=O_DIRECT`という形で、設定ファイル`my.cnf`のserverブロックに追記し再起動すれば良い

# abコマンド

- 注目ポイント
  - `Requests per second`
    - 大きい数字であればあるほど良い（高速と言える）
  - `Time per request`
    - 小さければ小さいほど良い
    - もし並列（`ab`コマンドの`c`オプションでの数値）を増やせば増やすほど、この値が大きくなるのであればCPU利用率が悪い可能性が高い
      - 本来マシンに備わっているCPUを有効に使えてないとか

出力内容例

```shell
$ ab -c 1 -n 10 http://localhost/
This is ApacheBench, Version 2.3 <$Revision: 1923142 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient).....done


Server Software:        nginx/1.30.5
Server Hostname:        localhost
Server Port:            80

Document Path:          /
Document Length:        33990 bytes

Concurrency Level:      1
Time taken for tests:   3.319 seconds
Complete requests:      10
Failed requests:        0
Total transferred:      341270 bytes
HTML transferred:       339900 bytes
Requests per second:    3.01 [#/sec] (mean)
Time per request:       331.879 [ms] (mean)
Time per request:       331.879 [ms] (mean, across all concurrent requests)
Transfer rate:          100.42 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.0      0       0
Processing:   323  332  11.5    329     364
Waiting:      323  332  11.4    329     363
Total:        323  332  11.5    329     364

Percentage of the requests served within a certain time (ms)
  50%    329
  66%    330
  75%    330
  80%    331
  90%    364
  95%    364
  98%    364
  99%    364
 100%    364 (longest request)

```
