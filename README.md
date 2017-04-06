# pgMemento

![alt text](https://github.com/pgMemento/pgMemento/blob/master/material/pgmemento_logo.png "pgMemento Logo")

pgMemento provides an audit trail for your data inside a PostgreSQL
database using triggers and server-side functions written in PL/pgSQL.
It also tracks DDL changes to enable schema versioning and offers
powerful mechanism to restore or repair past revisions.


## Index

1. License
2. About
3. System requirements
4. Background & References
5. How To
6. Future Plans
7. Media
8. Developers
9. Contact
10. Special thanks
11. Disclaimer


## 1. License

The scripts for pgMemento are open source under GNU Lesser General 
Public License Version 3.0. See the file LICENSE for more details. 


## 2. About

pgMemento logs DML and DDL changes inside a PostgreSQL database. 
These logs are bound to events and transactions and not to timestamp
fields that specify the validity interval as seen in many other auditing
approaches. This allows for rolling back events selectively by keeping 
the database consistent.

pgMemento uses triggers to log the changes. The OLD and the NEW version
of a tuple are accessable inside the corresponding trigger procedures.
pgMemento only logs the OLD version as the recent state can be queried
from the table. It pushes this priciple down on a columnar level meaning
that only deltas between OLD and NEW are stored when UPDATEs occur. Of
course, this is an overhead but it pays off in saving disk space and in
making rollbacks easier.

Logging only fragments can produce sparsely filled history/audit tables. 
Using a semistructured data type like JSONB can make the data logs more 
compact. In general, using JSONB for auditing has another big advantage:
The audit mechanism (triggers and audit tables) does not need to adapt
to schema changes. Actually, you do not even need shadow tables for each 
audited table. All logs can be written to one central table with a JSONB
field. 

![alt text](https://github.com/pgMemento/pgMemento/blob/master/material/generic_logging.png "Generic logging")

pgMemento provides functions to recreate a former table or database state
in a separate database schema incl. constraints and indexes. As event 
triggers are capturing any schema changes, the restored table or database
will have the layout of the past state.

An audit trail like pgMemento is probably not ideal for write-instensive
databases. However, as only OLD data is logged it will certainly take
less time to run out of disk space than other solutions. Nevertheless,
obsolete content can simply be removed from the logs at any time without
affecting the versioning mechanism.

pgMemento is written in plain PL/pgSQL. Thus, it can be set up on every
machine with PostgreSQL 9.5 or higher. I tagged a first version of 
pgMemento (v0.1) that uses the JSON data type and can be used along with
PostgreSQL 9.3, but it is slower and can not handle very big data as JSON
strings. Releases v0.2 and v0.3 require at least PostgreSQL 9.4. The 
master uses JSONB functions introduced in PostgreSQL 9.5. I recommend to
always use the newest version of pgMemento.


## 3. System requirements

* PostgreSQL 9.5


## 4. Background & References

The auditing approach of pgMemento is nothing new. Define triggers to log
changes in your database is a well known practice. There are other tools 
out there which can also be used. When I started the development for 
pgMemento I wasn't aware of that there are so many solutions out there
(and new ones popping up every once in while).

If you want a clearer table structure for logged data, say a history
table for each audited table, have a look at [tablelog](http://pgfoundry.org/projects/tablelog/) 
by Andreas Scherbaum. It's easy to query different versions of a row. 
Restoring former states is also possible. It writes all the data twice,
though. Runs only on Linux.

If you prefer to work with validity intervals for each row try out the
[temporal_tables](http://pgxn.org/dist/temporal_tables/) extension by Vlad Arkhipov or the
[table_version](http://pgxn.org/dist/table_version) extension by Jeremy Palmer.
[This talk](http://pgday.ru/files/papers/9/pgday.2015.magnus.hagander.tardis_orm.pdf) by Magnus Hagander goes in a similar direction.

If you like the idea of generic logging, but you prefer hstore over 
JSONB check out [audit trigger 91plus](http://wiki.postgresql.org/wiki/audit_trigger_91plus) by Craig Ringer.
It does not provide functions to restore previous database state or to 
rollback certain transactions.

If you want to use a tool, that's proven to run in production for several
years take a closer look at [Cyan Audit](http://pgxn.org/dist/cyanaudit/) by Moshe Jacobsen.
Logs are structured on columnar level, so auditing can also be switched
off for certain columns. DDL changes on tables are caught by an event 
trigger. Rollbacks of transactions are possible for single tables. 

If you think the days for using triggers for auditing are numbered because
of the new logical decoding feature of PostgreSQL you are probably right.
But this technology is still young and there are not many tools out there 
that provide the same functionality like pgMemento. A notable 
implementation is [Logicaldecoding](https://github.com/sebastian-r-schmidt/logicaldecoding) 
by Sebastian R. Schmidt. [pgaudit](https://github.com/2ndQuadrant/pgaudit) by 2ndQuadrant 
and its [fork](https://github.com/pgaudit/pgaudit) by David Steele are 
only logging transaction metadata at the moment and not the data itself.


## 5. How To

### 5.1. Add pgMemento to a database

A brief introduction about the different SQL files:
* `DDL_LOG.sql` enables logging of schema changes (DDL statements)
* `LOG_UTIL.sql` provides some helpe functions for handling the audited information
* `REVERT.sql` contains procedures to rollback changes of a certain transaction and
* `SCHEMA_MANAGEMENT.sql` includes functions to define constraints in the schema where tables have been restored
* `SETUP.sql` contains DDL scripts for tables and basic setup functions
* `VERSIONING.sql` is necessary to restore past tuple/table/database states

Run the `INSTALL_PGMEMENTO.sql` script with the psql client of 
PostgreSQL. Now a new schema will appear in your database called 
`pgmemento`. As of version 0.4 the `pgmemento` schema consist of 
5 log tables and 2 view:

* `TABLE audit_column_log`: Stores information about columns of audited tables (DDL log target)
* `TABLE audit_table_log`: Stores information about audited tables (DDL log target)
* `TABLE row_log`: Table for data log (DML log target)
* `TABLE table_event_log`: Stores metadata about table events related to transactions (DML log target)
* `TABLE transaction_log`: Stores metadata about transactions (DML log target)
* `VIEW audit_tables`: Displays tables currently audited by pgMemento incl. information about the transaction range
* `VIEW audit_tables_dependency`: Lists audited tables in order of their dependencies with each other

The following figure shows how the log tables are referenced with each
other:

![alt text](https://github.com/pgMemento/pgMemento/blob/master/material/log_tables.png "Log tables of pgMemento")


### 5.2. Start pgMemento

To enable auditing for an entire database schema simply run the `INIT.sql`
script. First, you are requested to specify the target schema. For the 
second parameter you can define a set of tables you want to exclude from
auditing (comma-separated list). As for the third parameter you can choose
if newly created tables shall be enabled for auditing automatically. 
`INIT.sql` also creates event triggers for the database to track schema
changes of audited tables.

Auditing can also be enabled manually for single tables using the 
following function, which adds an additional audit_id column to the table
and creates triggers that are fired during DML changes.

<pre>
SELECT pgmemento.create_table_audit(
  'table_A',
  'public'
);
</pre>

If `INIT.sql` has not been used event triggers can be created by calling
the following procedure:

<pre>
SELECT pgmemento.create_schema_event_trigger(1);
</pre>

By passing a 1 to the procedure an additional event trigger for 
`CREATE TABLE` events is created (not for `CREATE TABLE AS` events).

**ATTENTION:** It is important to generate a proper baseline on which a
table/database versioning can reflect on. Before you begin or continue
to work with the database and change its content, define the present 
state as the initial versioning state by executing the procedure
`pgmemento.log_table_state` (or `pgmemento.log_schema_state`). 
For each row in the audited tables another row will be written to the 
'row_log' table telling the system that it has been 'inserted' at the 
timestamp the procedure has been executed. Depending on the number of 
tables to alter and on the amount of data that has to be defined as 
INSERTed this process can take a while.

**HINT:** When setting up a new database I would recommend to start 
pgMemento after bulk imports. Otherwise the import will be slower and 
several different timestamps might appear in the transaction_log table.

Logging can be stopped and restarted by running the `STOP_AUDITING.sql`
and `START_AUDITING.sql` scripts. Note that theses scripts do not affect
(remove) the audit_id column in the logged tables.


### 5.3. Logging behaviour

#### 5.3.1. DML logging

pgMemento uses two logging stages. The first trigger is fired before 
each statement on each audited table. Every transaction is only logged 
once in the `transaction_log` table. Within the trigger procedure the 
corresponding table operations are logged as well in the `table_event_log`
table. Only one INSERT, UPDATE, DELETE and TRUNCATE event can be logged
per table per transaction. So, if two operations of the same kind are
applied against one table during one transaction the logged data is 
mapped to the first event that has been inserted into `table_event_log`.
In the next chapter you will see why this won't produce any consistency
issues.

The second logging stage is related two the data that has changed. 
Row-level triggers are fired after each operations on the audited tables. 
Within the trigger procedure the corresponding INSERT, UPDATE, DELETE or
TRUNCATE event for the current transaction is queried and each row is 
mapped against it.

For example, an UPDATE command on 'table_A' changing the value of some 
rows of 'column_B' to 'new_value' will appear in the log tables like this:

TRANSACTION_LOG

| ID  | txid_id  | stmt_date                | user_name  | client address  |
| --- |:-------- |:------------------------:|:----------:|:---------------:|
| 1   | 1000000  | 2017-02-22 15:00:00.100  | felix      | ::1/128         |

TABLE_EVENT_LOG

| ID  | transaction_id | op_id | table_operation | schema_name | table_name  | table_relid |
| --- |:--------------:|:-----:|:---------------:|:-----------:|:-----------:|:-----------:|
| 1   | 1000000        | 2     | UPDATE          | public      | table_A     | 44444444    |

ROW_LOG

| ID  | event_id  | audit_id | changes                  |
| --- |:---------:|:--------:|:------------------------:|
| 1   | 1         | 555      | {"column_B":"old_value"} |
| 2   | 1         | 556      | {"column_B":"old_value"} |
| 3   | 1         | 557      | {"column_B":"old_value"} |

As you can see only the changes are logged. DELETE (op_id = 3) and
TRUNCATE (op_id = 4) commands would cause logging of the complete rows
while INSERTs (op_id = 1) would leave a the 'changes' field blank. 
Thus, there is no data redundancy.

#### 5.3.2. DDL logging

Since v0.3 pgMemento supports DDL logging to capture schema changes.
This is important for restoring former table or database states (see 5.5).
The two tables `audit_table_log` and `audit_column_log` in the pgMemento
schema provide information at what range of transactions the audited 
tables and their columns exist. After a table is altered or dropped an
event trigger is fired to compare the recent state (at ddl_command_end) 
with the logs. pgMemento also saves data before `DROP SCHEMA`, `DROP TABLE`,
`DROP COLUMN` or `ALTER COLUMN ... TYPE ... USING` events occur
(at ddl_command_start). Dropping tables or schemas will lead to `TRUNCATE`
actions whereas field changes will be logged as either `ALTER COLUMN` or
`DROP COLUMN` events (both with op_id = 2 like UPDATEs).

**ATTENTION:** Data is NOT logged if DDL statements are called from 
functions because they can only be parsed if they sit in the top level 
query!  

If tables or columns are renamed data is not logged either, as it would
not change anyway. Comments inside query strings that fire the event
trigger are forbidden and will raise an exception. So far, changing the
data type of columns will only log the complete column if the keyword 
`USING` is found in the `ALTER TABLE` command. Also, note that 
transactions altering of dropping columns can not be reverted so far.

### 5.4. Query the logs

The logged information can already be of use, e.g. list all transactions 
that had an effect on a certain column by using the ? operator:

<pre>
SELECT t.txid 
  FROM pgmemento.transaction_log t
  JOIN pgmemento.table_event_log e ON t.txid = e.transaction_id
  JOIN pgmemento.row_log r ON r.event_id = e.id
  WHERE 
    r.audit_id = 4 
  AND 
    (r.changes ? 'column_B');
</pre>

List all rows that once had a certain value by using the @> operator:

<pre>
SELECT DISTINCT audit_id 
  FROM pgmemento.row_log
  WHERE 
    changes @> '{"column_B": "old_value"}'::jsonb;
</pre>

### 5.5. Revert certain transactions

The logged information can be used to revert certain transactions that
happened in the past. Reinsert deleted rows, remove imported data etc.
The procedure is called `revert_transaction`.

The procedure loops over each row that was affected by the given 
transaction. For data integrity reasons the order of operations and 
audit_ids is important. Imagine three tables A, B and C, with B and C
referencing A. Deleting entries in A requires deleting depending rows
in B and C. The order of events in one transaction can look like this:

<pre>
Txid 1000
1. DELETE from C
2. DELETE from A
3. DELETE from B
4. DELETE from A
</pre>

As said, pgMemento can only log one DELETE event on A. So, simply
reverting the events in reverse order won't work here. An INSERT in B 
requires exitsing entries in A.

<pre>
Revert Txid 1000
1. INSERT into B <-- ERROR: foreign key violation
2. INSERT into A
3. INSERT into C
</pre>

By joining against the `audit_tables_dependency` view we can produce the
correct revert order without violating foreign key constraints. B and C
have a higher depth than A. The order will be:

<pre>
Revert Txid 1000
1. INSERT into A
2. INSERT into B
3. INSERT into C
</pre>

For INSERTs and UPDATEs the reverse depth order is used. The same
distinction is used when resolving self-references on tables. A parent
element must be inserted before the tuples that are referencing it. 
The parent naturally has got a lower audit_id value. When reverting
INSERTs (younger) tuples with a higher audit_id need to be deleted first. 
When reverting DELETEs (older) tuples with a lower audit_id need to be
reinserted first. The ordering of audit_ids is partitioned by the
diffenrent events.

Reverting also works if foreign keys are set to `ON UPDATE CASCADE` or
`ON DELETE CASCADE` because the `audit_tables_dependency` produces the
correct order anyway and cross-referencing tuples in one table would
belong to the same event.

A range of transactions can be reverted by calling:

<pre>
SELECT pgmemento.revert_transactions(lower_txid, upper_txid);
</pre>

It uses nearly the same query but with an additional ordering by 
transaction ids (DESC). When reverting many transactions an alternative 
procedure can be used called `revert_distinct_transaction`. For each
distinct audit_it only the oldest table operation is applied to make the
revert process faster. It is also provided for transaction ranges.

<pre>
SELECT pgmemento.revert_distinct_transactions(lower_txid, upper_txid);
</pre>


### 5.6. Restore a past state of your database

A table state is restored with the procedure `pgmemento.restore_table_state
(start_from_txid, end_at_txid, 'name_of_audited_table', 'name_of_audited_schema', 'name_for_target_schema', 'VIEW', 0)`: 
* With a given range of transaction ids the user specifies the time slot
  he is interested in. If the first value is lower than the minimum txid 
  found in the transaction_log table a complete historic replica of the 
  table is created (as it looked like **before** the second given txid 
  has been executed). Note, that only then you are able to have a correct
  view on a past state of your table.
* The result is written to another schema specified by the user.
* Tables can be restored as VIEWs (default) or TABLEs. 
* If chosen VIEW the procedure can be executed again (e.g. by using
  another transaction id) and would replace the old view(s) if the last
  parameter is specified as 1.
* A whole database state might be restored with `pgmemento.restore_schema_state`.

How does the restoring work? Imagine a time line like this:

`1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8` [Transactions] <br/>
`I -> U -> D -> I -> U -> U -> D -> now` [Operations] <br/>
I = Insert, U = Update, D = Delete

After the first INSERT the row in `TABLE_A` looked like this.

TABLE_A

| ID  | column_B   | column_C | audit_id |
| --- |:----------:|:--------:|:--------:|
| 1   | some_value | abc      | 555      |

Imagine that this row is updated again in transactions 5 and 6
and deleted in transaction 7 e.g.

<pre>
UPDATE table_a SET column_B = 'final_value';
UPDATE table_a SET column_C = 'def';
DELETE FROM table_a WHERE id = 1;
</pre>

In the 'row_log' table this would be logged as follows:

| ID  | event_id  | audit_id | changes                                                           |
| --- |:---------:|:--------:|:-----------------------------------------------------------------:|
| ... | ...       | ...      | ...                                                               |
| 4   | 4         | 555      | NULL                                                              |
| ... | ...       | ...      | ...                                                               |
| 66  | 15        | 555      | {"column_B":"some_value"}                                         |
| ... | ...       | ...      | ...                                                               |
| 77  | 21        | 555      | {"column_C":"abc"}                                                |
| ... | ...       | ...      | ...                                                               |
| ... | ...       | ...      | ...                                                               |
| 99  | 81        | 555      | {"ID":1,"column_B":"final_value","column_C":"def","audit_id":555} |
| ... | ...       | ...      | ...                                                               |

#### 5.6.1. The next transaction after date x

As said in the beginning of this chapter the restore process requires a
transaction ID as a starting point for browsing through the audited logs.
Historic data will be restored as it has been when the transaction happend
excluding the changes this transaction might produce. As most users of an
audit trail solution are probably thinking in timestamps when querying 
the history of their database, the `transaction_log` table can also be 
queried using a timestamp. The next transaction id found after the given 
timestamp can be used for restoring.

<pre>
SELECT pgmemento.restore_schema_state(
  0 -- I'm using a 0 here to be sure I querying the complete history
  t.txid,
  'public',
  'test',
  'VIEW',
  1
FROM (
  SELECT txid
    FROM pgmemento.transaction_log
      WHERE stmt_date >= '2017-02-22 16:00:00'
      LIMIT 1
) t;
</pre>

Imagine if a user knows the date before the update jobs happened. In this 
case the result of the inner query would be 5.

#### 5.6.2. Fetching audit_ids (done internally)
 
For restoring, pgMemento needs to know which entries were valid when
transaction 5 started. This can be done by a simple JOIN between the log
tables querying the last event of each related audit_id using `DISTINCT ON
with ORDER BY audit_id, event_id DESC`. DELETE or TRUNCATE events would
need to filtered out later.

![alt text](https://github.com/pgMemento/pgMemento/blob/master/material/fetch_auditids_en.png "Fetching Audit_IDs")

<pre>
SELECT 
  f.audit_id,
  f.event_id,
  f.op_id 
FROM (
  SELECT DISTINCT ON (r.audit_id) 
    r.audit_id,
    r.event_id, 
    e.op_id
  FROM 
    pgmemento.row_log r
  JOIN 
    pgmemento.table_event_log e 
    ON e.id = r.event_id
  JOIN 
    pgmemento.transaction_log t
    ON t.txid = e.transaction_id
  WHERE
    t.txid >= 0 AND t.txid < 5
    AND e.table_relid = 'public.table_a'::regclass::oid
  ORDER BY 
    r.audit_id,
    e.id DESC
) f
WHERE f.op_id < 3
)
</pre>

For audit_id 555 this query would tell us that the row did exist before
transaction 5 and that its last event had been an INSERT (op_id = 1)
event with the ID 4. As said in 5.2. having a baseline where already
existing data is marked as INSERTed is really important because of this
query. If there is no initial event found for an audit_id it will not be
restored.

#### 5.6.3. Find the right historic values

For each fetched audit_id a row has to be reconstructed. This is where
things become very tricky because the historic field values can be 
scattered all throughout the row_log table due to the pgMemento's logging
behaviour (see example in 5.6). For each column we need to find JSONB
objects containing the column's name as a key. As learned in chapter 5.4
we could seach for e.g. `(changes ? 'column_B')` plus the audit_id. This
would give us two entries:

| changes                                                           |
| ----------------------------------------------------------------- |
| {"column_B":"some_value"}                                         |
| {"ID":1,"column_B":"final_value","column_C":"def","audit_id":555} |

By sorting on the internal ID of the row_log table we get the correct 
historic order of these logs. The value in the event_id column must be
bigger than the event ID we have extracted in the previous step. 
Otherwise, we would also get the logs of former revisions which are
already outdated by the time transaction 5 happened.

![alt text](https://github.com/pgMemento/pgMemento/blob/master/material/fetch_values_en.png "Fetching values")

So, the first entry we find for `column_B` is {"column_B":"some_value"}.
This log has been produced by the UPDATE event of transaction 5. Thus,
before transaction 5 `column_B` had the value "some_value". This is what
we have asked for. We do the same query for all the other columns. For
`column_C` we find the log `{"column_C":"abc"}`. So, the historic value
before transaction 5 has been "abc". For columns ID and audit_id there
is only one JSONB log found: The entire row, generated by the DELETE 
query of transaction 7. We can also find other values for the two fields
`column_B` and `column_C` in this log but they were created after
transaction 5.

Imagine if the row would not have been deleted, we would not find any
logs for e.g. the ID column. In this case, we need to query the recent
state of the table. We would have to consider that the table or the
column could have been renamed or that the column could have been dropped.
pgMemento takes this into account. If nothing is found at all (which
would not be reasonable) the value will be NULL.

#### 5.6.4. Window functions to bring it all together (done internally)

Until pgMemento v0.3 the retrieval of historic values was rolled out
in seperate queries for each column. This was too inefficient for a 
quick view into the past. 

<pre>
SELECT
  key1, -- ID
  value1,
  key2, -- column_B
  value2,
  ...
FROM (
  ... -- subquery from 5.6.2. (extracted event_ids and audit_ids)
) f
JOIN LATERAL(
  SELECT
    q1.key AS key1, -- ID
    q1.value AS value1,
    q2.key AS key2, -- column_B
    q2.value AS value2,
    ...
  FROM 
    (...) q1,
    (SELECT
       -- set constant for column name
       'column_B' AS key,
       -- set value, use COALESCE to handle NULLs
       COALESCE(
         -- get value from JSONB log
         (SELECT
            (changes -> 'column_B') 
          FROM 
            pgmemento.row_log
          WHERE
            audit_id = f.audit_id
            AND event_id > f.event_id
            AND (changes ? 'column_B')
          ORDER BY
            r.id
          LIMIT 1
         ),
         -- if NULL, query recent value
         (SELECT
            to_jsonb(column_B)
          FROM
            public.table_A
          WHERE
            audit_id = f.audit_id
         ),
         -- no logs, no current value = NULL
         NULL
       ) AS value
    ) q2,
    ...
)
  ON (true)
</pre>

Since v0.4 pgMemento uses a window function
with `FILTER` clauses that were introduced in PostgreSQL 9.4. This allows
for searching for different keys on same level of the query. A filter
can only be used in conjunction with an aggregate function. Luckily,
with jsonb_agg PostgreSQL offers a suitable function for the JSONB
logs. The window is ordered by the ID of the row_log table to get
the oldest log first. The window frame starts at the current row and
has no upper boundary.

<pre>
SELECT
  q.key1 , -- ID
  q.value1->>0,
  q.key2, -- column_B
  q.value2->>0,
  ...
FROM (
  SELECT DISTINCT ON (a.audit_id, x.audit_id)
    -- set constant for column name
    'id'::text AS key1,
    -- set value, use COALESCE to handle NULLs
    COALESCE(
      -- get value from JSONB log
      jsonb_agg(changes -> 'id')
        FILTER (WHERE changes ? 'id')
          OVER (ORDER BY a.id ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING),
      -- if NULL, query recent value
      to_jsonb(x.id),
      -- no logs, no current value = NULL
      NULL
    ) AS value1,
    'id'::text AS key2,
    -- set value, use COALESCE to handle NULLs
    COALESCE(
      jsonb_agg(changes -> 'column_B')
        FILTER (WHERE changes ? 'column_B')
          OVER (ORDER BY a.id ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING),
      to_jsonb(x.column_B),
      NULL
    ) AS value2,
    ...
  FROM (
    ... -- subquery from 5.6.2. (extracted event_ids and audit_ids)
  ) f
  LEFT JOIN
    pgmemento.row_log a 
    ON a.audit_id = f.audit_id
    AND a.event_id > f.event_id
  LEFT JOIN public.table_A x
    ON x.audit_id = f.audit_id
  WHERE
    f.op_id < 3
    ORDER BY a.audit_id, x.audit_id, a.id
) q
</pre>

Now, the row_log table and the audited table only appear once in the
query. They have to be joined with an `OUTER JOIN` against the queried list
of valid audit_ids because both could be missing an audit_id. As said,
this is very unlikely. For each audit_id only the first entry of the
result is of interest. This is done again with `DISTINCT ON`. As we are
using a window query the extracted JSONB array for each key can contain
all further historic values of the audit_id found in the logs
(`ROW BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING`), no matter if we 
strip out rows with `DISTINCT ON`. Only the first element if the array
(`ORDER BY a.id`) is necessary. Therefore, it has to be extracted in the
upper query with the `->>` operator.

#### 5.6.5. Generating JSONB objects and recreate table (done internally)

Now, that we got an alternating list of keys and values we could simply
call the PostgreSQL function jsonb_build_object to produce complete
tuples as JSONB. The last step of the restoring process is to bring these
generated JSONB objects into a tabular representation. PostgreSQL offers
the function `jsonb_populate_record` to do this job.

<pre>
SELECT * FROM jsonb_populate_record(null::table_A, jsonb_object);
</pre>

But it cannot be written just like that because we need to combine it
with a query that returns numerous JSONB objects. There is also the 
function `jsonb_populate_recordset` to return a set of records but it
needs all JSONB objects to be aggregated which is a little overhead.
The solution is to use a `LATERAL JOIN`:

<pre>
SELECT 
  p.*
FROM (
  SELECT 
    jsonb_build_object(
      q.key1 , -- ID
      q.value1->>0,
      q.key2, -- column_B
      q.value2->>0,
      ...
    ) AS log_entry
  FROM (
    ... -- query from previous steps
  ) q
) rq
JOIN LATERAL (
  SELECT 
    * 
  FROM
    jsonb_populate_record(
      null::table_A, -- template table
      rq.log_entry   -- reconstructed row as JSONB
    )
) p
  ON (true)
</pre>

This is also the moment when the DDL log tables are getting relevant.
In order to produce a correct historic replica of a table, the table 
schema for the requested time (transaction) window has to be known.
The upper transaction boundary is used to query the `audit_column_log`
table to reconstruct a template that reflects the historic table schema. 

The template are created as temporary tables. This means when restoring
the audited tables as VIEWs they only exist as long as the current
sessions lasts (`ON COMMIT PRESERVE ROWS`). When creating a new session
the restore procedure has to be called again. It doesn't matter if the
target schema already exist. When restoring the audited tables as BASE 
TABLEs, they will of course remain in the target schema but occupying
extra disk space.

#### 5.5.5. Restore revisions of a certain tuple

It is also possible to restore only revisions of a certain tuple with 
the function `pgmemento.generate_log_entry`. It requires a transaction
ID, the audit_id of the tuple and the corresponding table and schema 
name.

<pre>
SELECT 
  row_number() OVER () AS revision_no,
  p.*
FROM (
  SELECT
    pgmemento.generate_log_entry(
      e.transaction_id,
      r.audit_id,
      'my_table',
      'public'
    ) AS entry
  FROM 
    pgmemento.row_log r
  JOIN
    pgmemento.table_event_log e 
    ON e.id = r.event_id
  WHERE 
    r.audit_id = 12345
  ORDER BY
    e.transaction_id DESC,
	e.id DESC
) log
JOIN LATERAL ( 
  SELECT
    *
  FROM
    jsonb_populate_record(
      null::public.my_table,
      log.entry
    )
) p
  ON (true); 
</pre>


#### 5.5.6. Work with the past state

If past states were restored as tables they do not have primary keys 
or indexes assigned to them. References between tables are lost as well. 
If the user wants to work on the restored table or database state - 
like he would do with the production state - he can use the procedures
`pgmemento.pkey_table_state`, `pgmemento.fkey_table_state` and 
`pgmemento.index_table_state`. These procedures create primary keys,
foreign keys and indexes on behalf of the recent constraints defined
in the production schema. 

Note that if table and/or database structures have changed fundamentally 
over time it might not be possible to recreate constraints and indexes as 
their metadata is not yet logged by pgMemento. 


### 5.6. Uninstall pgMemento

In order to stop and remove pgMemento simply run the `UNINSTALL_PGMEMENTO.sql`
script.


## 6. Future Plans

Here are some plans I have for the next release:
* Have a test logic for all procedures
* Add revert for DDL changes
* Have log tables for primary keys, constraints, indexes etc.
* Have a view to store metadata of additional created schemas
  for former table / database states.

General thoughts:
* Better protection for log tables?
* Table partitioning strategy for row_log table (maybe [pg_pathman](https://github.com/postgrespro/pg_pathman) can help)
* Build a pgMemento PostgreSQL extension

I would be very happy if there are other PostgreSQL developers out there
who are interested in pgMemento and willing to help me to improve it.
Together we might create a powerful, easy-to-use versioning approach for
PostgreSQL.


## 7. Media

I gave a presentation in german at FOSSGIS 2015:
https://www.youtube.com/watch?v=EqLkLNyI6Yk

I gave another presentation in FOSSGIS-NA 2016:
http://slides.com/fxku/pgmemento_foss4gna16


## 8. Developers

Felix Kunde


## 9. Contact

felix-kunde@gmx.de


## 10. Special Thanks

* Adam Brusselback --> benchmarking and bugfixing
* Hans-Jürgen Schönig (Cybertech) --> recommend to use a generic JSON auditing
* Christophe Pettus (PGX) --> recommend to only log changes
* Claus Nagel (virtualcitySYSTEMS) --> conceptual advices about logging
* Ollyc (Stackoverflow) --> Query to list all foreign keys of a table
* Denis de Bernardy (Stackoverflow, mesoconcepts) --> Query to list all indexes of a table
* Ugur Yilmaz --> feedback and suggestions


## 11. Disclaimer

pgMemento IS PROVIDED "AS IS" AND "WITH ALL FAULTS." 
I MAKE NO REPRESENTATIONS OR WARRANTIES OF ANY KIND CONCERNING THE 
QUALITY, SAFETY OR SUITABILITY OF THE SKRIPTS, EITHER EXPRESSED OR 
IMPLIED, INCLUDING WITHOUT LIMITATION ANY IMPLIED WARRANTIES OF 
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, OR NON-INFRINGEMENT.

IN NO EVENT WILL I BE LIABLE FOR ANY INDIRECT, PUNITIVE, SPECIAL, 
INCIDENTAL OR CONSEQUENTIAL DAMAGES HOWEVER THEY MAY ARISE AND EVEN IF 
I HAVE BEEN PREVIOUSLY ADVISED OF THE POSSIBILITY OF SUCH DAMAGES.
STILL, I WOULD FEEL SORRY.
