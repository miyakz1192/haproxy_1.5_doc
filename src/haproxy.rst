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

main関数の解析::

  int main(int argc, char **argv)
  {
  	int err, retry;
  	struct rlimit limit;
  	char errmsg[100];
  	int pidfd = -1;

ここでローカル変数として、rlimit型構造体のインスタンス(limit)を
宣言する。rlimitの詳細は以下。
https://linuxjm.osdn.jp/html/LDP_man-pages/man2/setrlimit.2.html

::
  
  	init(argc, argv);
  	signal_register_fct(SIGQUIT, dump, SIGQUIT);
  	signal_register_fct(SIGUSR1, sig_soft_stop, SIGUSR1);
  	signal_register_fct(SIGHUP, sig_dump_state, SIGHUP);

init関数を実行して初期化と、シグナルハンドラの設定を行う::
  
  	/* Always catch SIGPIPE even on platforms which define MSG_NOSIGNAL.
  	 * Some recent FreeBSD setups report broken pipes, and MSG_NOSIGNAL
  	 * was defined there, so let's stay on the safe side.
  	 */
  	signal_register_fct(SIGPIPE, NULL, 0);

MSG_NOSIGNALを定義しているプラットフォームなら、いつもSIGPIPEを
キャッチする。ある最近のFreeBSDはbroken pipe時にMSG_NOSIGNALをレポートする。なので、安全サイドに倒して、SIGPIPEのハンドラを設定する。::
  
  	/* ulimits */
  	if (!global.rlimit_nofile)
  		global.rlimit_nofile = global.maxsock;

global.rlimit_nofileが0であれば、その値をglobal.maxsockで初期化。::
  
  	if (global.rlimit_nofile) {
  		limit.rlim_cur = limit.rlim_max = global.rlimit_nofile;
  		if (setrlimit(RLIMIT_NOFILE, &limit) == -1) {
  			Warning("[%s.main()] Cannot raise FD limit to %d.\n", argv[0], global.rlimit_nofile);
  		}
  	}
  
  	if (global.rlimit_memmax) {
  		limit.rlim_cur = limit.rlim_max =
  			global.rlimit_memmax * 1048576 / global.nbproc;
  #ifdef RLIMIT_AS
  		if (setrlimit(RLIMIT_AS, &limit) == -1) {
  			Warning("[%s.main()] Cannot fix MEM limit to %d megs.\n",
  				argv[0], global.rlimit_memmax);
  		}
  #else
  		if (setrlimit(RLIMIT_DATA, &limit) == -1) {
  			Warning("[%s.main()] Cannot fix MEM limit to %d megs.\n",
  				argv[0], global.rlimit_memmax);
  		}
  #endif
  	}
  
  	/* We will loop at most 100 times with 10 ms delay each time.
  	 * That's at most 1 second. We only send a signal to old pids
  	 * if we cannot grab at least one port.
  	 */
  	retry = MAX_START_RETRIES;
  	err = ERR_NONE;
  	while (retry >= 0) {
  		struct timeval w;
  		err = start_proxies(retry == 0 || nb_oldpids == 0);
  		/* exit the loop on no error or fatal error */
  		if ((err & (ERR_RETRYABLE|ERR_FATAL)) != ERR_RETRYABLE)
  			break;
  		if (nb_oldpids == 0 || retry == 0)
  			break;
  
  		/* FIXME-20060514: Solaris and OpenBSD do not support shutdown() on
  		 * listening sockets. So on those platforms, it would be wiser to
  		 * simply send SIGUSR1, which will not be undoable.
  		 */
  		if (tell_old_pids(SIGTTOU) == 0) {
  			/* no need to wait if we can't contact old pids */
  			retry = 0;
  			continue;
  		}
  		/* give some time to old processes to stop listening */
  		w.tv_sec = 0;
  		w.tv_usec = 10*1000;
  		select(0, NULL, NULL, NULL, &w);
  		retry--;
  	}
  
  	/* Note: start_proxies() sends an alert when it fails. */
  	if ((err & ~ERR_WARN) != ERR_NONE) {
  		if (retry != MAX_START_RETRIES && nb_oldpids) {
  			protocol_unbind_all(); /* cleanup everything we can */
  			tell_old_pids(SIGTTIN);
  		}
  		exit(1);
  	}
  
  	if (listeners == 0) {
  		Alert("[%s.main()] No enabled listener found (check the <listen> keywords) ! Exiting.\n", argv[0]);
  		/* Note: we don't have to send anything to the old pids because we
  		 * never stopped them. */
  		exit(1);
  	}
  
  	err = protocol_bind_all(errmsg, sizeof(errmsg));
  	if ((err & ~ERR_WARN) != ERR_NONE) {
  		if ((err & ERR_ALERT) || (err & ERR_WARN))
  			Alert("[%s.main()] %s.\n", argv[0], errmsg);
  
  		Alert("[%s.main()] Some protocols failed to start their listeners! Exiting.\n", argv[0]);
  		protocol_unbind_all(); /* cleanup everything we can */
  		if (nb_oldpids)
  			tell_old_pids(SIGTTIN);
  		exit(1);
  	} else if (err & ERR_WARN) {
  		Alert("[%s.main()] %s.\n", argv[0], errmsg);
  	}
  
  	/* prepare pause/play signals */
  	signal_register_fct(SIGTTOU, sig_pause, SIGTTOU);
  	signal_register_fct(SIGTTIN, sig_listen, SIGTTIN);
  
  	/* MODE_QUIET can inhibit alerts and warnings below this line */
  
  	global.mode &= ~MODE_STARTING;
  	if ((global.mode & MODE_QUIET) && !(global.mode & MODE_VERBOSE)) {
  		/* detach from the tty */
  		fclose(stdin); fclose(stdout); fclose(stderr);
  	}
  
  	/* open log & pid files before the chroot */
  	if (global.mode & (MODE_DAEMON | MODE_SYSTEMD) && global.pidfile != NULL) {
  		unlink(global.pidfile);
  		pidfd = open(global.pidfile, O_CREAT | O_WRONLY | O_TRUNC, 0644);
  		if (pidfd < 0) {
  			Alert("[%s.main()] Cannot create pidfile %s\n", argv[0], global.pidfile);
  			if (nb_oldpids)
  				tell_old_pids(SIGTTIN);
  			protocol_unbind_all();
  			exit(1);
  		}
  	}
  
  #ifdef CONFIG_HAP_CTTPROXY
  	if (global.last_checks & LSTCHK_CTTPROXY) {
  		int ret;
  
  		ret = check_cttproxy_version();
  		if (ret < 0) {
  			Alert("[%s.main()] Cannot enable cttproxy.\n%s",
  			      argv[0],
  			      (ret == -1) ? "  Incorrect module version.\n"
  			      : "  Make sure you have enough permissions and that the module is loaded.\n");
  			protocol_unbind_all();
  			exit(1);
  		}
  	}
  #endif
  
  	if ((global.last_checks & LSTCHK_NETADM) && global.uid) {
  		Alert("[%s.main()] Some configuration options require full privileges, so global.uid cannot be changed.\n"
  		      "", argv[0]);
  		protocol_unbind_all();
  		exit(1);
  	}
  
  	/* If the user is not root, we'll still let him try the configuration
  	 * but we inform him that unexpected behaviour may occur.
  	 */
  	if ((global.last_checks & LSTCHK_NETADM) && getuid())
  		Warning("[%s.main()] Some options which require full privileges"
  			" might not work well.\n"
  			"", argv[0]);
  
  	/* chroot if needed */
  	if (global.chroot != NULL) {
  		if (chroot(global.chroot) == -1 || chdir("/") == -1) {
  			Alert("[%s.main()] Cannot chroot(%s).\n", argv[0], global.chroot);
  			if (nb_oldpids)
  				tell_old_pids(SIGTTIN);
  			protocol_unbind_all();
  			exit(1);
  		}
  	}
  
  	if (nb_oldpids)
  		nb_oldpids = tell_old_pids(oldpids_sig);
  
  	/* Note that any error at this stage will be fatal because we will not
  	 * be able to restart the old pids.
  	 */
  
  	/* setgid / setuid */
  	if (global.gid) {
  		if (getgroups(0, NULL) > 0 && setgroups(0, NULL) == -1)
  			Warning("[%s.main()] Failed to drop supplementary groups. Using 'gid'/'group'"
  				" without 'uid'/'user' is generally useless.\n", argv[0]);
  
  		if (setgid(global.gid) == -1) {
  			Alert("[%s.main()] Cannot set gid %d.\n", argv[0], global.gid);
  			protocol_unbind_all();
  			exit(1);
  		}
  	}
  
  	if (global.uid && setuid(global.uid) == -1) {
  		Alert("[%s.main()] Cannot set uid %d.\n", argv[0], global.uid);
  		protocol_unbind_all();
  		exit(1);
  	}
  
  	/* check ulimits */
  	limit.rlim_cur = limit.rlim_max = 0;
  	getrlimit(RLIMIT_NOFILE, &limit);
  	if (limit.rlim_cur < global.maxsock) {
  		Warning("[%s.main()] FD limit (%d) too low for maxconn=%d/maxsock=%d. Please raise 'ulimit-n' to %d or more to avoid any trouble.\n",
  			argv[0], (int)limit.rlim_cur, global.maxconn, global.maxsock, global.maxsock);
  	}
  
  	if (global.mode & (MODE_DAEMON | MODE_SYSTEMD)) {
  		struct proxy *px;
  		struct peers *curpeers;
  		int ret = 0;
  		int *children = calloc(global.nbproc, sizeof(int));
  		int proc;
  
  		/* the father launches the required number of processes */
  		for (proc = 0; proc < global.nbproc; proc++) {
  			ret = fork();
  			if (ret < 0) {
  				Alert("[%s.main()] Cannot fork.\n", argv[0]);
  				protocol_unbind_all();
  				exit(1); /* there has been an error */
  			}
  			else if (ret == 0) /* child breaks here */
  				break;
  			children[proc] = ret;
  			if (pidfd >= 0) {
  				char pidstr[100];
  				snprintf(pidstr, sizeof(pidstr), "%d\n", ret);
  				shut_your_big_mouth_gcc(write(pidfd, pidstr, strlen(pidstr)));
  			}
  			relative_pid++; /* each child will get a different one */
  		}
  
  #ifdef USE_CPU_AFFINITY
  		if (proc < global.nbproc &&  /* child */
  		    proc < LONGBITS &&       /* only the first 32/64 processes may be pinned */
  		    global.cpu_map[proc])    /* only do this if the process has a CPU map */
  			sched_setaffinity(0, sizeof(unsigned long), (void *)&global.cpu_map[proc]);
  #endif
  		/* close the pidfile both in children and father */
  		if (pidfd >= 0) {
  			//lseek(pidfd, 0, SEEK_SET);  /* debug: emulate eglibc bug */
  			close(pidfd);
  		}
  
  		/* We won't ever use this anymore */
  		free(oldpids);        oldpids = NULL;
  		free(global.chroot);  global.chroot = NULL;
  		free(global.pidfile); global.pidfile = NULL;
  
  		if (proc == global.nbproc) {
  			if (global.mode & MODE_SYSTEMD) {
  				protocol_unbind_all();
  				for (proc = 0; proc < global.nbproc; proc++)
  					while (waitpid(children[proc], NULL, 0) == -1 && errno == EINTR);
  			}
  			exit(0); /* parent must leave */
  		}
  
  		/* we might have to unbind some proxies from some processes */
  		px = proxy;
  		while (px != NULL) {
  			if (px->bind_proc && px->state != PR_STSTOPPED) {
  				if (!(px->bind_proc & (1UL << proc)))
  					stop_proxy(px);
  			}
  			px = px->next;
  		}
  
  		/* we might have to unbind some peers sections from some processes */
  		for (curpeers = peers; curpeers; curpeers = curpeers->next) {
  			if (!curpeers->peers_fe)
  				continue;
  
  			if (curpeers->peers_fe->bind_proc & (1UL << proc))
  				continue;
  
  			stop_proxy(curpeers->peers_fe);
  			/* disable this peer section so that it kills itself */
  			curpeers->peers_fe = NULL;
  		}
  
  		free(children);
  		children = NULL;
  		/* if we're NOT in QUIET mode, we should now close the 3 first FDs to ensure
  		 * that we can detach from the TTY. We MUST NOT do it in other cases since
  		 * it would have already be done, and 0-2 would have been affected to listening
  		 * sockets
  		 */
  		if (!(global.mode & MODE_QUIET) || (global.mode & MODE_VERBOSE)) {
  			/* detach from the tty */
  			fclose(stdin); fclose(stdout); fclose(stderr);
  			global.mode &= ~MODE_VERBOSE;
  			global.mode |= MODE_QUIET; /* ensure that we won't say anything from now */
  		}
  		pid = getpid(); /* update child's pid */
  		setsid();
  		fork_poller();
  	}
  
  	protocol_enable_all();
  	/*
  	 * That's it : the central polling loop. Run until we stop.
  	 */
  	run_poll_loop();
  
  	/* Free all Hash Keys and all Hash elements */
  	appsession_cleanup();
  	/* Do some cleanup */ 
  	deinit();
      
  	exit(0);
  }
  



init関数
=========

global構造体の初期化タイミング
==================================

以下の通り、mainが始まった段階である程度初期化されている
メンバーがある(例：chksize)。これはデフォルト値で、
haproxy.cのglobal構造体の定義時に構造体の各メンバに
代入されている値である。::


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
  
