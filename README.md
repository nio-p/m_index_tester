# m_index_hands_on

## setup

```
# local
$ docker-compose up -d
$ docker container ls
$ docker exec -it CONTAINER_ID /bin/bash

# container
$ mysql -uroot -pHoge1234_Hoge1234

# mysql
$ use test;
```

## test

```
# まずはテーブル構成をチェックしましょう。
$ desc users;
+-------------+--------------+------+-----+-------------------+-----------------------------------------------+
| Field       | Type         | Null | Key | Default           | Extra                                         |
+-------------+--------------+------+-----+-------------------+-----------------------------------------------+
| id          | int          | NO   | PRI | NULL              | auto_increment                                |
| status      | int unsigned | NO   |     | NULL              |                                               |
| name        | varchar(50)  | NO   |     | NULL              |                                               |
| postal_code | varchar(20)  | NO   |     | NULL              |                                               |
| prefecture  | int unsigned | NO   |     | NULL              |                                               |
| created_at  | timestamp    | NO   |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED                             |
| updated_at  | timestamp    | NO   |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED on update CURRENT_TIMESTAMP |
+-------------+--------------+------+-----+-------------------+-----------------------------------------------+

# 次に、usersテーブルに何件データが入っているか確認しましょう。
# 約10万件入っています。
$ select count(*) from users;
+----------+
| count(*) |
+----------+
|   400105 |
+----------+
1 row in set (0.03 sec)

# prefecture（都道府県コード）は1 ~ 47までなので、まずは東京都のユーザーを探してみましょう。
$ select * from users where prefecture = 13;

# 8544件がヒットし、取得に0.24secかかりました。
> 8544 rows in set (0.24 sec)

# 次に、実行計画を見てみましょう。
$ explain select * from users where prefecture = 13;

# typeを見てみると、ALLと表示されています。所謂全件検索です。
# rowsは行数の見積もりです。今回の場合はcountで表示された件数とほぼ一緒です。
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | users | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 380782 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+

# 今後データが増えると検索時間が更にかかります。あまりよろしくないので改善してみましょう。
# prefectureカラムにindexを追加してみましょう。
$ alter table users add index (prefecture);

# 成功したら、再度検索してみましょう。
$ select * from users where prefecture = 13;
> 8544 rows in set (0.05 sec)

# 取得時間が短縮されました。
# 次に、実行計画を再度見てみましょう。
# typeがrefになり、rowsが取得数と同じ8544になりました。これなら合格です。
$ explain select * from users where prefecture = 13;
+----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | users | NULL       | ref  | prefecture    | prefecture | 4       | const | 8544 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+-------+

# 次に、statusとprefectureの2つを使って絞り込んでみましょう。
# statusは1 ~ 10で設定されていますので、今回は東京都のstatusが4のユーザーを取得し、実行計画を見てみましょう。
$ select * from users where prefecture = 13 and status = 4;
> 891 rows in set (0.03 sec)
+----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | users | NULL       | ref  | prefecture    | prefecture | 4       | const | 8544 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+-------------+

# 結果だけ見るとあまり悪くありません。
# が、rowsはprefecture絞り込みのままから変わらず、filteredは10です。
# ここから更にrowsを891、filteredを100にしてみましょう。
# さっそくstatusにindexを貼ってみましょう。
$ alter table users add index (status);

# 次に実行計画を見てみましょう。
$ explain select * from users where prefecture = 13 and status = 4;
+----+-------------+-------+------------+-------------+-------------------+-------------------+---------+------+------+----------+-------------------------------------------------+
| id | select_type | table | partitions | type        | possible_keys     | key               | key_len | ref  | rows | filtered | Extra                                           |
+----+-------------+-------+------------+-------------+-------------------+-------------------+---------+------+------+----------+-------------------------------------------------+
|  1 | SIMPLE      | users | NULL       | index_merge | prefecture,status | prefecture,status | 4,4     | NULL | 1715 |   100.00 | Using intersect(prefecture,status); Using where |
+----+-------------+-------+------------+-------------+-------------------+-------------------+---------+------+------+----------+-------------------------------------------------+

# typeがindex_mergeとなりました。
# 実はあまりよろしくありません。以下の公式ドキュメントを見てみましょう。index_mergeはrefよりも遅いです。
# https://dev.mysql.com/doc/refman/5.6/ja/explain-output.html#explain-join-types
# また、rowsも減りはしたものの、正しい数値ではありません。
# これを改善するために、複合インデックスを設定してみましょう。
$ alter table users add index (prefecture, status);

# 実行計画を見る前に、indexの状況を見てみましょう。
$ show index from users;
+-------+------------+--------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table | Non_unique | Key_name     | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-------+------------+--------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| users |          0 | PRIMARY      |            1 | id          | A         |      380782 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| users |          1 | prefecture   |            1 | prefecture  | A         |          45 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| users |          1 | status       |            1 | status      | A         |           9 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| users |          1 | prefecture_2 |            1 | prefecture  | A         |          45 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| users |          1 | prefecture_2 |            2 | status      | A         |         496 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+-------+------------+--------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+

# どのテーブルにどういったindexが貼られているか確認したいときに使いましょう。
# では、実行計画を見てみましょう。
$ explain select * from users where prefecture = 13 and status = 4;
+----+-------------+-------+------------+------+--------------------------------+--------------+---------+-------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys                  | key          | key_len | ref         | rows | filtered | Extra |
+----+-------------+-------+------------+------+--------------------------------+--------------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | users | NULL       | ref  | prefecture,status,prefecture_2 | prefecture_2 | 8       | const,const |  891 |   100.00 | NULL  |
+----+-------------+-------+------------+------+--------------------------------+--------------+---------+-------------+------+----------+-------+

# typeはrefになり、rowsも取得分と同じ。良さそうです。
# 実際にクエリを打ってみます。
$ select * from users where prefecture = 13 and status = 4;
> 891 rows in set (0.00 sec)
```
