## Apply new sharded VSchema

1. Use the following commands to create sequence tables and remove `auto_increment` attribute of the ID columns:

   ```bash
   $ vtctlclient ApplySchema -- --sql-file create_commerce_seq.sql commerce
   $ vtctlclient ApplyVSchema -- --vschema_file vschema_commerce_seq.json commerce
   $ vtctlclient ApplyVSchema -- --vschema_file vschema_customer_sharded.json customer
   $ vtctlclient ApplySchema -- --sql-file create_customer_sharded.sql customer
   ```

## Create new shards

2. Use the following commands to add new shards as well as two primary tablets:

   ```bash
   for i in 300 301 302; do
    CELL=zone1 TABLET_UID=$i ./scripts/mysqlctl-up.sh
    SHARD=-80 CELL=zone1 KEYSPACE=customer TABLET_UID=$i ./scripts/vttablet-up.sh
   done
   
   for i in 400 401 402; do
    CELL=zone1 TABLET_UID=$i ./scripts/mysqlctl-up.sh
    SHARD=80- CELL=zone1 KEYSPACE=customer TABLET_UID=$i ./scripts/vttablet-up.sh
   done
   
   $ vtctlclient InitShardPrimary -- --force customer/-80 zone1-300
   $ vtctlclient InitShardPrimary -- --force customer/80- zone1-400
   ```

## Reshard the keyspace

3. Use `Reshard` to reshard the keyspace `customer/0` into shards `customer/-80` and `customer/80-`:

   ```bash
   $ vtctlclient Reshard -- --source_shards '0' --target_shards '-80,80-' Create customer.cust2cust
   ```

4. Validate the correctness of the operation:

   ```bash
   $ vtctlclient VDiff customer.cust2cust
   Summary for customer: {ProcessedRows:5 MatchingRows:5 MismatchedRows:0 ExtraRowsSource:0 ExtraRowsTarget:0}
   Summary for corder: {ProcessedRows:5 MatchingRows:5 MismatchedRows:0 ExtraRowsSource:0 ExtraRowsTarget:0}
   ```

5. Switch read operations:

   ```bash
   $ vtctlclient Reshard -- --tablet_types=rdonly,replica SwitchTraffic customer.cust2cust
   ```

6. Switch write operations:

   ```bash
   $ vtctlclient Reshard -- --tablet_types=primary SwitchTraffic customer.cust2cust
   ```

## Finalize the resharding

7. Delete the old data and stop the old MySQL instances of shard `customer/0`:

   ```bash
   $ vtctlclient Reshard Complete customer.cust2cust
   
   for i in 200 201 202; do
    CELL=zone1 TABLET_UID=$i ./scripts/vttablet-down.sh
    CELL=zone1 TABLET_UID=$i ./scripts/mysqlctl-down.sh
   done
   ```

## Select data from the sharded keyspace

8. Use the following queries to select data from the shards:

   ```bash
   $ mysql --table < ../common/select_customer-80_data.sql
   Using customer/-80
   Customer
   +-------------+--------------------+
   | customer_id | email              |
   +-------------+--------------------+
   |           1 | alice@domain.com   |
   |           2 | bob@domain.com     |
   |           3 | charlie@domain.com |
   |           5 | eve@domain.com     |
   +-------------+--------------------+
   COrder
   +----------+-------------+----------+-------+
   | order_id | customer_id | sku      | price |
   +----------+-------------+----------+-------+
   |        1 |           1 | SKU-1001 |   100 |
   |        2 |           2 | SKU-1002 |    30 |
   |        3 |           3 | SKU-1002 |    30 |
   |        5 |           5 | SKU-1002 |    30 |
   +----------+-------------+----------+-------+
   
   $ mysql --table < ../common/select_customer80-_data.sql
   Using customer/80-
   Customer
   +-------------+----------------+
   | customer_id | email          |
   +-------------+----------------+
   |           4 | dan@domain.com |
   +-------------+----------------+
   COrder
   +----------+-------------+----------+-------+
   | order_id | customer_id | sku      | price |
   +----------+-------------+----------+-------+
   |        4 |           4 | SKU-1002 |    30 |
   +----------+-------------+----------+-------+
   ```

​		Now, if you insert new data in the `customer` or `corder` table, some of them will be inserted into the `customer/-80` shard and some into the `customer/80-` shard.