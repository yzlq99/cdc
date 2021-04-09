## kafka connect （Debezium）

0. 使用起来也很简单，消息格式也是 JSON，但是 Update 的时候消息是 Update 前后的快照，也就是要找到变化字段还需要比较，没有 maxwell 方便。
1. 文档非常的全面
2. 支持 MySQL、MongoDB、PostgreSQL、Oracle、SQL Server、Db2、Cassandra 
3. 消息格式（不好分辨那个字段变化了）
    **Insert**
    ```json
    {
        "schema": { ... },
        "payload": { 
            "op": "c",
            "ts_ms": 1465491411815,
            "before": null,
            "after": {
                "id": 1004,
                "first_name": "Anne",
                "last_name": "Kretchmar",
                "email": "annek@noanswer.org"
            },
            "source": {
                "version": "1.2.0.Final",
                "connector": "mysql",
                "name": "mysql-server-1",
                "ts_ms": 0,
                "snapshot": false,
                "db": "inventory",
                "table": "customers",
                "server_id": 0,
                "gtid": null,
                "file": "mysql-bin.000003",
                "pos": 154,
                "row": 0,
                "thread": 7,
                "query": "INSERT INTO customers (first_name, last_name, email) VALUES ('Anne', 'Kretchmar', 'annek@noanswer.org')"
            }
        }
    }
    ```
    **Update**
    ```json
    {
        "schema": { ... },
        "payload": {
            "before": { 
                "id": 1004,
                "first_name": "Anne",
                "last_name": "Kretchmar",
                "email": "annek@noanswer.org"
            },
            "after": { 
                "id": 1004,
                "first_name": "Anne Marie",
                "last_name": "Kretchmar",
                "email": "annek@noanswer.org"
            },
            "source": { 
                "version": "1.2.0.Final",
                "name": "mysql-server-1",
                "connector": "mysql",
                "name": "mysql-server-1",
                "ts_ms": 1465581,
                "snapshot": false,
                "db": "inventory",
                "table": "customers",
                "server_id": 223344,
                "gtid": null,
                "file": "mysql-bin.000003",
                "pos": 484,
                "row": 0,
                "thread": 7,
                "query": "UPDATE customers SET first_name='Anne Marie' WHERE id=1004"
            },
            "op": "u", 
            "ts_ms": 1465581029523 
        }
    }
    ```
    **Delete**
    ```json
    {
        "schema": { ... },
        "payload": {
            "before": { 
                "id": 1004,
                "first_name": "Anne Marie",
                "last_name": "Kretchmar",
                "email": "annek@noanswer.org"
            },
            "after": null, 
            "source": { 
                "version": "1.2.0.Final",
                "connector": "mysql",
                "name": "mysql-server-1",
                "ts_ms": 1465581,
                "snapshot": false,
                "db": "inventory",
                "table": "customers",
                "server_id": 223344,
                "gtid": null,
                "file": "mysql-bin.000003",
                "pos": 805,
                "row": 0,
                "thread": 7,
                "query": "DELETE FROM customers WHERE id=1004"
            },
            "op": "d", 
            "ts_ms": 1465581902461 
        }
    }
    ```

4. Filter 支持 JS filter，也很强大

## maxwell

0. 轻量，使用起来非常简单，消息格式只支持 JSON，Update 时记录 Update 后的状态以及变化字段的 old 值，这种做法决定其很容易找到变化的字段
1. 文档比较简洁
2. 仅支持 MySQL（扩展性没有 kafka connect 强）
3. 消息格式
    **Insert**
    ```json
    mysql> insert into test.e set m = 4.2341, c = now(3), comment = 'I am a creature of light.';
    {
        "database":"test",
        "table":"e",
        "type":"insert",
        "ts":1477053217,
        "xid":23396,
        "commit":true,
        "position":"master.000006:800911",
        "server_id":23042,
        "thread_id":108,
        "primary_key": [1, "2016-10-21 05:33:37.523000"],
        "primary_key_columns": ["id", "c"],
        "data":{
            "id":1,
            "m":4.2341,
            "c":"2016-10-21 05:33:37.523000",
            "comment":"I am a creature of light."
        }
    }
    ```
    **Update**
    ```json
    mysql> update test.e set m = 5.444, c = now(3) where id = 1;
    {
        "database":"test",
        "table":"e",
        "type":"update",
        "ts":1477053234,
        ...
        "data":{
            "id":1,
            "m":5.444,
            "c":"2016-10-21 05:33:54.631000",
            "comment":"I am a creature of light."
        },
        "old":{
            "m":4.2341,
            "c":"2016-10-21 05:33:37.523000"
        }
    }
    ``` 
    **Delete**
    ```json
    mysql> delete from test.e where id = 1;
    {
        "database":"test",
        "table":"e",
        "type":"delete",
        ...
        "data":{
            "id":1,
            "m":5.444,
            "c":"2016-10-21 05:33:54.631000",
            "comment":"I am a creature of light."
        }
    }
    ``` 

4. Filter 比较强大，支持 Javascript Filters