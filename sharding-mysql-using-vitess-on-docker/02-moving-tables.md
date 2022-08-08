## Create new tablets

The `customer` and `corder` tables are closely related and both growing quickly. So, it's better to have them in the same keyspace. We will create a `customer` keyspace and move the tables `customer` and `corder` to it.

1.  Create 3 new tablets with IDs `200` to `202` in a new keyspace named `customer`:

   ```bash
   for i in 200 201 202; do
    CELL=zone1 TABLET_UID=$i ./scripts/mysqlctl-up.sh
    CELL=zone1 KEYSPACE=customer TABLET_UID=$i ./scripts/vttablet-up.sh
   done
   
   $ vtctlclient InitShardPrimary -- --force customer/0 zone1-200
   $ vtctlclient ReloadSchemaKeyspace customer
   ```

## Check new tablets

2. Get the list of current tablets:

   ```bash
   $ mysql --table --execute="show vitess_tablets"
   +-------+----------+-------+------------+---------+------------------+-----------+----------------------+
   | Cell  | Keyspace | Shard | TabletType | State   | Alias            | Hostname  | PrimaryTermStartTime |
   +-------+----------+-------+------------+---------+------------------+-----------+----------------------+
   | zone1 | commerce | 0     | PRIMARY    | SERVING | zone1-0000000100 | localhost | 2022-08-08T00:37:21Z |
   | zone1 | commerce | 0     | REPLICA    | SERVING | zone1-0000000101 | localhost |                      |
   | zone1 | commerce | 0     | RDONLY     | SERVING | zone1-0000000102 | localhost |                      |
   | zone1 | customer | 0     | PRIMARY    | SERVING | zone1-0000000200 | localhost | 2022-08-08T00:52:39Z |
   | zone1 | customer | 0     | REPLICA    | SERVING | zone1-0000000201 | localhost |                      |
   | zone1 | customer | 0     | RDONLY     | SERVING | zone1-0000000202 | localhost |                      |
   +-------+----------+-------+------------+---------+------------------+-----------+----------------------+
   ```

## Move the tables

3. Use `MoveTables` workflow to copy tables `customer` and `corder` from the keyspace `commerce` to `customer` without downtime:

   ```bash
   $ vtctlclient MoveTables -- --source commerce --tables 'customer,corder' Create customer.commerce2customer
   ```

​		In the command above, `Create` is an action for `MoveTables` workflow, and `customer.commerce2customer` is the workflow identifier.

​		Any changes to the tables after the `MoveTables` is executed will be copied faithfully to the new copy of these tables in the `customer` keyspace.

## Check routing rules

4. Use `GetRoutingRules` to get routing rules of the `commerce` keyspace:

   ```bash
   $ vtctlclient GetRoutingRules commerce
   {
     "rules": [
       {
         "fromTable": "customer",
         "toTables": [
           "commerce.customer"
         ]
       },
       {
         "fromTable": "customer.customer",
         "toTables": [
           "commerce.customer"
         ]
       },
       {
         "fromTable": "corder",
         "toTables": [
           "commerce.corder"
         ]
       },
       {
         "fromTable": "customer.corder",
         "toTables": [
           "commerce.corder"
         ]
       }
     ]
   }
   ```

5. Use `VReplicationExec` to monitor the process of copying tables:

   ```bash
   $ vtctlclient VReplicationExec zone1-0000000200 "select * from _vt.copy_state"
   +----------+------------+--------+
   | vrepl_id | table_name | lastpk |
   +----------+------------+--------+
   +----------+------------+--------+
   ```

6. Validate the correctness of the operation:

   ```bash
   $ vtctlclient VDiff customer.commerce2customer
   Summary for corder: {ProcessedRows:5 MatchingRows:5 MismatchedRows:0 ExtraRowsSource:0 ExtraRowsTarget:0}
   Summary for customer: {ProcessedRows:5 MatchingRows:5 MismatchedRows:0 ExtraRowsSource:0 ExtraRowsTarget:0}
   ```

## Switch `SELECT` statements to read from the new keyspace

7. Use `SwitchTraffic` action of `MoveTables` workflow to switch `SELECT` statements to read from the keyspace `customer`:

   ```bash
   $ vtctlclient MoveTables -- --tablet_types=rdonly,replica SwitchTraffic customer.commerce2customer
   ```

8. Check the routing rules of `commerce` keyspace again:

   ```bash
   $ vtctlclient GetRoutingRules commerce
   {
     "rules": [
       {
         "fromTable": "commerce.corder@rdonly",
         "toTables": [
           "customer.corder"
         ]
       },
       {
         "fromTable": "commerce.corder@replica",
         "toTables": [
           "customer.corder"
         ]
       },
       {
         "fromTable": "customer.customer@rdonly",
         "toTables": [
           "customer.customer"
         ]
       },
       {
         "fromTable": "customer@rdonly",
         "toTables": [
           "customer.customer"
         ]
       },
       {
         "fromTable": "commerce.customer@replica",
         "toTables": [
           "customer.customer"
         ]
       },
       {
         "fromTable": "corder",
         "toTables": [
           "commerce.corder"
         ]
       },
       {
         "fromTable": "customer.corder@replica",
         "toTables": [
           "customer.corder"
         ]
       },
       {
         "fromTable": "customer.customer@replica",
         "toTables": [
           "customer.customer"
         ]
       },
       {
         "fromTable": "customer.corder",
         "toTables": [
           "commerce.corder"
         ]
       },
       {
         "fromTable": "corder@rdonly",
         "toTables": [
           "customer.corder"
         ]
       },
       {
         "fromTable": "customer.corder@rdonly",
         "toTables": [
           "customer.corder"
         ]
       },
       {
         "fromTable": "customer",
         "toTables": [
           "commerce.customer"
         ]
       },
       {
         "fromTable": "customer.customer",
         "toTables": [
           "commerce.customer"
         ]
       },
       {
         "fromTable": "commerce.customer@rdonly",
         "toTables": [
           "customer.customer"
         ]
       },
       {
         "fromTable": "corder@replica",
         "toTables": [
           "customer.corder"
         ]
       },
       {
         "fromTable": "customer@replica",
         "toTables": [
           "customer.customer"
         ]
       }
     ]
   }
   ```

## Switch write operations

9. Use `SwitchTraffic` action of `MoveTables` workflow again, but this time to switch write operation to the keyspace `customer`:

   ```bash
   $ vtctlclient MoveTables -- --tablet_types=primary SwitchTraffic customer.commerce2customer
   ```

   If you don't specify the `--tablet_types` parameter `SwitchTraffic` will start serving traffic from the target for all tablet types.

10. Check the routing rules of `commerce` keyspace again:

   ```bash
   $ vtctlclient GetRoutingRules commerce
   {
     "rules": [
       {
         "fromTable": "commerce.customer",
         "toTables": [
           "customer.customer"
         ]
       },
       {
         "fromTable": "customer",
         "toTables": [
           "customer.customer"
         ]
       },
       {
         "fromTable": "commerce.corder",
         "toTables": [
           "customer.corder"
         ]
       },
       {
         "fromTable": "corder",
         "toTables": [
           "customer.corder"
         ]
       }
     ]
   }
   ```

## Finalize the move

11. Use `Complete` action of `MoveTables` to delete the old tables:

    ```bash
    $ vtctlclient MoveTables Complete customer.commerce2customer
    ```

12. After completing the movement, selecting customers data from the keyspace `commerce` should fail:

    ```bash
    $ mysql --table < ../common/select_commerce_data.sql
    Using commerce/0
    Customer
    ERROR 1146 (42S02) at line 4: vtgate: http://localhost:15001/: target: commerce.0.primary, used tablet: zone1-100
    (localhost): vttablet: rpc error: code = NotFound desc = Table 'vt_commerce.customer' doesn't exist (errno 1146)
    (sqlstate 42S02) (CallerID: userData1): Sql: "select * from customer", BindVars: {}
    ```

    

