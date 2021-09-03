# 开发限制用户权限的插件-yuandb

## 插件功能说明

禁止数据库OPR用户（xxxxOPR）执行工具命令，即只能执行select、insert、update、delete等语句。这样OPR用户就可以放心给应用程序连接数据库了，而不用担心应用在线执行ddl语句从而导致数据库性能问题。



## 注意事项

- 基于hook开发的插件，在使用`drop extension`语句删除插件时，PG并不会调用插件的`_PG_fini`函数，因此，插件在初始化函数`_PG_init`中修改的钩子没有恢复到之前的值，虽然插件已经被卸载了，但是插件对应的钩子函数其实还是会继续生效，这一点本人已经使用gdb工具调试了插件`pg_stat_statements`验证了。

- 删除插件`pg_stat_statements`时，只是删除了插件创建的几个sql函数和view（函数`pg_stat_statements`和视图`pg_stat_statements`)，以下是创建的sql文件中的定义：

  ```sql
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
  
  -- Register a view on the function for ease of use.
  CREATE VIEW pg_stat_statements AS
    SELECT * FROM pg_stat_statements(true);
  
  GRANT SELECT ON pg_stat_statements TO PUBLIC;
  ```

- 删除创建`pg_stat_statements`后，如果没有重启数据库，还可以通过执行以下sql语句，确认该插件还是在工作的（当然也可以使用gdb调试验证）：

  **使用插件的的文件名替换上述代码中的`MODULE_PATHNAME`，就可以创建函数pg_stat_statements了。**通以下语句创建了函数和view后，可以查看该view，发现之前的数据都还存在。**注意，插件pg_stat_statements的记录的内容都是保存在内存中的。**

  ```sql
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
  AS 'pg_stat_statements', 'pg_stat_statements_1_3'
  LANGUAGE C STRICT VOLATILE PARALLEL SAFE;
  ```

  



## 创建执行效果

安装插件后，opr用户就不能再执行ddl语句了，否则就会报错。

```sql
postgres=# \c postgres testopr
You are now connected to database "postgres" as user "testopr".
postgres=> \d
                    List of relations
 Schema |       Name       |       Type        |  Owner  
--------+------------------+-------------------+---------
 public | books            | table             | admin
 public | books_id_seq     | sequence          | admin
 public | pgbench_accounts | table             | admin
 public | pgbench_branches | table             | admin
 public | pgbench_history  | table             | admin
 public | pgbench_tellers  | table             | admin
 public | t_hash           | partitioned table | admin
 public | t_hash0          | table             | admin
 public | t_hash1          | table             | admin
 public | t_hash1_subp     | partitioned table | admin
 public | t_hash1_subp1    | table             | admin
 public | t_hash1_subp5    | table             | admin
 public | t_hash2          | table             | admin
 public | t_hash2_subp1    | partitioned table | admin
 public | t_hash2_subp2    | table             | admin
 public | t_hash2_supb3    | table             | admin
 public | t_hash3          | table             | admin
 public | t_hash_subp1     | table             | admin
 public | test_ddl         | table             | testopr
(19 rows)

postgres=> drop table test_ddl;    -- opr用户执行drop table命令报错。
ERROR:  OPR user cannot execute ddl statements.
postgres=> \d
                    List of relations
 Schema |       Name       |       Type        |  Owner  
--------+------------------+-------------------+---------
 public | books            | table             | admin
 public | books_id_seq     | sequence          | admin
 public | pgbench_accounts | table             | admin
 public | pgbench_branches | table             | admin
 public | pgbench_history  | table             | admin
 public | pgbench_tellers  | table             | admin
 public | t_hash           | partitioned table | admin
 public | t_hash0          | table             | admin
 public | t_hash1          | table             | admin
 public | t_hash1_subp     | partitioned table | admin
 public | t_hash1_subp1    | table             | admin
 public | t_hash1_subp5    | table             | admin
 public | t_hash2          | table             | admin
 public | t_hash2_subp1    | partitioned table | admin
 public | t_hash2_subp2    | table             | admin
 public | t_hash2_supb3    | table             | admin
 public | t_hash3          | table             | admin
 public | t_hash_subp1     | table             | admin
 public | test_ddl         | table             | testopr   -- 该表没有被删除
(19 rows)

postgres=> 
```



# 插件的安装

## 编译安装

- 将插件`user_acl`放到postgresql的源码的contrbi目录下：

  ```sh
  [root@uat-racdb01 user_acl]# pwd
  /root/postgresql-12.3/contrib/user_acl
  [root@uat-racdb01 user_acl]# tree
  .
  ├── Makefile
  ├── user_acl--1.0.sql
  ├── user_acl.c
  └── user_acl.control
  
  0 directories, 4 files
  [root@uat-racdb01 user_acl]# 
  ```

  



- 进入插件`user_acl`的根目录，然后执行以下命令：

```c
make
make install
```



```sh
root@uat-racdb01 user_acl]# make
make -C ../../src/backend generated-headers
make[1]: 进入目录“/root/postgresql-12.3/src/backend”
make -C catalog distprep generated-header-symlinks
make[2]: 进入目录“/root/postgresql-12.3/src/backend/catalog”
make[2]: 对“distprep”无需做任何事。
make[2]: 对“generated-header-symlinks”无需做任何事。
make[2]: 离开目录“/root/postgresql-12.3/src/backend/catalog”
make -C utils distprep generated-header-symlinks
make[2]: 进入目录“/root/postgresql-12.3/src/backend/utils”
make[2]: 对“distprep”无需做任何事。
make[2]: 对“generated-header-symlinks”无需做任何事。
make[2]: 离开目录“/root/postgresql-12.3/src/backend/utils”
make[1]: 离开目录“/root/postgresql-12.3/src/backend”
gcc -std=gnu99 -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -g -O0 -fPIC -I. -I. -I../../src/include  -D_GNU_SOURCE   -c -o user_acl.o user_acl.c
gcc -std=gnu99 -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -g -O0 -fPIC -shared -o user_acl.so user_acl.o -L../../src/port -L../../src/common    -Wl,--as-needed -Wl,-rpath,'/usr/local/postgres-12.3/lib',--enable-new-dtags  
[root@uat-racdb01 user_acl]# make install
make -C ../../src/backend generated-headers
make[1]: 进入目录“/root/postgresql-12.3/src/backend”
make -C catalog distprep generated-header-symlinks
make[2]: 进入目录“/root/postgresql-12.3/src/backend/catalog”
make[2]: 对“distprep”无需做任何事。
make[2]: 对“generated-header-symlinks”无需做任何事。
make[2]: 离开目录“/root/postgresql-12.3/src/backend/catalog”
make -C utils distprep generated-header-symlinks
make[2]: 进入目录“/root/postgresql-12.3/src/backend/utils”
make[2]: 对“distprep”无需做任何事。
make[2]: 对“generated-header-symlinks”无需做任何事。
make[2]: 离开目录“/root/postgresql-12.3/src/backend/utils”
make[1]: 离开目录“/root/postgresql-12.3/src/backend”
/usr/bin/mkdir -p '/usr/local/postgres-12.3/lib'
/usr/bin/mkdir -p '/usr/local/postgres-12.3/share/extension'
/usr/bin/mkdir -p '/usr/local/postgres-12.3/share/extension'
/usr/bin/install -c -m 755  user_acl.so '/usr/local/postgres-12.3/lib/user_acl.so'
/usr/bin/install -c -m 644 ./user_acl.control '/usr/local/postgres-12.3/share/extension/'
/usr/bin/install -c -m 644 ./user_acl--1.0.sql  '/usr/local/postgres-12.3/share/extension/'
[root@uat-racdb01 user_acl]# 
```



## 修改PostgreSQL的配置文件

修改PostgreSQL的配置文件，为参数`shared_preload_libraries`增加我们的插件`user_acl`：

```sh
[postgres@uat-racdb01 pg12_5433]$ grep shared_preload postgresql.conf
shared_preload_libraries = 'user_acl'	# (change requires restart)
[postgres@uat-racdb01 pg12_5433]$ 
```



## 数据库安装插件

```sql
postgres=# create extension user_acl;
CREATE EXTENSION
```



## 验证插件

- testopr用户不能删除表

  ```sql
  postgres=# \c postgres testopr;
  You are now connected to database "postgres" as user "testopr".
  postgres=> \d
                      List of relations
   Schema |       Name       |       Type        |  Owner  
  --------+------------------+-------------------+---------
   public | books            | table             | admin
   public | books_id_seq     | sequence          | admin
   public | pgbench_accounts | table             | admin
   public | pgbench_branches | table             | admin
   public | pgbench_history  | table             | admin
   public | pgbench_tellers  | table             | admin
   public | t_hash           | partitioned table | admin
   public | t_hash0          | table             | admin
   public | t_hash1          | table             | admin
   public | t_hash1_subp     | partitioned table | admin
   public | t_hash1_subp1    | table             | admin
   public | t_hash1_subp5    | table             | admin
   public | t_hash2          | table             | admin
   public | t_hash2_subp1    | partitioned table | admin
   public | t_hash2_subp2    | table             | admin
   public | t_hash2_supb3    | table             | admin
   public | t_hash3          | table             | admin
   public | t_hash_subp1     | table             | admin
   public | test_ddl         | table             | testopr
  (19 rows)
  
  postgres=> drop table test_ddl;
  ERROR:  OPR user cannot execute ddl statements.
  postgres=> 
  ```

- opr用户不能创建表、truncate表

  ```sql
  postgres=# \c postgres testopr;
  You are now connected to database "postgres" as user "testopr".
  postgres=> create table test_books(id int);
  ERROR:  OPR user cannot execute ddl statements.
  postgres=> truncate table books;
  ERROR:  OPR user cannot execute ddl statements.
  postgres=> 
  ```

- opr用户不能增加列

  ```sql
  postgres=> \conninfo
  You are connected to database "postgres" as user "testopr" on host "localhost" (address "::1") at port "5433".
  postgres=>  alter table books add column flag int;
  ERROR:  OPR user cannot execute ddl statements.
  postgres=>
  ```

  



## 其他用户可以正常删除表

```sql
[postgres@uat-racdb01 pg12_5433]$ psql -h localhost -U admin postgres -p 5433
psql (12.3)
Type "help" for help.

postgres=# \d
                    List of relations
 Schema |       Name       |       Type        |  Owner  
--------+------------------+-------------------+---------
 public | books            | table             | admin
 public | books_id_seq     | sequence          | admin
 public | pgbench_accounts | table             | admin
 public | pgbench_branches | table             | admin
 public | pgbench_history  | table             | admin
 public | pgbench_tellers  | table             | admin
 public | t_hash           | partitioned table | admin
 public | t_hash0          | table             | admin
 public | t_hash1          | table             | admin
 public | t_hash1_subp     | partitioned table | admin
 public | t_hash1_subp1    | table             | admin
 public | t_hash1_subp5    | table             | admin
 public | t_hash2          | table             | admin
 public | t_hash2_subp1    | partitioned table | admin
 public | t_hash2_subp2    | table             | admin
 public | t_hash2_supb3    | table             | admin
 public | t_hash3          | table             | admin
 public | t_hash_subp1     | table             | admin
 public | test_ddl         | table             | testopr
(19 rows)

postgres=# drop table test_ddl;
DROP TABLE
postgres=# 
```





# 插件代码

## 文件`user_acl.c`

```c
/*-------------------------------------------------------------------------
 *
 * user_acl.c
 *	  user privilege admin:
 *       These users whose name ends with 'opr' could not execute ddl statments.
 *
 * Copyright (c) 2009-2019, PostgreSQL Global Development Group
 *
 * IDENTIFICATION
 *	  contrib/user_acl/user_acl.c
 *
 *-------------------------------------------------------------------------
 */

#include "postgres.h"


#include "tcop/utility.h"
#include "miscadmin.h"
#include "utils/elog.h"

/*
#include "catalog/namespace.h"
#include "catalog/pg_ts_dict.h"
#include "commands/defrem.h"
#include "lib/stringinfo.h"
#include "tsearch/ts_cache.h"
#include "tsearch/ts_locale.h"
#include "tsearch/ts_public.h"
#include "utils/builtins.h"
#include "utils/lsyscache.h"
#include "utils/regproc.h"
#include "utils/syscache.h"
*/

PG_MODULE_MAGIC;

/*---- Function declarations ----*/

void		_PG_init(void);
void		_PG_fini(void);

//static ExecutorStart_hook_type prev_ExecutorStart = NULL;
//static ExecutorRun_hook_type prev_ExecutorRun = NULL;
static ProcessUtility_hook_type prev_ProcessUtility = NULL;

void uacl_ProcessUtility(PlannedStmt *pstmt, const char *queryString,
								ProcessUtilityContext context, ParamListInfo params,
								QueryEnvironment *queryEnv,
								DestReceiver *dest, char *completionTag);

/*
 * Module load callback
 */
void
_PG_init(void)
{
	/*
	 * Install hooks.
	 */
	//prev_ExecutorStart = ExecutorStart_hook;    /* 生成执行计划的hook */
	//ExecutorStart_hook = pgss_ExecutorStart;
	//prev_ExecutorRun = ExecutorRun_hook;    /* 执行sql语句. */
	//ExecutorRun_hook = pgss_ExecutorRun;
	//prev_ExecutorFinish = ExecutorFinish_hook;
	//ExecutorFinish_hook = pgss_ExecutorFinish;
	//prev_ExecutorEnd = ExecutorEnd_hook;
	//ExecutorEnd_hook = pgss_ExecutorEnd;
	prev_ProcessUtility = ProcessUtility_hook;   /* 非DML语句的执行钩子，比如create table、alter table语句. */
	ProcessUtility_hook = uacl_ProcessUtility;
}

/*
 * Module unload callback
 */
void
_PG_fini(void)
{
	/* Uninstall hooks. */
	ProcessUtility_hook = prev_ProcessUtility;
}


/* Hook for plugins to get control in ProcessUtility() */
void uacl_ProcessUtility(PlannedStmt *pstmt,
						const char *queryString, ProcessUtilityContext context,
						ParamListInfo params,
						QueryEnvironment *queryEnv,
						DestReceiver *dest, char *completionTag)
{
    char *username;
    char *pos;


    username = GetUserNameFromId(GetUserId(), false);
    pos = strstr(username, "opr");
    if( pos != NULL && (pos - username) +3 == strlen(username))
    {
        ereport(ERROR, 
                (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
                 errmsg("OPR user cannot execute ddl statements.")));
    }

    if(prev_ProcessUtility)
    {
       prev_ProcessUtility(pstmt, queryString, context, params, queryEnv, dest, completionTag);
    }
    else
    {
        standard_ProcessUtility(pstmt, queryString, context, params, queryEnv, dest, completionTag);
    }
    

}
```





## Makefile

```makefile
# contrib/brother/Makefile

MODULE_big = user_acl
OBJS = user_acl.o

EXTENSION = user_acl
DATA = user_acl--1.0.sql

# uncomment the following two lines to enable cracklib support
# PG_CPPFLAGS = -DUSE_CRACKLIB '-DCRACKLIB_DICTPATH="/usr/lib/cracklib_dict"'
# SHLIB_LINK = -lcrack

ifdef USE_PGXS
PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
else
subdir = contrib/user_acl
top_builddir = ../..
include $(top_builddir)/src/Makefile.global
include $(top_srcdir)/contrib/contrib-global.mk
endif 
```





## 插件的control文件

### 文件路径

```sh
contrib/user_acl/user_acl.control
```



### 文件内容

```sh
# user_acl extension
comment = 'probit opr user executing ddl statements.'
default_version = '1.0'
module_pathname = '$libdir/user_acl'
relocatable = true
```





##  插件的sql文件

### 文件路径

```sh
contrib/user_acl/user_acl--1.0.sql
```



### 文件内容

```sh
-- complain if script is sourced in psql, rather than via CREATE EXTENSION
\echo Use "CREATE EXTENSION user_acl" to load this file. \quit
```





# 删除插件

## PG基于钩子的插件删除是不彻底的

**PG通过语句`DROP EXTENSION EXTENSION_NAME`删除插件时，其实并没有调用插件的函数`_PG_fini`，所以插件在函数`_PG_init`中对钩子的修改并没有恢复。因此，即使删除的插件，其实插件还是会通过钩子被PG调用的。**这一点，我已经通过gdb调试验证了。而且pg的以下代码段也明确说明了`drop extension`只是清理系统表`pg_extension`中对应的信息。

- 文件名

  ```sh
  src/backend/commands
  ```

- 删除插件的函数`RemoveExtensionById`

```c
/*
 * Guts of extension deletion.
 *
 * All we need do here is remove the pg_extension tuple itself.  Everything
 * else is taken care of by the dependency infrastructure.
 */
void
RemoveExtensionById(Oid extId)
{
	Relation	rel;
	SysScanDesc scandesc;
	HeapTuple	tuple;
	ScanKeyData entry[1];

	/*
	 * Disallow deletion of any extension that's currently open for insertion;
	 * else subsequent executions of recordDependencyOnCurrentExtension()
	 * could create dangling pg_depend records that refer to a no-longer-valid
	 * pg_extension OID.  This is needed not so much because we think people
	 * might write "DROP EXTENSION foo" in foo's own script files, as because
	 * errors in dependency management in extension script files could give
	 * rise to cases where an extension is dropped as a result of recursing
	 * from some contained object.  Because of that, we must test for the case
	 * here, not at some higher level of the DROP EXTENSION command.
	 */
	if (extId == CurrentExtensionObject)
		ereport(ERROR,
				(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
				 errmsg("cannot drop extension \"%s\" because it is being modified",
						get_extension_name(extId))));

	rel = table_open(ExtensionRelationId, RowExclusiveLock);

	ScanKeyInit(&entry[0],
				Anum_pg_extension_oid,
				BTEqualStrategyNumber, F_OIDEQ,
				ObjectIdGetDatum(extId));
	scandesc = systable_beginscan(rel, ExtensionOidIndexId, true,
								  NULL, 1, entry);

	tuple = systable_getnext(scandesc);

	/* We assume that there can be at most one matching tuple */
	if (HeapTupleIsValid(tuple))
		CatalogTupleDelete(rel, &tuple->t_self);

	systable_endscan(scandesc);

	table_close(rel, RowExclusiveLock);
}
```





## 函数调用栈

```gdb
(gdb) bt
#0  RemoveExtensionById (extId=16535) at extension.c:1865
#1  0x0000000000581d23 in doDeletion (object=0x231cb98, flags=0) at dependency.c:1484
#2  0x000000000058185d in deleteOneObject (object=0x231cb98, depRel=0x7ffe416e65f0, flags=0) at dependency.c:1259
#3  0x000000000058048e in deleteObjectsInList (targetObjects=0x231cb60, depRel=0x7ffe416e65f0, flags=0) at dependency.c:269
#4  0x0000000000580651 in performMultipleDeletions (objects=0x2262fe0, behavior=DROP_RESTRICT, flags=0) at dependency.c:430
#5  0x000000000065041a in RemoveObjects (stmt=0x223eb10) at dropcmds.c:126
#6  0x00000000008e7fef in ExecDropStmt (stmt=0x223eb10, isTopLevel=true) at utility.c:1749
#7  0x00000000008e7c0d in ProcessUtilitySlow (pstate=0x2262ec8, pstmt=0x223ee70, queryString=0x223e068 "drop extension user_acl;", 
    context=PROCESS_UTILITY_TOPLEVEL, params=0x0, queryEnv=0x0, dest=0x223ef68, completionTag=0x7ffe416e6dd0 "") at utility.c:1591
#8  0x00000000008e6421 in standard_ProcessUtility (pstmt=0x223ee70, queryString=0x223e068 "drop extension user_acl;", context=PROCESS_UTILITY_TOPLEVEL, 
    params=0x0, queryEnv=0x0, dest=0x223ef68, completionTag=0x7ffe416e6dd0 "") at utility.c:839
#9  0x00007fec83644bd0 in uacl_ProcessUtility (pstmt=0x223ee70, queryString=0x223e068 "drop extension user_acl;", context=PROCESS_UTILITY_TOPLEVEL, params=0x0, 
    queryEnv=0x0, dest=0x223ef68, completionTag=0x7ffe416e6dd0 "") at user_acl.c:110
#10 0x00000000008e5866 in ProcessUtility (pstmt=0x223ee70, queryString=0x223e068 "drop extension user_acl;", context=PROCESS_UTILITY_TOPLEVEL, params=0x0, 
    queryEnv=0x0, dest=0x223ef68, completionTag=0x7ffe416e6dd0 "") at utility.c:356
#11 0x00000000008e48a2 in PortalRunUtility (portal=0x22d2f38, pstmt=0x223ee70, isTopLevel=true, setHoldSnapshot=false, dest=0x223ef68, 
    completionTag=0x7ffe416e6dd0 "") at pquery.c:1175
#12 0x00000000008e4aba in PortalRunMulti (portal=0x22d2f38, isTopLevel=true, setHoldSnapshot=false, dest=0x223ef68, altdest=0x223ef68, 
    completionTag=0x7ffe416e6dd0 "") at pquery.c:1321
#13 0x00000000008e3fff in PortalRun (portal=0x22d2f38, count=9223372036854775807, isTopLevel=true, run_once=true, dest=0x223ef68, altdest=0x223ef68, 
    completionTag=0x7ffe416e6dd0 "") at pquery.c:796
#14 0x00000000008de0d5 in exec_simple_query (query_string=0x223e068 "drop extension user_acl;") at postgres.c:1215
#15 0x00000000008e2256 in PostgresMain (argc=1, argv=0x2269510, dbname=0x2269350 "postgres", username=0x223ad18 "admin") at postgres.c:4247
#16 0x0000000000839272 in BackendRun (port=0x2260cd0) at postmaster.c:4448
#17 0x0000000000838a38 in BackendStartup (port=0x2260cd0) at postmaster.c:4139
#18 0x0000000000834df1 in ServerLoop () at postmaster.c:1704
#19 0x00000000008346c0 in PostmasterMain (argc=3, argv=0x2238c80) at postmaster.c:1377
#20 0x0000000000755bd8 in main (argc=3, argv=0x2238c80) at main.c:228
(gdb) 
```



## 删除插件`user_acl`

删除插件后，插件还可以继续工作，如下：

```sql
postgres=# create extension user_acl;
CREATE EXTENSION
postgres=# drop extension user_acl;
DROP EXTENSION
postgres=# \c postgres testopr
You are now connected to database "postgres" as user "testopr".
postgres=> drop view pg_stat_statements;
ERROR:  OPR user cannot execute ddl statements.
postgres=> 
```

