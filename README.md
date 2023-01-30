<div align="right">
pinentry-tty patch for cygwin 2022-09-15<br>
added about pinentry-curse( pinentry-1.2.1 ) 2023-01-10<br>
added try using cygwin 3.5.0 2023-01-30
</div>

# Cygwin で pinentry-tty を使う

## やりたい事
gpg コマンド実行時、そのままターミナルからパスフレーズを入力したい。

## 背景
Cygwin の pinentry パッケージ（pinentry-1.0.0-2）には pinentry-tty が同梱されていません。

## パッチ作成の経緯
パッチを当てずに作成した pinentry-tty.exe を使うと、次のようにパスフレーズを入力できずに操作がキャンセルされ終了します。
~~~
パスフレーズ:
gpg: 署名に失敗しました: 操作がキャンセルされました
gpg: signing failed: 操作がキャンセルされました
~~~
プログラムを追ってみると、まず `tcsetattr() @ tty/pinentry-tty.c#61` が失敗している事が分かります。この時の errno は 5(EIO) でした。

[tcsetattr の説明](https://pubs.opengroup.org/onlinepubs/009696799/functions/tcsetattr.html) によると、errno 5 は次のようです。
> *The process group of the writing process is orphaned, and the writing process is not ignoring or blocking SIGTTOU.*

そこで、次のように gpg-agent を起動してみると、`tcsetattr() @ tty/pinentry-tty.c#61` は通過しますが、その後 `fgetc () @ tty/pinentry-tty.c#339` が errno 5(EIO) で失敗します。
~~~
$ (trap '' SIGTTOU; gpg-connect-agent /bye)
~~~

[fgetc の説明](https://pubs.opengroup.org/onlinepubs/9699919799/functions/fgetc.html) によると、errno 5 は次のようです。
> *A physical I/O error has occurred, or the process is in a background process group attempting to read from its controlling terminal, and either the calling thread is blocking SIGTTIN or the process is ignoring SIGTTIN or the process group of the process is orphaned. This error may also be generated for implementation-defined reasons.*

この条件を見る限り、SIGTTIN をいじる必要はなさそうですが、試しに無視してみます。
~~~
$ (trap '' SIGTTOU SIGTTIN; gpg-connect-agent /bye)
~~~
すると `fgetc () @ tty/pinentry-tty.c#339` も通過し、パスフレーズを入力できるようになりました。

**pinentry-tty.c.ign-SIGTTOU-and-SIGTTIN.patch** は、これを元に、`main() @ tty/pinentry.c#565` に入った直後に、SIGTTOU と SIGTTIN を無視しています。ですので、
- gpg-agent 起動時に trap した場合は、パッチは必要ありません。
- パッチを当てた場合は、gpg-agent 起動時に trap する必要はありません。

## Linux との比較
Linux ではこのような事は不要です。`tty_cmd_handler() @ tty/pinentry-tty.c#499` に入った所で SIGTTOU、SIGTTIN それぞれの sa_handler の様子を確認すると、Linux は共に 0(SIG_DFL) となっており、パスフレーズを入力できますが、Cygwin はその状況（trap なしで起動の場合は共に 0(SIG_DFL) となる）では失敗します。Cygwin は trap することで sa_handler の値が共に 1(SIG_IGN) となり、その場合に、パスフレーズ入力ができるようになります。

私はこの違いが何に起因しているのか分かりません。この Linux と Cygwin の違いを理解されている方が、修正を提供して下さる事を期待しています。

## pinentry-tty を作り、設置・指定する
pinentry ソースパッケージ（pinentry-1.0.0-2-src）に pinentry-tty が同梱されていますので、それを使います。
~~~
$ tar Jxf pinentry-1.0.0-2-src.tar.xz
$ cd pinentry-1.0.0-2.src
$ patch < /path/to/pinentry.cygport.only-make-pinentry-tty.patch
$ cygport pinentry.cygport prep
~~~
gpg-agent 起動時に SIGTTOU と SIGTTIN を無視する場合、次のパッチは不要です。
~~~
$ cd pinentry-1.0.0-2.x86_64/src/pinentry-1.0.0/tty/
$ patch < /path/to/pinentry-tty.c.ign-SIGTTOU-and-SIGTTIN.patch
$ cd ../../../../
~~~
以降の /usr/local/PinentryTty/ はお好みです。
~~~
$ cygport pinentry.cygport compile
$ mkdir -p /usr/local/PinentryTty/
$ cp ./pinentry-1.0.0-2.x86_64/build/tty/pinentry-tty.exe /usr/local/PinentryTty/
~~~
次の行を ~/.gnupg/gpg-agent.conf の適当な所に追加し、設置した pinentry-tty を指定します。
~~~
pinentry-program /usr/local/PinentryTty/pinentry-tty
~~~

## pinentry-curse を使う
pinentry 1.2.1<sup>(*1)</sup> を、[gnupg.org](https://www.gnupg.org/download/index.html#pinentry) から入手し、make します。
~~~
$ tar jxf pinentry-1.2.1.tar.bz2
$ cd pinentry-1.2.1
$ ./configure --enable-pinentry-curses --enable-pinentry-tty
$ make
~~~
curses/pinentry-curses.exe をお好みの場所へ置き、~/.gnupg/gpg-agent.conf の pinentry-program に指定後、gpg-agent 起動時に SIGTTOU と SIGTTIN を無視して利用します<sup>（*2）</sup>。
~~~
$ (trap '' SIGTTOU SIGTTIN; gpg-connect-agent /bye)
~~~

SIGTTIN を無視すると動作する理由は分かりません。[fgetc の説明](https://pubs.opengroup.org/onlinepubs/9699919799/functions/fgetc.html) では、SIGTTIN を 無視する/しない に係わらず、エラーになると考えられる為です。Cygwin の実装上の理由だとすると、その実装が変わらない間はこの方法で利用できそうです。

<sub>(*1) どのバージョンからかは分かりませんが、入手時 最新の 1.2.1 は日本語もきれいに納まってレイアウトされます！</sub>

<sub>(*2) 敢えて、SIGTTOU と SIGTTIN を無視するパッチを作るならば、pinentry ではなく、gpg-agent を変更する方が pinentry-tty と pinentry-curse それぞれにパッチ当てずに済みますので良いかもしれません（GnuPG のソースパッケージ中の agent/gpg-agent.c 見ると、`main() @ agent/gpg-agent.c#1766` 辺りで SIGPIPE に対して SIG_IGN を設定していますので、そこで SIGTTOU と SIGTTIN を SIG_IGN に設定すると良さそうです。ここは gpg-agent に対する要求処理に入って行く手前のようです）。</sub>

## cygwin 3.5.0-0.150.g4b157b44cad0(Test) を使ってみる
SIGTTOU, SIGTTIN の trap で動作する事をソース上で追ってみると、`fhandler_termios::bg_check() @ winsup/cygwin/fhandler/termios.cc#204` に興味が出てきました。そこで FAQ 6.21 に従って cygwin を make し確認（トレース）しようとすると、trap 無しで pinentry-{tty,curses} が動作しました。

試しに cygwin 3.5.0-0.150.g4b157b44cad0(Test) を使ってみると、こちらも trap 無しで動作します。

どうやら、3.5 バージョンでは pinentry-{tty,curses} が動作するようです。pinentry パッケージも併せて更新されることを期待しています。
