# 数据库 URL pattern

|DataBase|UrlPattern|URLs|
|--|--|--|
|MySQL|||
|PostgreSQL|||
|Oracle|||
|SQLServer|||
|MsSQL|||
|h2|||
|Derby|  1、`jdbc:derby:[subprotocol:][databaseName][;attribute=value]*` <br/> 2、`jdbc:derby://server[:port]/databaseName[;attribute=value]*`|jdbc:derby:directory:mydb<br/> jdbc:derby:memory:mydb;create=true<br/>jdbc:derby:memory:C:\test\mydb<br/>jdbc:derby:memory:/home/test/mydb<br/>jdbc:derby:classpath:/test/mydb<br/>jdbc:derby:jar:(C:/dbs.jar)test/mydb<br/>jdbc:derby:mydb<br/>jdbc:derby:/test/mydb;create=true;create=true<br/>jdbc:derby:test/mydb;create=true<br/>jdbc:derby:A:/test/mydb;create=true<br/>jdbc:derby:test/mydb;create=true<br/>jdbc:derby://localhost:1527/mydb;create=true;user=root;password=pass<br/>jdbc:derby://localhost:1527/mydb<br/>jdbc:derby://localhost/mydb;create=true<br/>jdbc:derby://localhost:1527/c:\temp\mydb<br/>jdbc:derby://localhost:1527//opt/test/mydb;create=true<br/>jdbc:derby://localhost:1527/memory:mydb;create=true<br/>jdbc:derby://localhost:1527/memory:C:\test\mydbmydb;create=true<br/>jdbc:derby://localhost:1527/memory:/test/mydb;create=true|
|Sqlite|`jdbc:sqlite:[subprotocol:][path/][databaseName][?attribute=value]*`|jdbc:sqlite:C:/work/mydatabase.db<br/>jdbc:sqlite:/home/leo/work/mydatabase.db<br/>jdbc:sqlite::memory:<br/>jdbc:sqlite::resource:org/yourdomain/sample.db<br/>jdbc:sqlite::resource:http://www.xerial.org/svn/project/XerialJ/trunk/sqlite-jdbc/src/test/java/org/sqlite/sample.db<br/>jdbc:sqlite::resource:jar:http://www.xerial.org/svn/project/XerialJ/trunk/sqlite-jdbc/src/test/resources/testdb.jar!/sample.db<br/>jdbc:sqlite:db.sqlite?hexkey_mode=sse<br/>jdbc:sqlite::memory:?jdbc.explicit_readonly=true<br/>jdbc:sqlite:sample.db|
|DB2|1、`jdbc:db2:databasename`<br/> 2、`jdbc:db2://[hostname]:[port]/[database_name]:user=[username];password=[password]`|jdbc:db2://localhost:50000/mydb:user=root;password=pass<br/>jdbc:db2:mydb:user=root;password=pass<br/>jdbc:db2://clusterip:50000/mydb|
|Sybase|`jdbc:sybase:Tds:host:port[/database][?connection_property=value;]`|jdbc:sybase:Tds:localhost:5000/testdb?charset=utf-8<br/>jdbc:sybase:Tds:localhost:5000?USER=sa&PASSWORD=secret|
|OceanBase|`jdbc:oceanbase:[hamode:]//host:port/databasename?[username&password]&[opt1=val1&opt2=val2...`]|jdbc:oceanbase://localhost:2881/mydb?user=root@sys&password=pass&pool=false&useBulkStmts=true&rewriteBatchedStatements=false&useServerPrepStmts=true<br/>jdbc:oceanbase://primaryhost:2888,secondaryhost1,secondaryhost2/mydb?user=root@sys&password=pass&pool=false&useBulkStmts=true&rewriteBatchedStatements=false&useServerPrepStmts=true<br/>jdbc:oceanbase:hamode://clusterip:2881/test?user=root@sys&password=pass&pool=false&useBulkStmts=true&rewriteBatchedStatements=false&useServerPrepStmts=true|
|TiDB|||
|LevelDB|||
