============================================================
haproxy.cの解析
============================================================

グローバルスコープのデータ構造
===============================

変数
=====

+-------------+-------+-----------+------------------------+
|メンバ       |型     |def値      |意味                    |
+=============+=======+===========+========================+
|cfg_cfgfiles |list   |LIST_HEAD_ |?                       |
|             |       |INIT       |                        |
+-------------+-------+-----------+------------------------+
|pid          |int    |なし       |haproxyのpid            |
+-------------+-------+-----------+------------------------+
|relative_pid |int    |1          |                        |
+-------------+-------+-----------+------------------------+
|stopping     |int    |なし       |non zero means stopping |
|             |       |           |in progress             |
+-------------+-------+-----------+------------------------+
|jobs         |int    |0          |number of active jobs   |
|             |       |           |(conns, listeners       |
|             |       |           |, active tasks, ...)    |
+-------------+-------+-----------+------------------------+
|trush        |chunk  |{}         |this is used to drain   |
|             |       |           |data, and as a temporary|
|             |       |           |buffer for sprintf      |
+-------------+-------+-----------+------------------------+
|swap_buffer  |char*  |NULL       |this buffer is always   |  
|             |       |           |the same size as        |
|             |       |           |standard buffers and    |
|             |       |           |its used for swapping   |
|             |       |           |data inside a buffer    |
+-------------+-------+-----------+------------------------+
|nb_oldpids   |int    |0          |                        |
+-------------+-------+-----------+------------------------+
|zero         |int    |0          |                        |
+-------------+-------+-----------+------------------------+
|one          |int    |1          |                        |
+-------------+-------+-----------+------------------------+
|nolinger     |linger |{.l_onoff=1|                        |
|             |       |,          |                        |
|             |       |.l_linger=0|                        |
|             |       |}          |                        |
+-------------+-------+-----------+------------------------+
|hostname     |char   |無し       |MAX_HOSTNAME_LEN        |
+-------------+-------+-----------+------------------------+
|localpeer    |char   |無し       |MAX_HOSTNAME_LEN        |
+-------------+-------+-----------+------------------------+
|shut_your_   |int    |0          |used from everywhere    |
|big_mouth_   |       |           |just to drain results   |
|gcc_int      |       |           |we don't want to        |
|             |       |           |read and which          |
|             |       |           |recent versions of gcc  |
|             |       |           |increasingly and        |
|             |       |           |annoyingly complain     |
|             |       |           |about                   |
+-------------+-------+-----------+------------------------+
|global_      |list   |LIST_HEAD_ |list of the temporarily |
|listener_    |       |INIT       |limited listeners       |
|queue        |       |           |because of lack         |
|             |       |           |of resource             |
+-------------+-------+-----------+------------------------+
|global_      |task*  |           |                        |
|listener_    |       |           |                        |
|queue_task   |       |           |                        |
+-------------+-------+-----------+------------------------+
|manager_     |task*  |           |                        |
|global_      |       |           |                        |
|listener_    |       |           |                        |
|queue        |       |           |                        |
+-------------+-------+-----------+------------------------+
|warned       |uint   |0          |                        |
+-------------+-------+-----------+------------------------+
|             |       |           |                        |
+-------------+-------+-----------+------------------------+

プロセス制御系の変数
/* Here we store informations about the pids of the processes we may pause
 * or kill. We will send them a signal every 10 ms until we can bind to all
 * our ports. With 200 retries, that's about 2 seconds.
 */
+-------------+-------+-----------+------------------------+
|メンバ       |型     |def値      |意味                    |
+=============+=======+===========+========================+
|oldpids      |int*   |NULL       |                        |
+-------------+-------+-----------+------------------------+
|oldpids_sig  |int    |無し       |                        |
+-------------+-------+-----------+------------------------+




global構造体
-------------

構造体直下

+-------------+-------+-----------+------------------------+
|メンバ       |型     |def値      |意味                    |
+=============+=======+===========+========================+
|nbproc       |int?   |1          |?                       |
+-------------+-------+-----------+------------------------+
|req_count    |int?   |0          |?                       |
+-------------+-------+-----------+------------------------+
|logsrvs      |list?  |LIST_HEAD_ |                        |
|             |       |INIT       |                        |
+-------------+-------+-----------+------------------------+
|maxzlibmem   |int?   |値1        |?                       |
+-------------+-------+-----------+------------------------+
|comp_rate_lim|int?   |0          |?                       |
+-------------+-------+-----------+------------------------+
|ssl_server_  |int?   |SSL_SERVER_|                        |
|verify       |       |VERIFY_    |                        |
|             |       |REQUIRED   |                        |
+-------------+-------+-----------+------------------------+
|maxsslconn   |int?   |DEFAULT_   |                        |
|             |       |MAXSSLCONN |                        |
+-------------+-------+-----------+------------------------+
|             |       |           |                        |
+-------------+-------+-----------+------------------------+

値1:DEFAULT_MAXZLIBMEM * 1024U * 1024U,

注意。以下のメンバはコンパイルオプションによって、デフォルト値が異なる。::

  #ifdef DEFAULT_MAXZLIBMEM
  	.maxzlibmem = DEFAULT_MAXZLIBMEM * 1024U * 1024U,
  #else
  	.maxzlibmem = 0,
  #endif


unix_bind直下
+-------------+-------+-----------+------------------------+
|メンバ       |型     |def値      |意味                    |
+=============+=======+===========+========================+
|uid          |int?   |           |                        |
+-------------+-------+-----------+------------------------+
|gid          |int?   |           |                        |
+-------------+-------+-----------+------------------------+
|mode         |int?   |           |                        |
+-------------+-------+-----------+------------------------+

tune配下
+-------------+-------+-----------+------------------------+
|メンバ       |型     |def値      |意味                    |
+=============+=======+===========+========================+
|bufsize      |       |BUFSIZE    |                        |
+-------------+-------+-----------+------------------------+
|maxrewrite   |       |BUFSIZE    |                        |
+-------------+-------+-----------+------------------------+
|chksize      |       |BUFSIZE    |                        |
+-------------+-------+-----------+------------------------+
|sslcachesize |       |SSLCACHE   |                        |
|             |       |SIZE       |                        |
+-------------+-------+-----------+------------------------+
|ssl_max_     |       |DEFAULT_SSL|                        |
|record       |       |MAX_RECORD |                        |
+-------------+-------+-----------+------------------------+
|zlibmemlevel |       |8          |                        |
+-------------+-------+-----------+------------------------+
|zlibwindow   |       |MAX_WBITS  |                        |
|size         |       |           |                        |
+-------------+-------+-----------+------------------------+
|comp_maxlevel|       |1          |                        |
+-------------+-------+-----------+------------------------+
|idle_timer   |       |DEFAULT_   |                        |
|             |       |IDLE_TIMER |                        |
+-------------+-------+-----------+------------------------+
|idle_timer   |       |1000       |                        |
+-------------+-------+-----------+------------------------+
|             |       |           |                        |
+-------------+-------+-----------+------------------------+


main関数
==========


init関数
=========

global構造体の初期化タイミング
==================================

以下の通り、mainが始まった段階である程度初期化されている
メンバーがある(例：chksize)。これはデフォルト値で、
haproxy.cのglobal構造体の定義時に構造体の各メンバに
代入されている値である。


  [root@chefserver haproxy-1.5]# gdb ./haproxy 
  GNU gdb (GDB) Red Hat Enterprise Linux (7.2-60.el6_4.1)
  Copyright (C) 2010 Free Software Foundation, Inc.
  License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
  This is free software: you are free to change and redistribute it.
  There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
  and "show warranty" for details.
  This GDB was configured as "x86_64-redhat-linux-gnu".
  For bug reporting instructions, please see:
  <http://www.gnu.org/software/gdb/bugs/>...
  Reading symbols from /root/git_source/haproxy-1.5/haproxy...done.
  (gdb) b haproxy.c:1349
  Breakpoint 1 at 0x40487b: file src/haproxy.c, line 1349.
  (gdb) run -f haproxy.config 
  Starting program: /root/git_source/haproxy-1.5/haproxy -f haproxy.config
  
  Breakpoint 1, main (argc=3, argv=0x7fffffffe448) at src/haproxy.c:1349
  warning: Source file is more recent than executable.
  1349    printf("miyakz DEBUG:  start\n");
  Missing separate debuginfos, use: debuginfo-install glibc-2.12-1.132.el6.x86_64 nss-softokn-freebl-3.14.3-9.el6.x86_64
  (gdb) l
  1344    int err, retry;
  1345    struct rlimit limit;
  1346    char errmsg[100];
  1347    int pidfd = -1;
  1348  
  1349    printf("miyakz DEBUG:  start\n");
  1350  
  1351    init(argc, argv);
  1352    signal_register_fct(SIGQUIT, dump, SIGQUIT);
  1353    signal_register_fct(SIGUSR1, sig_soft_stop, SIGUSR1);
  (gdb) p global
  $1 = {uid = 0, gid = 0, nbproc = 1, maxconn = 0, hardmaxconn = 0, 
    ssl_server_verify = 1, conn_per_sec = {curr_sec = 0, curr_ctr = 0, 
      prev_ctr = 0}, sess_per_sec = {curr_sec = 0, curr_ctr = 0, prev_ctr = 0}, 
    ssl_per_sec = {curr_sec = 0, curr_ctr = 0, prev_ctr = 0}, 
    ssl_fe_keys_per_sec = {curr_sec = 0, curr_ctr = 0, prev_ctr = 0}, 
    ssl_be_keys_per_sec = {curr_sec = 0, curr_ctr = 0, prev_ctr = 0}, 
    comp_bps_in = {curr_sec = 0, curr_ctr = 0, prev_ctr = 0}, comp_bps_out = {
      curr_sec = 0, curr_ctr = 0, prev_ctr = 0}, cps_lim = 0, cps_max = 0, 
    sps_lim = 0, sps_max = 0, ssl_lim = 0, ssl_max = 0, ssl_fe_keys_max = 0, 
    ssl_be_keys_max = 0, shctx_lookups = 0, shctx_misses = 0, comp_rate_lim = 0, 
    maxpipes = 0, maxsock = 0, rlimit_nofile = 0, rlimit_memmax = 0, 
    maxzlibmem = 0, mode = 0, req_count = 0, last_checks = 0, spread_checks = 0, 
    max_spread_checks = 0, max_syslog_len = 0, chroot = 0x0, pidfile = 0x0, 
    node = 0x0, desc = 0x0, log_tag = 0x0, logsrvs = {n = 0x69a770, 
      p = 0x69a770}, log_send_hostname = 0x0, tune = {maxpollevents = 0, 
      maxaccept = 0, options = 0, recv_enough = 0, bufsize = 16384, 
      maxrewrite = 8192, client_sndbuf = 0, client_rcvbuf = 0, 
      server_sndbuf = 0, server_rcvbuf = 0, chksize = 16384, pipesize = 0, 
      max_http_hdr = 0, cookie_len = 0, comp_maxlevel = 1, idle_timer = 1000}, 
    unix_bind = {prefix = 0x0, ux = {uid = 4294967295, gid = 4294967295, 
        mode = 0}}, cpu_map = {0 <repeats 64 times>}, stats_fe = 0x0}
  (gdb) 
  
