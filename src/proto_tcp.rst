=================================================================
proto_tcpの解析
=================================================================

tcp_bind_listener
==================

start_proxiesのlistener->proto->bindから呼ばれる。

動作概要
==========

動作詳細
==========

コードの詳細は以下の通り::

  int tcp_bind_listener(struct listener *listener, char *errmsg, int errlen)
  {
  	__label__ tcp_return, tcp_close_return;
  	int fd, err;
  	int ext, ready;
  	socklen_t ready_len;
  	const char *msg = NULL;
  
  	/* ensure we never return garbage */
  	if (errlen)
  		*errmsg = 0;
  
  	if (listener->state != LI_ASSIGNED)
  		return ERR_NONE; /* already bound */
  
  	err = ERR_NONE;
  
  	/* if the listener already has an fd assigned, then we were offered the
  	 * fd by an external process (most likely the parent), and we don't want
  	 * to create a new socket. However we still want to set a few flags on
  	 * the socket.
  	 */
  	fd = listener->fd;
  	ext = (fd >= 0);
  
  	if (!ext && (fd = socket(listener->addr.ss_family, SOCK_STREAM, IPPROTO_TCP)) == -1) {
  		err |= ERR_RETRYABLE | ERR_ALERT;
  		msg = "cannot create listening socket";
  		goto tcp_return;
  	}

すでにソケットが取得されていない、かつ、socketがエラーを返した場合は、ERR_RETRYABLEと
ERR_ALERTを立てて復帰する(注：socketのエラーってリトライして救えたっけ？)

なお、このソケットはリスナーのソケットである。
::
  
  	if (fd >= global.maxsock) {
  		err |= ERR_FATAL | ERR_ABORT | ERR_ALERT;
  		msg = "not enough free sockets (raise '-n' parameter)";
  		goto tcp_close_return;
  	}



fdの値がglobal.maxsock以上の場合、ERR_FATAL,ERR_ABORT,ERR_ALERTを立てて、
復帰する(注：fdの値ってエラー有無の条件に伝えたっけ？man socketの返り値の説明を
見てもそのようなことは読み取れないのだが・・・

返り値
       成功した場合、新しいソケットのファイル・ディスクリプターを返す。エラ ー
              が発生した場合は -1 を返し、 errno を適切に設定する。

::

  
  	if (fcntl(fd, F_SETFL, O_NONBLOCK) == -1) {
  		err |= ERR_FATAL | ERR_ALERT;
  		msg = "cannot make socket non-blocking";
  		goto tcp_close_return;
  	}

fcntlにF_SETFLを指定し、O_NONBLOCKをソケットに指定する(言い換えれば、O_NONBLOCKを
指定したいがために、F_SETFLを指定する）。

       F_SETFL (long)
              ファイル状態フラグに arg で指定された値を設定する。 arg のうち、
              ファイルのアクセスモード (O_RDONLY, O_WRONLY, O_RDWR) とファイル
              作 成フラグ (すなわち O_CREAT, O_EXCL, O_NOCTTY, O_TRUNC) に関す
              るビットは無視される。 Linux では、このコマンドで変更できるの は
              O_APPEND,  O_ASYNC, O_DIRECT, O_NOATIME, O_NONBLOCK フラグだけで
              ある。

ノンブロッキング、非同期I/Oについては以下の記事が参考になる。
http://d.hatena.ne.jp/cou929_la/20121103/1351950688

なお、上記のブログでは、fcntlでF_SETFLするまえに、現状のフラグ値を取得して、
そこに設定したい内容を立たせている（ビットフラグ）。しかし、HAproxyの
コードを見てみると、事前にgetするようなことはしていない。
ちょっと怪しい感じがする(デフォルト値が消えてしまうプログラミングとなっている。
フラグのデフォルト値が無いというLinuxの仕様であればよいのだが)。
::
  
  	if (!ext && setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one)) == -1) {
  		/* not fatal but should be reported */
  		msg = "cannot do so_reuseaddr";
  		err |= ERR_ALERT;
  	}

次にソケットのオプションの設定を行う。"man setsockopt"によると、"ソケット API 層でオプションを操作"するためには、第２引数にSOL_SOCKETを指定する必要がある。

SO_REUSEADDRは、HAproxy側がソケットをcloseした後に、すぐに同一アドレスで
HAProxyを起動できるようにするためである(グレースフルrebootの実現のため)。
SO_REUSEADDRの解説は以下のブログがわかりやすい。
http://www.geekpage.jp/programming/winsock/so_reuseaddr.php

なお、ポート番号についても同様の効果を許すSO_REUSEPORTはLinux kernel 3.9から
追加された。本当はこれも使う必要があるのかもしれない。と言ってもman 7 socketをみると
SO_REUSEADDRの説明は以下であり、


       SO_REUSEADDR
              bind(2) コールに与えられたアドレスが正しいかを判断するルールで、
              ロ ーカルアドレスの再利用を可能にする。つまり AF_INET ソケットな
              ら、そのアドレスにバインドされたアクティブな listen 状態のソケッ
              トが存在しない限り、バインドが行える。 listen 状態のソケットがア
              ドレス INADDR_ANY で特定のポートにバインドされている場合には、こ
              のポートに対しては、どんなローカルアドレスでもバインドできない。
              引き数はブール整数のフラグである。

HAProxyのグレースフルreboot場合は、listenerのソケットをクローズしてから、新しいHAProxyが
起動することになるので、結局は、SO_REUSEPORTを指定せず、SO_REUSEADDRだけでも十分ということになる。
::
  
  	if (listener->options & LI_O_NOLINGER)
  		setsockopt(fd, SOL_SOCKET, SO_LINGER, &nolinger, sizeof(struct linger));

次にソケットに対して、SO_LINGERを指定している。SO_LINGERの説明はman 7 socketに
よると以下の通り。


       SO_LINGER
              SO_LINGER オプションを取得・設定する。引き数には linger 構造体を
              取る。

                  struct linger {
                      int l_onoff;    /* linger active */
                      int l_linger;   /* how many seconds to linger for */
                  };

              有 効になっていると、 close(2) や shutdown(2) は、そのソケットに
              キューイングされたメッセージがすべて送信完了するか、 linger ( 居
              残り) タイムアウトになるまで返らない。無効になっていると、これら
              のコールはただちに戻り、クローズ動作はバックグラウンドで行われる
              。 ソケットのクローズを exit(2) の一部として行った場合には、残っ
              ているソケットのクローズ動作は必ずバックグラウンドに送られる。

HAProxyで渡しているnolingerは以下であるから、

const struct linger nolinger = { .l_onoff = 1, .l_linger = 0 };

linger自体は有効にしているものの、実質は0秒でexitが復帰することになる。
(待たずにHAProxyのグレースフルrebootが行えるのか？という疑問はある)

::
  
  #ifdef SO_REUSEPORT
  	/* OpenBSD supports this. As it's present in old libc versions of Linux,
  	 * it might return an error that we will silently ignore.
  	 */
  	if (!ext)
  		setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &one, sizeof(one));
  #endif
  
  	if (!ext && (listener->options & LI_O_FOREIGN)) {
  		switch (listener->addr.ss_family) {
  		case AF_INET:
  			if (1
  #if defined(IP_TRANSPARENT)
  			    && (setsockopt(fd, SOL_IP, IP_TRANSPARENT, &one, sizeof(one)) == -1)
  #endif
  #if defined(IP_FREEBIND)
  			    && (setsockopt(fd, SOL_IP, IP_FREEBIND, &one, sizeof(one)) == -1)
  #endif
  #if defined(IP_BINDANY)
  			    && (setsockopt(fd, IPPROTO_IP, IP_BINDANY, &one, sizeof(one)) == -1)
  #endif
  #if defined(SO_BINDANY)
  			    && (setsockopt(fd, SOL_SOCKET, SO_BINDANY, &one, sizeof(one)) == -1)
  #endif
  			    ) {
  				msg = "cannot make listening socket transparent";
  				err |= ERR_ALERT;
  			}
  		break;
  		case AF_INET6:
  			if (1
  #if defined(IPV6_TRANSPARENT)
  			    && (setsockopt(fd, SOL_IPV6, IPV6_TRANSPARENT, &one, sizeof(one)) == -1)
  #endif
  #if defined(IP_FREEBIND)
  			    && (setsockopt(fd, SOL_IP, IP_FREEBIND, &one, sizeof(one)) == -1)
  #endif
  #if defined(IPV6_BINDANY)
  			    && (setsockopt(fd, IPPROTO_IPV6, IPV6_BINDANY, &one, sizeof(one)) == -1)
  #endif
  #if defined(SO_BINDANY)
  			    && (setsockopt(fd, SOL_SOCKET, SO_BINDANY, &one, sizeof(one)) == -1)
  #endif
  			    ) {
  				msg = "cannot make listening socket transparent";
  				err |= ERR_ALERT;
  			}
  		break;
  		}
  	}
  
  #ifdef SO_BINDTODEVICE
  	/* Note: this might fail if not CAP_NET_RAW */
  	if (!ext && listener->interface) {
  		if (setsockopt(fd, SOL_SOCKET, SO_BINDTODEVICE,
  			       listener->interface, strlen(listener->interface) + 1) == -1) {
  			msg = "cannot bind listener to device";
  			err |= ERR_WARN;
  		}
  	}
  #endif
  #if defined(TCP_MAXSEG)
  	if (listener->maxseg > 0) {
  		if (setsockopt(fd, IPPROTO_TCP, TCP_MAXSEG,
  			       &listener->maxseg, sizeof(listener->maxseg)) == -1) {
  			msg = "cannot set MSS";
  			err |= ERR_WARN;
  		}
  	}
  #endif
  #if defined(TCP_DEFER_ACCEPT)
  	if (listener->options & LI_O_DEF_ACCEPT) {
  		/* defer accept by up to one second */
  		int accept_delay = 1;
  		if (setsockopt(fd, IPPROTO_TCP, TCP_DEFER_ACCEPT, &accept_delay, sizeof(accept_delay)) == -1) {
  			msg = "cannot enable DEFER_ACCEPT";
  			err |= ERR_WARN;
  		}
  	}
  #endif
  #if defined(TCP_FASTOPEN)
  	if (listener->options & LI_O_TCP_FO) {
  		/* TFO needs a queue length, let's use the configured backlog */
  		int qlen = listener->backlog ? listener->backlog : listener->maxconn;
  		if (setsockopt(fd, IPPROTO_TCP, TCP_FASTOPEN, &qlen, sizeof(qlen)) == -1) {
  			msg = "cannot enable TCP_FASTOPEN";
  			err |= ERR_WARN;
  		}
  	}
  #endif
  #if defined(IPV6_V6ONLY)
  	if (listener->options & LI_O_V6ONLY)
                  setsockopt(fd, IPPROTO_IPV6, IPV6_V6ONLY, &one, sizeof(one));
  	else if (listener->options & LI_O_V4V6)
                  setsockopt(fd, IPPROTO_IPV6, IPV6_V6ONLY, &zero, sizeof(zero));
  #endif
  
  	if (!ext && bind(fd, (struct sockaddr *)&listener->addr, listener->proto->sock_addrlen) == -1) {
  		err |= ERR_RETRYABLE | ERR_ALERT;
  		msg = "cannot bind socket";
  		goto tcp_close_return;
  	}
  
  	ready = 0;
  	ready_len = sizeof(ready);
  	if (getsockopt(fd, SOL_SOCKET, SO_ACCEPTCONN, &ready, &ready_len) == -1)
  		ready = 0;
  
  	if (!(ext && ready) && /* only listen if not already done by external process */
  	    listen(fd, listener->backlog ? listener->backlog : listener->maxconn) == -1) {
  		err |= ERR_RETRYABLE | ERR_ALERT;
  		msg = "cannot listen to socket";
  		goto tcp_close_return;
  	}
  
  #if defined(TCP_QUICKACK)
  	if (listener->options & LI_O_NOQUICKACK)
  		setsockopt(fd, IPPROTO_TCP, TCP_QUICKACK, &zero, sizeof(zero));
  #endif
  
  	/* the socket is ready */
  	listener->fd = fd;
  	listener->state = LI_LISTEN;
  
  	fdtab[fd].owner = listener; /* reference the listener instead of a task */
  	fdtab[fd].iocb = listener->proto->accept;
  	fd_insert(fd);
  
   tcp_return:
  	if (msg && errlen) {
  		char pn[INET6_ADDRSTRLEN];
  
  		addr_to_str(&listener->addr, pn, sizeof(pn));
  		snprintf(errmsg, errlen, "%s [%s:%d]", msg, pn, get_host_port(&listener->addr));
  	}
  	return err;
  
   tcp_close_return:
  	close(fd);
  	goto tcp_return;
  }
  
  /* This function creates all TCP sockets bound to the protocol entry <proto>.
   * It is intended to be used as the protocol's bind_all() function.
   * The sockets will be registered but not added to any fd_set, in order not to
   * loose them across the fork(). A call to enable_all_listeners() is needed
   * to complete initialization. The return value is composed from ERR_*.
   */
  static int tcp_bind_listeners(struct protocol *proto, char *errmsg, int errlen)
  {
  	struct listener *listener;
  	int err = ERR_NONE;
  
  	list_for_each_entry(listener, &proto->listeners, proto_list) {
  		err |= tcp_bind_listener(listener, errmsg, errlen);
  		if (err & ERR_ABORT)
  			break;
  	}
  
  	return err;
  }






