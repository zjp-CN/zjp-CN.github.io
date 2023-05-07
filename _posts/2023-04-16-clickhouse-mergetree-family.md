```sql
-- without ver - the last inserted 'wins'
CREATE TABLE IF NOT EXISTS myFirstReplacingMT
(
    `key` Int64,
    `someCol` String,
    `eventTime` DateTime
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (key);

INSERT INTO myFirstReplacingMT Values (1, 'first', '2020-01-01 01:01:01');
INSERT INTO myFirstReplacingMT Values (1, 'second', '2020-01-01 00:00:00');

SELECT tuple(someCol) FROM myFirstReplacingMT FINAL;

-- 定义一个除了表引擎不同，其他部分完全与 myFirstReplacingMT 相同的表
CREATE TABLE IF NOT EXISTS mySecondReplacingMT
(
    `key` Int64,
    `someCol` String,
    `eventTime` DateTime
)
ENGINE = MergeTree
PRIMARY KEY (key);

-- 复制
ALTER TABLE mySecondReplacingMT ATTACH PARTITION tuple() FROM myFirstReplacingMT;
SELECT * FROM mySecondReplacingMT;

DROP TABLE myFirstReplacingMT SYNC;
-- DROP TABLE mySecondReplacingMT SYNC;
```
