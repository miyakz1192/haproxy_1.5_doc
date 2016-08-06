==========================================================
解析結果サマリ＆メモ
==========================================================

大事な流れ
===========

main@haproxy.c

シグナルハンドラの設定
  - もうちょっと細かく

rlimit関連の設定
  - もうちょっと細かく

start_proxiesを実行してlistenerに属しているprotocolをbindする処理を行う。
(src/proto_tcp.rst)


proxy(グローバル変数)がhaproxyの本体(proxyはリスト構造)。proxy毎に、listenerが存在する。また、listenerにはprotocol listenerが存在する。

proxyの状態
================

proxy.hから。::

  /* values for proxy->state */
  enum pr_state {
  	PR_STNEW = 0,           /* proxy has not been initialized yet */
  	PR_STREADY,             /* proxy has been initialized and is ready */
  	PR_STFULL,              /* frontend is full (maxconn reached) */
  	PR_STPAUSED,            /* frontend is paused (during hot restart) */
  	PR_STSTOPPED,           /* proxy is stopped (end of a restart) */
  	PR_STERROR,             /* proxy experienced an unrecoverable error */
  } __attribute__((packed));


listenerの状態遷移
===================

以下の通り::


  /* listener state */
  enum li_state {
  	LI_NEW	= 0,    /* not initialized yet */
  	LI_INIT,        /* all parameters filled in, but not assigned yet */
  	LI_ASSIGNED,    /* assigned to the protocol, but not listening yet */
  	LI_PAUSED,      /* listener was paused, it's bound but not listening  */
  	LI_LISTEN,      /* started, listening but not enabled */
  	LI_READY,       /* started, listening and enabled */
  	LI_FULL,        /* reached its connection limit */
  	LI_LIMITED,     /* transient state: limits have been reached, listener is queued */
  } __attribute__((packed));
  
  /* Listener transitions
   * calloc()     set()      add_listener()       bind()
   * -------> NEW ----> INIT ----------> ASSIGNED -----> LISTEN
   * <-------     <----      <----------          <-----
   *    free()   bzero()     del_listener()       unbind()
   *
   * The file descriptor is valid only during these three states :
   *
   *             disable()
   * LISTEN <------------ READY
   *   A|   ------------>  |A
   *   ||  !max & enable() ||
   *   ||                  ||
   *   ||              max ||
   *   || max & enable()   V| !max
   *   |+---------------> FULL
   *   +-----------------
   *            disable()
   *
   * The LIMITED state my be used when a limit has been detected just before
   * using a listener. In this case, the listener MUST be queued into the
   * appropriate wait queue (either the proxy's or the global one). It may be
   * set back to the READY state at any instant and for any reason, so one must
   * not rely on this state.
   */



メモ
====
以下のグローバル変数の意味、初期値、初期化タイミングがわからない
peers
fdtab
cur_poller








