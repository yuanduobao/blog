# 简述为PostgreSQL添加插件的几种方法

# 开发插件的几种方法

   我目前了解的PG插件大约有以下几种：

- 利用hook，例如插件pg_stat_statement。

- 建立C函数，然后在数据库中创建SQL函数进行关联，很多系统函数就是使用这种方法，比如系统函数`pg_logical_slot_seek_changes`。

- 使用PG提供的回调函数接口，比如逻辑复制，比如pg自带的插件`test_decoding`就使用了逻辑解码插件提供的回调函数：

  ```c
  /* specify output plugin callbacks */
  void
  _PG_output_plugin_init(OutputPluginCallbacks *cb)
  {
  	AssertVariableIsOfType(&_PG_output_plugin_init, LogicalOutputPluginInit);
  
  	cb->startup_cb = pg_decode_startup;
  	cb->begin_cb = pg_decode_begin_txn;
  	cb->change_cb = pg_decode_change;
  	cb->truncate_cb = pg_decode_truncate;
  	cb->commit_cb = pg_decode_commit_txn;
  	cb->filter_by_origin_cb = pg_decode_filter;
  	cb->shutdown_cb = pg_decode_shutdown;
  	cb->message_cb = pg_decode_message;
  }
  ```

  函数调用流程：

  ```sh
  create_logical_replication_slot(SQL函数)  
       --> create_logical_replication_slot(src/backend/replication/slotfuncs.c)
       --> CreateInitDecodingContext(src/backend/replication/logical/logical.c，创建逻辑解码上下文context)
       --> StartupDecodingContext(src/backend/replication/logical/logical.c，创建逻辑解码上下文context)
       --> LoadOutputPlugin(src/backend/replication/logical/logical.c，加载插件，)
          --> load_external_function(查找初始化函数_PG_output_plugin_init,src/backend/utils/fmgr/dfmgr.c)
          --> _PG_output_plugin_init(调用插件的初始化函数，初始化回调函数接口)
  ```

  

  **PG本身就是最好的老师，大家有兴趣可以看一下contrib目录下的插件。 **

  

  下面将对两种方式进行介绍： 

  

# 利用hook创建创建

   1、利用hook建立插件，hook是PG中可以对PG运行机制进行修改的一种方式，关于hook，可以参考以下文章：

   >  https://my.oschina.net/Suregogo/blog/312848

   

   数据库中可用的hook有：

   ```sh
   [root@uat-racdb01 postgresql-12.3]# grep -r hook_type --include='*.c' src |awk -F ':' '{print $2}' |awk -F' ' '{if($1=="static"){print $2} else {print $1} }' |sort -u
   check_password_hook_type
   ClientAuthentication_hook_type
   create_upper_paths_hook_type
   emit_log_hook_type
   ExecutorCheckPerms_hook_type
   ExecutorEnd_hook_type
   ExecutorFinish_hook_type
   ExecutorRun_hook_type
   ExecutorStart_hook_type
   explain_get_index_name_hook_type
   ExplainOneQuery_hook_type
   get_attavgwidth_hook_type
   get_index_stats_hook_type
   get_relation_info_hook_type
   get_relation_stats_hook_type
   join_search_hook_type
   object_access_hook_type
   PGDLLIMPORT
   planner_hook_type
   post_parse_analyze_hook_type
   ProcessUtility_hook_type
   row_security_policy_hook_type
   set_join_pathlist_hook_type
   set_rel_pathlist_hook_type
   shmem_startup_hook_type
   [root@uat-racdb01 postgresql-12.3]# pwd
   /root/postgresql-12.3
   [root@uat-racdb01 postgresql-12.3]# grep -r hook_type --include='*.c' src |awk -F ':' '{print $2}' |awk -F' ' '{if($1=="static"){print $2} else {print $1} }' |sort -u |wc -l
   25
   [root@uat-racdb01 postgresql-12.3]# 
   ```

   

   

   具体怎么基于hook开发pg插件，可以自行百度。

   

   # 建立C函数并建立关联 

 很多系统函数就是使用这种方法创建的，比如逻辑复制相关的函数`pg_logical_slot_peek_changes`，其主要步骤是：

- 编写c语言函数，比如：

  ```sh
  src/backend/replication/logicalfuncs.c
  ```

  

  ```c
  /*
   * SQL function returning the changestream as text, only peeking ahead.
   */
  Datum
  pg_logical_slot_peek_changes(PG_FUNCTION_ARGS)
  {
  	return pg_logical_slot_get_changes_guts(fcinfo, false, false);
  }
  ```

  

  **sql函数对应的c语言函数的原因都是一样的，如下：**

  ```c
  Datum sql_function_name(PG_FUNCTION_ARGS)
  ```

  






- 编写创建sql函数的语句，将sql函数名跟C语言函数名关联起来。

  ```sh
  src/backend/catalog/system_views.sql
  ```

  

  ```sql
  CREATE OR REPLACE FUNCTION pg_logical_slot_peek_changes(
      IN slot_name name, IN upto_lsn pg_lsn, IN upto_nchanges int, VARIADIC options text[] DEFAULT '{}',
      OUT lsn pg_lsn, OUT xid xid, OUT data text)
  RETURNS SETOF RECORD
  LANGUAGE INTERNAL
  VOLATILE ROWS 1000 COST 1000
  AS 'pg_logical_slot_peek_changes';
  ```

  

​       **以上sql语句会在系统字典表`pg_proc`中增加该函数的相应信息，以后pg就可以根据该字典表找到sql函数对应的c语言函数名（sql函数名和C语言函数名可以不同）。**

当然，如果是自己开发的插件，还需要指定C语言函数所在的文件名（不含文件名后缀.so），比如插件`pg_stat_statements`的文件`pg_stat_statments--1.4.sql`中就有创建sql函数的代码：

```sql
-- Register functions.
CREATE FUNCTION pg_stat_statements_reset()
RETURNS void
AS 'MODULE_PATHNAME'
LANGUAGE C PARALLEL SAFE;

CREATE FUNCTION pg_stat_statements(IN showtext boolean,
    OUT userid oid,
    OUT dbid oid,
    OUT queryid bigint,
    OUT query text,
    OUT calls int8,
    OUT total_time float8,
    OUT min_time float8,
    OUT max_time float8,
    OUT mean_time float8,
    OUT stddev_time float8,
    OUT rows int8,
    OUT shared_blks_hit int8,
    OUT shared_blks_read int8,
    OUT shared_blks_dirtied int8,
    OUT shared_blks_written int8,
    OUT local_blks_hit int8,
    OUT local_blks_read int8,
    OUT local_blks_dirtied int8,
    OUT local_blks_written int8,
    OUT temp_blks_read int8,
    OUT temp_blks_written int8,
    OUT blk_read_time float8,
    OUT blk_write_time float8
)
RETURNS SETOF record
AS 'MODULE_PATHNAME', 'pg_stat_statements_1_3'
LANGUAGE C STRICT VOLATILE PARALLEL SAFE;
```

**其中的参数`MODULE_PATHNAME`就是动态库文件名，可以查看系统字典表`pg_proc`，字段`probin`就是动态库文件的名字（不含后缀.so)**：

```sql
postgres=# select oid,proname,prosrc,probin from pg_proc where proname='pg_stat_statements';
  oid  |      proname       |         prosrc         |           probin           
-------+--------------------+------------------------+----------------------------
 16573 | pg_stat_statements | pg_stat_statements_1_3 | $libdir/pg_stat_statements
(1 row)

postgres=# 
```



pg的配置变量`libdir`的值如下：

```sh
[root@uat-racdb01 postgresql-12.3]# pg_config |grep -i libdir
LIBDIR = /usr/local/postgres-12.3/lib
PKGLIBDIR = /usr/local/postgres-12.3/lib
[root@uat-racdb01 postgresql-12.3]# 
```

