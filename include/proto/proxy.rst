==================================================================
proxy.hの解析
==================================================================

proxyの状態を表す定義
==========================

構造体は以下::

  /* values for proxy->state */
  enum pr_state {
  	PR_STNEW = 0,           /* proxy has not been initialized yet */
  	PR_STREADY,             /* proxy has been initialized and is ready */
  	PR_STFULL,              /* frontend is full (maxconn reached) */
  	PR_STPAUSED,            /* frontend is paused (during hot restart) */
  	PR_STSTOPPED,           /* proxy is stopped (end of a restart) */
  	PR_STERROR,             /* proxy experienced an unrecoverable error */
  } __attribute__((packed));
  
  
