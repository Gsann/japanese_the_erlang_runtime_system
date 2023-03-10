# 1. Erlang Runtime Systemの導入

Erlang Runtime System（ERTS）は多くの相互依存するコンポーネントからなる複雑なシステムです。ERTSは非常にポータブルな方法で書かれているので、板ガムのようなコンピュータからテラバイトのメモリを持つ最大のマルチコアシステムまで、どんなものでも動作させることができます。このようなシステムの性能を自分のアプリケーションに最適化できるようにするためには、自分のアプリケーションを知るだけでなく、ERTSそのものを徹底的に理解する必要があるのです。

## 1.1 ERTSとErlang Runtime System

任意のErlang Runtime Systemと、Erlang Runtime Systemの特定の実装は違います。Ericssonによる "Erlang/OTP" はErlangとErlang Runtime Systemのデファクトスタンダード実装です。この本ではこの実装をERTSと呼ぶか、Erlang RunTime Systemと綴ります（OTPの定義については1.3節を参照してください）。

Erlang Runtime Systemとは何か、Erlang仮想マシンとは何かという公式な定義はありません。ERTSから実装の詳細を取り除けば、理想的なプラトニックシステムがどんなものか想像できるかもしれません。これは残念ながら循環する定義です。なぜなら、実装に特有の詳細を特定するためには、一般的な定義を知る必要があるからです。Erlangの世界では、私たちはこのことを心配するのは現実的ではありません。

私たちはErlang Runtime Systemという言葉を、Ericssonによる特定の実装（Erlang RunTime Systemまたは通常は単にERTSと呼びます）とは対照的に、Erlang Runtime Systemの一般的な考え方を示すために使うことにします。

注意 この本はほとんどERTSについての本で、一般的なErlang Runtime Systemについてはほんの少ししか書かれていません。もしあなたが、私たちが一般的な原理について話していると明言されていない限り、Ericssonの実装について話していると仮定するならば、おそらくそれは正しいでしょう。

## 1.2. 本書の読み方

本書の第二部では、アプリケーションのためにランタイムシステムをチューニングする方法と、アプリケーションとランタイムシステムのプロファイリングとデバッグの方法について見ていきます。システムをチューニングする方法を本当に知るためには、システムを知ることも必要です。本書の第一部では、ランタイムシステムがどのように動作するかを深く理解することができます。

第一部の次以降の章では、システムの各コンポーネントを個別に説明します。他のコンポーネントがどのように実装されているかを完全に理解しなくても、これらの章のどれかを読むことはできるはずですが、それぞれのコンポーネントが何であるかについての基本的な理解は必要でしょう。この入門編の残りの章では、第一部の残りの章を好きな順番で読破するだけの基礎的な理解と語彙を身につけることができるはずです。

しかしながら、もし時間があれば、最初のうちはこの本を順番に読んでください。ErlangやERTSに特有の単語や、この本で特別に使われている単語は、たいてい最初に出てきたところで説明されています。そしてその単語を知ったら、特定のコンポーネントで問題が発生したときにいつでも戻って第一部を参照することができます。


## 1.3. ERTS

### 1.3.1. The Erlang Node (ERTS)

ElixirやErlangのアプリケーションやシステムを起動するとき、実際に起動するのはErlangノードです。このノードではErlang RunTime Systemと仮想マシンBEAMが動作しています。(またはErlangの他の実装（節1.4参照）)を実行します。

あなたのアプリケーションコードはErlangノードで実行され、ノードのすべてのレイヤーはアプリケーションのパフォーマンスに影響します。私たちはノードを構成するレイヤーのスタックを見ていきます。これはあなたのシステムを様々な環境で動かすためのオプションを理解するのに役立ちます。

オブジェクト指向の用語では、ErlangノードはErlang Runtime Systemクラスのオブジェクトだと言うことができます。Javaの世界ではJVMインスタンスがこれに相当します。

Elixir/Erlangのコードの実行はすべてノード内で行われます。Erlangノードは1つのOSプロセスで動作し、1台のマシンで複数のErlangノードを動作させることができます。

Erlang OTPドキュメントによると、ノードとは実際には実行中のランタイムシステムで、名前が与えられています。つまり、 `--name NAME@HOST` や `--sname NAME` （Erlangランタイムでは -name と -sname ）というコマンドラインスイッチで名前を付けずにElixirを起動すると、ランタイムはあってもノードはない状態になってしまいます。このようなシステムでは `Node.alive?` (Erlangでは`is_alive()`）の関数はfalseを返します。


```
$ iex
Erlang/OTP 19 [erts-8.1] [source-0567896] [64-bit] [smp:4:4]
              [async-threads:10] [hipe] [kernel-poll:false]

Interactive Elixir (1.4.0) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> Node.alive?
false
iex(2)>
```

ランタイムシステム自体は、用語の使い分けにそれほど厳密ではありません。名前をつけていなくてもノードの名前を聞くことができます。Elixirでは`Node.list`という関数に `:this` という引数をつけて使いますし、Erlangでは `nodes(this)...` と呼びます。

```
iex(2)> Node.list :this
[:nonode@nohost]
iex(3)>
```

本書では、ランタイムの実行中のインスタンスには、名前が与えられているかどうかに関わらず、ノードという用語を使用することにします。


### 1.3.2. 実行環境におけるレイヤー

あなたのプログラム（アプリケーション）は1つまたは複数のノードで実行され、プログラムのパフォーマンスはアプリケーションのコードだけでなく、ERTSスタック内のコードの下にある全てのレイヤーに依存します。図1では、1台のマシン上で2つのErlangノードが動作しているERTSスタックの図解が見られます。

Figure 1. ERTS Stack <img width="200" alt="qiita-square" src="figure/Figure 1. ERTS Stack.png">

Elixirを使用している場合、スタックにさらに別のレイヤーが存在します。

Figure 2. Elixir Stack <img width="200" alt="qiita-square" src="figure/Figure 2. Elixir Stack.png">

ここでは、スタックの各レイヤーを見て、アプリケーションの必要性に応じてどのようにチューニングできるかを見てみましょう。

スタックの一番下には、あなたが実行しているハードウェアがあります。アプリのパフォーマンスを向上させる最も簡単な方法は、より良いハードウェアで実行することでしょう。経済的、物理的な制約や環境の問題でハードウェアをアップグレードできない場合は、より高いレベルのスタックを探索する必要があるかもしれません。

ハードウェアの選択で最も重要なのは、マルチコアかどうか、32ビットか64ビットか、の2点です。マルチコアを使うかどうか、32 ビットか 64 ビットかによって、ERTS の異なるビルドが必要です。

スタックの下から2番目の層は、OSレベルです。ERTSはWindowsのほとんどのバージョンと、Linux、VxWorks、FreeBSD、Solaris、Mac OS Xなど、ほとんどのPOSIX「準拠」オペレーティングシステムで実行されます。現在、ERTSの開発のほとんどはLinuxとOS Xで行われており、これらのプラットフォームで最高のパフォーマンスを期待することができます。しかし、Ericssonは多くのプロジェクトで内部的にSolarisを使用しており、ERTSは何年も前からSolaris用にチューニングされています。ユースケースによっては、Solarisシステムで最高のパフォーマンスを得ることができるかもしれません。OSの選択は、通常、性能要件に基づくものではなく、他の要因によって制限されます。組み込みアプリケーションを構築する場合はRaspbianやVxWorkに制限されるかもしれませんし、何らかの理由でエンドユーザーやクライアントアプリケーションを構築する場合はWindowsを使用しなければならないかもしれません。ERTSのWindowsへの移植は、これまでのところ最優先事項ではなく、パフォーマンスやメンテナンスの観点からもベストな選択とは言えないかもしれません。64ビットERTSを使用する場合は、当然ながら64ビットマシンと64ビットOSの両方が必要です。本書では、OSに特化した質問はあまり取り上げず、ほとんどの例ではLinuxで実行することを想定しています。

スタックの3番目のレイヤーはErlang Runtime System（Erlangランタイムシステム）です。私たちの場合、これはERTSになります。この層と4番目の層であるErlang Virtual Machine (BEAM)がこの本の全てです。

第5層のOTPはErlangの標準ライブラリを提供します。OTPはもともと "Open Telecom Platform" の略で、堅牢なアプリケーション（電話交換機など）を構築するためのビルディングブロック（`supervisor`, `gen_server`, `gen_tcp`など）を供給するErlangライブラリのことです。初期の頃、OTPのライブラリと意味はERTSに同梱されている他のすべての標準ライブラリと混同されました。現在では、ほとんどの人がOTPをErlangとともにERTSとEricssonが提供しているすべてのErlangライブラリの名前として「Erlang/OTP」を使っています。これらの標準ライブラリを知り、いつ、どのように使用するかを知ることで、アプリケーションのパフォーマンスを大幅に向上させることができます。この本では、標準ライブラリやOTPの詳細には触れません。これらの点をカバーしている本は他にもたくさんあります。

Elixirプログラムを実行する場合、第6層はElixir環境とElixirライブラリを提供します。

最後に、第7層（APP）は、アプリケーションと、使用するサードパーティライブラリです。アプリケーションは、その下にあるレイヤーが提供するすべての機能を利用することができます。ハードウェアのアップグレードとは別に、アプリケーションのパフォーマンスを最も容易に向上させることができるのは、この部分でしょう。18章では、アプリケーションのプロファイリングと最適化に役立つヒントとツールを紹介しています。第19章では、アプリケーションがクラッシュする原因を探る方法と、アプリケーションのバグを発見する方法について紹介します。

Erlangノードを構築して実行する方法については付録Aを、Erlangノードの構成要素についてはこの本の残りの部分を読んでください。

### 1.3.3. ディストリビューション

Erlang設計者の重要な洞察の1つは、24時間365日動作するシステムを構築するためには、ハードウェアの故障に対応する必要があるということでした。したがって、少なくとも2つの物理的なマシンにシステムを分散させる必要があります。それぞれのマシンでノードを起動し、お互いに接続することで、プロセスが同じノードで動いているのと同じようにノードを越えて通信できるようにします。

Figure 3. Distributed Applications <img width="350" alt="qiita-square" src="figure/Figure 3. Distributed Applications.png">

### 1.3.4. Erlangコンパイラ

ErlangコンパイラはErlangのソースコード、.erlファイルからBEAM（仮想マシン）用の仮想マシンコードにコンパイルする役割を担っています。コンパイラ自体はErlangで書かれていて、それ自身によってBEAMのコードにコンパイルされ、通常は実行中のErlangノードで利用できます。ランタイムシステムを起動するために、bootstrapディレクトリにコンパイラを含む多くのプリコンパイルされたBEAMファイルがあります。

コンパイラの詳細については、第2章を参照してください。

### 1.3.5. Erlang仮想マシン:BEAM

BEAMはErlangのコードを実行するためのErlang仮想マシンです。ちょうどJavaのコードを実行するためのJVMのようなものです。BEAMはErlang Nodeの中で動きます。

```
BEAMという名前はもともとBogdan's Erlang Abstract Machineの略でしたが、今ではほとんどの人が現在保守している人の名前をとってBjörn's Erlang Abstract Machineと呼んでいます。
```

ERTSがより一般的なErlangランタイムシステムの実装であるように、BEAMはより一般的なErlang仮想マシン(EVM)の実装です。EVMを構成するものの定義はありませんが、BEAMは実際には2つのレベルの命令を持っています `Generic Instructions`（汎用命令）と`Specific Instructions`（特殊命令）です。汎用命令セットはEVMの設計図のようなものです。

BEAMの詳細については、第5章、第6章、第7章を参照してください。

### 1.3.6. プロセス

Erlangのプロセスは基本的にOSのプロセスのように動きます。各プロセスはそれ自身のメモリ（メールボックス、ヒープ、スタック）とプロセスに関する情報を持つプロセスコントロールブロック（PCB）を持っています。

すべてのErlangコードの実行はプロセスのコンテキスト内で行われます。1つのErlangノードには多くのプロセスを持つことができ、それらはメッセージパッシングやシグナルを通して通信することができます。Erlangプロセスは他のErlangノード上のプロセスと、そのノードが接続されている限り通信することができます。

プロセスおよび PCB についてより詳しく知りたい場合は、3章を参照してください。

### 1.3.7. スケジューリング

スケジューラは実行するErlangプロセスを選択する責任があります。基本的にスケジューラは2つのキュープロセスを保持しています。実行可能なプロセスの実行可能キューと、メッセージを受け取るのを待っているプロセスの待ち行列です。待ち行列の中のプロセスがメッセージを受け取るかタイムアウトになると、そのプロセスは準備完了の待ち行列に移動します。

スケジューラは実行可能キューから最初のプロセスを選び、1つのタイムスライスを実行するためにBEAMに渡します。BEAMはタイムスライスを使い切った時点で実行中のプロセスを先取りし、そのプロセスを実行可能キューの末尾に追加します。もしプロセスがタイムスライスを使い切る前にレシーブでブロックされた場合、代わりに待ち行列に追加されます。

Erlangは本質的に並行処理です。つまり、各プロセスは概念的には他のすべてのプロセスと同時に動いていますが、実際には仮想マシンの中で動いているのは1つのプロセスだけなのです。マルチコアマシンではErlangは実際には複数のスケジューラを動かしていて、通常は物理コアごとに1つ、それぞれが自分のキューを持っています。このようにしてErlangは真の並列性を実現します。複数のコアを利用するために、ERTSはSMPモードでビルドされなければなりません（付録Aを参照）。SMPは`Symmetric MultiProcessing`の略で、複数のCPUのうちのどれか1つでプロセスを実行する能力です。

実際には、プロセス間に優先順位があり、待ち行列はタイミングホイールで実装されるなど、より複雑な様相を呈しています。これらについては、第11章で詳しく説明します。

### 1.3.8. Erlangタグスキーム

Erlangは動的型付け言語なので、ランタイムシステムはそれぞれのデータオブジェクトの型を追跡する方法が必要です。これはタグ付けのスキームで行われます。それぞれのデータオブジェクトやそのポインタには、オブジェクトのデータ型に関する情報を持ったタグがあります。

基本的にポインタの一部のビットはタグ用に確保され、エミュレータはそのタグのビットパターンを見てオブジェクトのタイプを判断することができます。

これらのタグは、パターンマッチングや型検査、プリミティブ操作のほか、ガベージコレクタでも使用されています。

タグ付けの完全なスキームについては、第4章で説明します。

### 1.3.9. メモリハンドリング

Erlangは自動メモリ管理を採用しているので、プログラマはメモリの割り当てや解放を心配する必要はありません。各プロセスはヒープとスタックを持ち、どちらも必要に応じて大きくしたり小さくしたりすることができます。

プロセスがヒープ領域を使い果たした場合、仮想マシンはまずガベージコレクションによって空きヒープ領域を取り戻そうとします。ガベージコレクタは、プロセススタックとヒープを調べて、生きているデータを新しいヒープにコピーし、死んでいるデータはすべて捨てます。それでもヒープ領域が足りない場合は、より大きなヒープが新たに割り当てられ、生きているデータはそこに移動されます。

参照カウントされたバイナリの処理を含む、現在の世代コピー型ガベージコレクタの詳細は、第12章に記載されています。

HiPEコンパイルされたネイティブコードを使用するシステムでは、各プロセスは実際にはBEAMスタックとネイティブスタックの2つのスタックを持っています。

### 1.3.10. インタプリタとコマンドラインインターフェース

Erlangノードをerlで起動すると、コマンドプロンプトが表示されます。これはErlangの`read eval print loop` (REPL)、コマンドラインインターフェース (CLI)、または単にErlangのシェルです。

実際にErlangのコードを入力して、シェルから直接実行することができます。この場合、コードはBEAMコードにコンパイルされてBEAMによって実行されるのではなく、Erlangインタプリタによってパースされて解釈されます。一般的に解釈されたコードはコンパイルされたコードと全く同じように動作しますが、いくつかの微妙な違いがあります。これらの違いやシェルの他のすべての側面については20章で説明します。

## 1.4. 他のErlangの実装

この本は主にEricsson/OTPによるErlangの「標準」実装であるERTSを扱いますが、他にもいくつかの実装があり、この節ではそれらのいくつかを簡単に見ていきます。

### 1.4.1. Erlang on Xen

Erlang on Xen (http://erlangonxen.org) はサーバーハードウェア上で直接動作するErlang実装で、間にOSレイヤーを介さず、薄いXenクライアントだけで動作します。

Erlang on Xenの仮想マシンであるLingはBEAMとほぼ100%バイナリ互換があります。`xref:the_eox_stack`ではErlang on XenのErlang Solution Stackの実装がERTS Stackとどう違うか見ることができます。ここで注意しなければならないのは、Erlang on Xenスタックにはオペレーティングシステムがないということです。

LingはBEAMの汎用命令セットを実装しているので、OTP層からBEAMコンパイラを再利用してErlangをLingにコンパイルすることが可能です。

Figure 4. Erlang On Xen <img width="350" alt="qiita-square" src="figure/Figure 4. Erlang On Xen.png">

### 1.4.2. Erjang

Erjang (http://www.erjang.org) はJVM上で動作するErlangの実装です。.beamファイルをロードして、コードをJavaの.classファイルにリコンパイルします。Erjangは（一般的な）BEAMとほぼ100%バイナリ互換性があります。

`xref:the_erjang_stack`ではErjangのErlang Solution Stackの実装がERTS Stackとどう違うかを見ることができます。ここで注目すべき点は、仮想マシンとしてJVMがBEAMに取って代わり、ErjangはVMの上にJavaで実装することによってERTSのサービスを提供していることです。

Figure 5. Erlang on the JVM <img width="350" alt="qiita-square" src="figure/Figure 5. Erlang on the JVM.png">

ERTSの主要な構成要素と必要な語彙の基本的な理解ができたので、各コンポーネントの詳細に飛び込むことができるようになりました。もし、あるコンポーネントを理解したいのであれば、その章に直接ジャンプすることができます。また、特定の問題に対する解決策をどうしても見つけたい場合は、第二部の適切な章に飛び、システムを調整、微調整、またはデバッグするためのさまざまな方法を試すことができます。