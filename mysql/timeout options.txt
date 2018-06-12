mysqld will timeout database connections based on two server options:

interactive_timeout(https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_interactive_timeout):交互连接的超时时间。对于交互连接，如果超过这个时间没有新交互，则关闭连接

wait_timeout(https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_wait_timeout)
Both are 28,800 seconds (8 hours) by default.非交互连接。

交互连接：比如在mysql客户端打开了黑窗口。
非交互连接：比如在项目中使用db。本质上，调用mysql_real_connect()中使用CLIENT_INTERACTIVE选项的客户端

You can set these options in /etc/my.cnf

If your connections are persistent (opened via mysql_pconnect) you could lower these numbers to something reasonable like 600 (10 minutes) or even 60 (1 minute). Or, if your app works just fine, you can leave the default. This is up to you.

You must set these as follows in my.cnf (takes effect after mysqld is restarted):

[mysqld]
interactive_timeout=180
wait_timeout=180
If you do not want to restart mysql, then run these two commands:

SET GLOBAL interactive_timeout = 180;
SET GLOBAL wait_timeout = 180;
This will not close the connections already open. This will cause new connections to close in 180 seconds.