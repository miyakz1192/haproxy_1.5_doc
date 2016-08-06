============================================================
proxy.cの解析
============================================================

概要
=====

このモジュールはhaproxy.cから呼ばれるhaproxyの中核をなす関数が入っているっぽい。

start_proxies
=================

proxyを起動する関数::


  /*
   * This function creates all proxy sockets. It should be done very early,
   * typically before privileges are dropped. The sockets will be registered
   * but not added to any fd_set, in order not to loose them across the fork().
   * The proxies also start in READY state because they all have their listeners
   * bound.
   *
   * Its return value is composed from ERR_NONE, ERR_RETRYABLE and ERR_FATAL.
   * Retryable errors will only be printed if <verbose> is not zero.
   */
  int start_proxies(int verbose)
  {
  	struct proxy *curproxy;
  	struct listener *listener;
  	int lerr, err = ERR_NONE;
  	int pxerr;
  	char msg[100];
  
  	for (curproxy = proxy; curproxy != NULL; curproxy = curproxy->next) {
  		if (curproxy->state != PR_STNEW)
  			continue; /* already initialized */

まず、グローバル変数のproxy(haproxyの本丸)にcurproxyを合わせる。状態がPR_STNEW(未初期化状態)以外の場合は初期化済みとみなして、continueする。::
  
  		pxerr = 0;
  		list_for_each_entry(listener, &curproxy->conf.listeners, by_fe) {
  			if (listener->state != LI_ASSIGNED)
  				continue; /* already started */
  
  			lerr = listener->proto->bind(listener, msg, sizeof(msg));

ココが何が行われるのかがよくわからないが、listenerとproto(このlistenerが属しているprotocol)の関連付けが行われる。仮想関数になっているので、実際の動作をさせてみて確認したほうが早い。確認結果は詳細1を参照。例えば、bindはtcp_bind_listenersになる::
  
  			/* errors are reported if <verbose> is set or if they are fatal */
  			if (verbose || (lerr & (ERR_FATAL | ERR_ABORT))) {
  				if (lerr & ERR_ALERT)
  					Alert("Starting %s %s: %s\n",
  					      proxy_type_str(curproxy), curproxy->id, msg);
  				else if (lerr & ERR_WARN)
  					Warning("Starting %s %s: %s\n",
  						proxy_type_str(curproxy), curproxy->id, msg);
  			}

エラーが発生した場合はメッセージを表示して、処理自体は継続する。::
  
  			err |= lerr;
  			if (lerr & (ERR_ABORT | ERR_FATAL)) {
  				pxerr |= 1;
  				break;
  			}
  			else if (lerr & ERR_CODE) {
  				pxerr |= 1;
  				continue;
  			}

ERR_ABORTとERR_FATALの場合は処理を中断して、それ以外でERR_CODEがある場合は、
pxerrフラグを立てて処理を継続.ERR_CODEの定義は以下なので、::

  #define ERR_CODE	(ERR_RETRYABLE|ERR_FATAL|ERR_ABORT)	/* mask */

まずは、最初に、ERR_ABORTまたはERR_FATALとやばそうなエラーが発生してないかを判定してから、
ERR_RETRYABLEな場合を判定している::

  		}
  
  		if (!pxerr) {
  			curproxy->state = PR_STREADY;
  			send_log(curproxy, LOG_NOTICE, "Proxy %s started.\n", curproxy->id);
  		}

エラーが発生していなければ、proxyの状態をPR_STREADYにして、proxyが起動した旨のログメッセージを出す::
  
  		if (err & ERR_ABORT)
  			break;
  	}
  
  	return err;
  }
  

詳細1
======

lister->protoのダンプ例は以下::

  (gdb) p *listener->proto
  $6 = {name = "tcpv4\000\000\000\000\000\000\000\000\000\000", sock_domain = 2, 
    sock_type = 1, sock_prot = 6, sock_family = 2, sock_addrlen = 16, 
    l3_addrlen = 4, accept = 0x414054 <listener_accept>, 
    bind = 0x48bfae <tcp_bind_listener>, 
    bind_all = 0x48c540 <tcp_bind_listeners>, 
    unbind_all = 0x413f90 <unbind_all_listeners>, 
    enable_all = 0x413de4 <enable_all_listeners>, disable_all = 0, 
    connect = 0x48af3f <tcp_connect_server>, get_src = 0x48bb8e <tcp_get_src>, 
    get_dst = 0x48bbd7 <tcp_get_dst>, drain = 0x48bc6b <tcp_drain>, 
    pause = 0x48c6af <tcp_pause_listener>, listeners = {n = 0x70b350, 
      p = 0x70b350}, nb_listeners = 1, list = {n = 0x6e4f58, p = 0x6e1a20}}
  (gdb) 
  
ここで、bind(tcp_bind_listeners)の定義は(src/proto_tcp.c)にある。tcp_bind_listenersの
第一引数に渡すlistenerから、アドレスファミリ、バインド対象のアドレスの情報を取得し、
bind/listenシステムコールを実行する。





  
