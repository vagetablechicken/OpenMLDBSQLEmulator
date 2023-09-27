# Simulator of OpenMLDB

## CLI framework
https://code.google.com/archive/p/cliche/wikis/Manual.wiki github https://github.com/budhash/cliche 

MIT License

https://docs.spring.io/spring-shell/docs/3.1.3/docs/index.html 更丰富，但可以先用简单的cliche，结构上没有太大差别

## Build

You can build the simulator without toydb(so you can't `run`). But if you want, prepare a toydb package, and put it in `src/main/resources`, then you can `run` in simulator.

```bash
# without toydb
mvn package -DskipTests

# with toydb, better to run test
cp toydb_run_engine src/main/resources
mvn package
```

## CLI commands

`#` is comment.

Can't run a command in multi lines, e.g. `val select * from t1;` can't be written as
```
val select *
from
t1;
```

There are some quite useful builtin commands.
```
?help to hint
?list to get all cmds
!run-script filename reads and executes commands from given file.
```

You can run simulator script by `!run-script filename`, and the script file is just a text file with commands in it.
e.g.
```
!run-script src/test/resources/simple.sim
```

Extra:
```
!set-display-time true/false toggles displaying of command execution time. Time is shown in milliseconds and includes only your method's physical time.
!enable-logging filename and !disable-logging control logging, i.e. duplication of all Shell's input and output in a file.
```

Now we have some commands:

### creations

Note that if exists, the table will be replaced. Default db is `simdb`.

- `use <db>` use db, create if not exists
- `addtable <table_name> c1 t1,c2 t2, ...` create/replace table in current db
    - abbreviate: `t <table_name> c1 t1,c2 t2, ...`

- `adddbtable <db_name> <table_name> c1 t1,c2 t2, ...` create/replace table in specified db, if db not exists, create it
    - abbreviate: `dt <table_name> c1 t1,c2 t2, ...`
- `sql <create table sql>` create table by sql

- `showtables` / `st` list all tables

### `genddl <sql>`

Addtable before, and want to deploy the sql, then use this to generate ddl which has the precise index.

But the method `genDDL` in openmldb jdbc hasn't support multi db yet, so we can't use this method to parse sqls that have multi db.

- Example1
```
addt t1 "a int, b int"
addt t2 "a int, b int"
genddl select *, count(b) over w1 from t1 window w1 as (partition by a order by b rows between 1 preceding and current row)
```
output:
```
CREATE TABLE IF NOT EXISTS t1(
	a int,
	b int,
	index(key=(a), ttl=1, ttl_type=latest, ts=`b`)
);
CREATE TABLE IF NOT EXISTS t2(
	a int,
	b int
);
```
No deploy about t2, so the sql that create t2 is just a simple create table. And the sql that create t1 is a create table with index.

- Example2
```
addt t1 "a int, b int"
addt t2 "a int, b int"
genddl select *, count(b) over w1 from t1 window w1 as (union t2 partition by a order by b rows_range between 1d preceding and current row)
```
output:
```
CREATE TABLE IF NOT EXISTS t1(
	a int,
	b int,
	index(key=(a), ttl=1440m, ttl_type=absolute, ts=`b`)
);
CREATE TABLE IF NOT EXISTS t2(
	a int,
	b int,
	index(key=(a), ttl=1440m, ttl_type=absolute, ts=`b`)
);
```
It's a union window, so the sqls that create t1 and t2 both have index.

### validations

- `val <sql>` validate sql in batch mode, tables should be created before
- `valreq <sql>` validate sql in request mode, tables should be created before

### run in toydb

`run <yaml_file>` can run a yaml file in toydb, and the yaml file can be generated by `gencase`. Only support one case now.

Each case should have table creations and one sql. We'll run the sql (default mode is `request`, you can set `batch`) and get the result, then compare the result with the expected result.

You can generate such a yaml file to reproduce a bug, and ask for help.

TODO yaml可保存，但要将表和SQL都转移到真实集群还需要用户手动操作，导出sql（创建库表+上线SQL）可以方便用户导入真实集群。

由于表都有数据，要支持CLI去加表并添加数据，操作挺繁杂的，暂时不支持。也就是，“run in toydb”不能中途增删表，表必须存在于yaml中。但可以中途val sql，如果你想真实run这个sql，dump sql到yaml，然后run in toydb。

```bash
# step 1
gencase
# step 2 modify the yaml file to add table and data

# step 3 load yaml to get table catalog, then val sql in simulator, or you can skip this step (just write the sql in yaml)
loadcase
valreq <sql>
# dump will rewrite the yaml file, discard the old one, comments will be lost, be careful
dumpcase <sql>

# step 4 run sql in toydb
run
```

TODO toydb package?

TODO sql log? cmd log? history? can't use arrow in CLI
