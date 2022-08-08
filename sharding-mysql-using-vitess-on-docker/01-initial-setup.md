## Check out the `vitessio/vitess` repository

1. Clone the `vitessio/vitess` repository and checkout into the `release-14.0` branch:

```bash
$ git clone git@github.com:vitessio/vitess.git
$ cd vitess
$ git checkout release-14.0
```

## Build the docker image

2. Build Vitess Docker image:

```bash
$ make docker_local
```

## Run the docker image

3. Run a Vitess container:

```bash
$ ./docker/local/run.sh
```

​	This will set up a MySQL replication topology, as well as `etcd`, `vtctld`, and `vtgate` services.

  - `vtgate` listens on http://127.0.0.1:15001/debug/status
  - `vtctld` listens on http://127.0.0.1:15000/debug/status
  - The control panel is available at http://localhost:15000/app/


​	There are 3 tablets in the container.

4. Use the following `mysql` commands to connect to various tablets:

- `mysql commerce`
- `mysql commerce@primary`
- `mysql commerce@replica`
- `mysql commerce@rdonly`

5. Use the following queries to know about the tables:

```sql
USE commerce;

DESCRIBE product;
SELECT * FROM product;

DESCRIBE customer;
SELECT * FROM customer;

DESCRIBE corder;
SELECT * FROM corder;
```

​	The tables are in a single unsharded keyspace named `commerce`. Unsharded keyspaces have a single shard named `0` (or `-`).

## Get the list of current tablets

6. Execute `show vitess_tablets` to get the list of current tablets:

```bash
$ mysql --table --execute="show vitess_tablets"
+-------+----------+-------+------------+---------+------------------+-----------+----------------------+
| Cell  | Keyspace | Shard | TabletType | State   | Alias            | Hostname  | PrimaryTermStartTime |
+-------+----------+-------+------------+---------+------------------+-----------+----------------------+
| zone1 | commerce | 0     | PRIMARY    | SERVING | zone1-0000000100 | localhost | 2022-08-08T00:37:21Z |
| zone1 | commerce | 0     | REPLICA    | SERVING | zone1-0000000101 | localhost |                      |
| zone1 | commerce | 0     | RDONLY     | SERVING | zone1-0000000102 | localhost |                      |
+-------+----------+-------+------------+---------+------------------+-----------+----------------------+
```

## Sample data

7. Use the queries in `insert_commerce_data.sql` to insert some sample data:

```bash
$ mysql --table < ../common/insert_commerce_data.sql
```

8. Use the queries in `select_commerce_data.sql` to select the inserted data:

```bash
$ mysql --table < ../common/select_commerce_data.sql
Using commerce
Customer
+-------------+--------------------+
| customer_id | email              |
+-------------+--------------------+
|           1 | alice@domain.com   |
|           2 | bob@domain.com     |
|           3 | charlie@domain.com |
|           4 | dan@domain.com     |
|           5 | eve@domain.com     |
+-------------+--------------------+
Product
+----------+-------------+-------+
| sku      | description | price |
+----------+-------------+-------+
| SKU-1001 | Monitor     |   100 |
| SKU-1002 | Keyboard    |    30 |
+----------+-------------+-------+
COrder
+----------+-------------+----------+-------+
| order_id | customer_id | sku      | price |
+----------+-------------+----------+-------+
|        1 |           1 | SKU-1001 |   100 |
|        2 |           2 | SKU-1002 |    30 |
|        3 |           3 | SKU-1002 |    30 |
|        4 |           4 | SKU-1002 |    30 |
|        5 |           5 | SKU-1002 |    30 |
+----------+-------------+----------+-------+
```

Exiting the docker shell terminates and destroys the Vitess cluster. DO NOT exit the docker shell to proceed to the next sections.