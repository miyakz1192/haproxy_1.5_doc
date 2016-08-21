=================================================================
proto_tcpの解析
=================================================================

tcp_bind_listener
==================

start_proxiesのlistener->proto->bindから呼ばれる。

動作概要
==========

1) listenerのソケットを作る
2) O_NONBLOCKに設定する
3) SO_REUSEADDRを設定する(グレースフルrebootのため)
4) SO_LINGER(ソケットclose時に待たないようにするため)
5) IP_TRANSPARENTを指定し、透過型proxyとして動作
6) IP_FREEBINDを指定し、net interfaceに存在しないIP/PORTでbind可能にする
7) SO_BINDTODEVICEでバインド先ローカルnet interfaceを指定
8) TCP_MAXSEG でTCPセグメントサイズを必要に応じて設定
9) TCP_DEFER_ACCEPTを設定し、サーバのlistenからacceptまでの動作速度を改善(3WHのlast ACKを待たずに処理開始)
10) TCP_FASTOPENを設定し、3WHのパケットを減らす
11) TCP_QUICKACKを指定して、即座にACKを返すようにする。

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

さらに、ここでsetsockoptを使ってソケットオプションの設定を行っている。
IP_TRANSPARENTのman 7 ipの説明を見ると以下。

  IP_TRANSPARENT (Linux 2.6.24 以降)
  このブール値のオプションを有効にすると、 このソケットで透過プロキシ (transparent proxy) ができるようになる。 このソケットオプションを使うと、呼び出したアプリケーションは、 ローカルではない IP アドレスをバインドして、ローカルの端点として自分以外のアドレス (foreign address) を持つクライアントやサーバの両方として動作できるようになる。 注意: この機能が動作するためには、自分以外のアドレス宛のパケットが透過プロキシが動作するマシン (すなわちソケットオプション IP_TRANSPARENT を利用するアプリケーションが動作しているシステム) 経由で転送されるように、 ルーティングが設定される必要がある。 このソケットオプションを有効にするには、スーパーユーザー特権 (CAP_NET_ADMIN ケーパビリティ) が必要である。
  iptables の TPROXY ターゲットで透過プロキシリダイレクション (TProxy redirection) を行うには、リダイレクトされるソケットに対して このオプションを設定する必要がある。

透過型proxy自体の説明は以下が参考になる
https://tech.jstream.jp/blog/cache/transparent_proxy/

また、IP_FREEBINDについての説明は以下。

  IP_FREEBIND (Linux 2.4 以降)
  このブール値のオプションを有効にすると、ローカルではない IP アドレスや存在 しない IP アドレスをバインドできるようになる。これを使うと、対応するネット ワークインターフェイスがなかったり、アプリケーションがソケットをバインドしようと する時点で特定の動的 IP アドレスが有効になっていなかったりしても、ソケットを 接続待ち状態 (listening) にできるようになる。 このオプションは、下記に説明がある ip_nonlocal_bind /proc インターフェイス のソケット単位の設定である。

IP_FREEBINDとIP_TRANSPARENTはセットで使うのであろう。

★ 注意★  IP_FREEBINDとIP_TRANSPARENTはCentOS6.5のman 7 ipには載っていなかった。
おそらく、未サポートなのだろう。CentOS系でHAProxyを使って、途中で
バージョンを上げる場合は非互換が起きるのでチェックが必要かもしれない。

IP_BINDANY/SO_BINDANYはどうも、FreeBSDのオプションらしい。

::


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

IPv4と同様のことを、IPv6で行っているだけ。
::
  
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

man 7 socketによると説明は以下。

  SO_BINDTODEVICE
  このソケットを、引き数で渡したインターフェース名で指定される ("eth0" のような) 特定のデバイスにバインドする。 名前が空文字列だったり、オプションの長さ (optlen) が 0 の場合には、 ソケットのバインドが削除される。 渡すオプションは、インターフェース名が 入ったヌル文字で終端された可変長の文字列である。 文字列の最大のサイズは IFNAMSIX である。 ソケットがインターフェースにバインドされると、 その特定のインターフェースから受信されたパケットだけを処理する。 このオプションはいくつかのソケットタイプ、 特に AF_INET に対してのみ動作する点に注意すること。 パケットソケットではサポートされていない (通常の bind(2) を使うこと)。
  Linux 3.8 より前のバージョンでは、このソケットオプションは getsockname(2) で設定することはできたが、取得することができなかった。 Linux 3.8 以降では、読み出すことができる。 optlen 引き数には、 デバイス名を格納するのに十分なバッファーサイズを渡すべきであり、 IFNAMSIZ バイトにすることを推奨する。 実際のデバイス名の長さは optlen 引き数に格納されて返される。

::


  #if defined(TCP_MAXSEG)
  	if (listener->maxseg > 0) {
  		if (setsockopt(fd, IPPROTO_TCP, TCP_MAXSEG,
  			       &listener->maxseg, sizeof(listener->maxseg)) == -1) {
  			msg = "cannot set MSS";
  			err |= ERR_WARN;
  		}
  	}
  #endif

man 7 tcpによると説明は以下。

  TCP_MAXSEG
  送出 TCP パケットの最大セグメントサイズ。 Linux 2.2 以前と Linux 2.6.28 以降では、このオプションを接続確立の前に設定すると、初期パケット で他端にアナウンスする MSS の値も変化する。インターフェースの MTU より も大きな (あるいは大きくなってしまった) 値は効果を持たない。 また TCP は、この値よりも最小・最大の制限の方を優先する。

注：ジャンボパケットを使える環境だと有効に働くオプションなのだろうか。

::

  #if defined(TCP_DEFER_ACCEPT)
  	if (listener->options & LI_O_DEF_ACCEPT) {
  		/* defer accept by up to one second */
  		int accept_delay = 1;
  		if (setsockopt(fd, IPPROTO_TCP, TCP_DEFER_ACCEPT, &accept_delay, sizeof(accept_delay)) == -1) {
  			msg = "cannot enable DEFER_ACCEPT";
  			err |= ERR_WARN;
  		}
  	}

man 7 tcpによると以下。
  TCP_DEFER_ACCEPT (Linux 2.4 以降)
  これを用いると、リスナはデータがソケットに到着した時のみ目覚めるようになる。 整数値 (秒) をとり、 TCP が接続を完了しようと試みる回数を制限できる。 移植性の必要なプログラムではこのオプションを用いるべきではない。

この仕組み自体の解説は以下のブログが参考になる。
http://blog.yuuk.io/entry/2013/07/21/022859

::

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

FASTOPENについては以下のblog記事が参考になる。
https://html5experts.jp/jxck/3529/
::

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

ソケットをbindする。一応、man 2 bindの説明は以下。

説明
       socket(2)  でソケットが作成されたとき、そのソケットは名前空間 (アドレス
       ・ファミリー) に存在するが、アドレスは割り当てられていない。 bind()  は
       、 ファイルディスクリプタ sockfd で参照されるソケットに addr で指定され
       たアドレスを割り当てる。 addrlen には addr が指すアドレス構造体のサイズ
       を バイト単位で指定する。伝統的にこの操作は「ソケットに名前をつける」と
       呼ばれる。


::
  
  	ready = 0;
  	ready_len = sizeof(ready);
  	if (getsockopt(fd, SOL_SOCKET, SO_ACCEPTCONN, &ready, &ready_len) == -1)
  		ready = 0;

SO_ACCEPTCONNのman 7 socketの説明は以下。
  SO_ACCEPTCONN
  このソケットが listen(2) によって接続待ち受け状態に設定されているかどうかを示す値を返す。 値 0 は listen 状態のソケットでないことを、 値 1 は listen 状態のソケットであることを示す。このソケットオプションは読み込み専用である。

getsockoptが失敗した場合は、readyを0に設定する。つまりlisten状態のソケットが無いということを前提にする。

::
  
  	if (!(ext && ready) && /* only listen if not already done by external process */
  	    listen(fd, listener->backlog ? listener->backlog : listener->maxconn) == -1) {
  		err |= ERR_RETRYABLE | ERR_ALERT;
  		msg = "cannot listen to socket";
  		goto tcp_close_return;
  	}

もし、外部プロセスによって該当ソケットがすでにlisten状態になければ、listenを
実行する。

(HAProxyでこのタイミングで外部プロセスがこのソケットをlistenすることってあるの？)
::

  
  #if defined(TCP_QUICKACK)
  	if (listener->options & LI_O_NOQUICKACK)
  		setsockopt(fd, IPPROTO_TCP, TCP_QUICKACK, &zero, sizeof(zero));
  #endif

TCP_QUICKACKのman 7 tcpの説明は以下。
  TCP_QUICKACK (Linux 2.4.4 以降)
  設定されていると quickack モードを有効にし、クリアされると無効にする。 通常の TCP 動作では ack は必要に応じて遅延されるのに対し、 quickack モードでは ack はすぐに送信される。 このフラグは永続的なものではなく、 quickack モードから/モードへ切り替えるためのものである。 これ以降の TCP プロトコルの動作によっては、 内部のプロトコル処理や、遅延 ack タイムアウトの発生、 データ転送などの要因によって、 再び quickack から出たり入ったりする。 移植性の必要なプログラムではこのオプションを用いるべきではない。
(このタイミングで外部プロセスがこのソケットをlistenすることってあるの？)

なお、このオプションは有効になったり、ならなかったりするらしい。詳細は以下のblog参照。
http://www.anarg.jp/personal/t-tugawa/note/misc/delayed_ack.html

まぁ、このオプションは動作自体が不安定らしいが、なぜ、そもそもHAProxyで
このオプションを設定する必要があるのだろう？その理解には以下のwikiが
役に立つ。https://ja.wikipedia.org/wiki/TCP%E9%81%85%E5%BB%B6ACK

太郎と花子の話だが、要するにクライアント側がもしかしたら、nagleアルゴリズムが
有効になっている場合がある。それを考慮して、HAProxy側がどんどん、
ACKを返して、nagleを使っているクライアントの送信を促すようにしていると思う。

::

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


