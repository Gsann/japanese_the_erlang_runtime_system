# 序文

この本は、正しく美しいコードを書く方法について書かれたものではありません。この本は、プロファイリングやパフォーマンスチューニングについてもあまり触れていません。しかし、この本にはトレースとプロファイリングの章があり、ボトルネックや不必要なリソースの使い方を見つけるのに役立ちます。また、パフォーマンスチューニングの章もあります。

この2つの章はこの本の最後の章であり、この本全体はこの章のために構成されています。しかしこの本の本当のゴールは、あなたがErlangアプリケーションのパフォーマンスを本当に理解するために必要なすべての情報、すべての血生臭い詳細を提供することです。

## 本書について

こんな人におすすめです。Erlangのインストールをチューニングしたい。VMクラッシュをデバッグする方法を知りたい。Erlangアプリケーションのパフォーマンスを改善したい。Erlangが実際にどのように動作するのか理解したい。自分自身のランタイム環境を構築する方法を知りたい。

VMをデバッグしたい場合 VMを拡張したい場合 パフォーマンスを調整したい場合 - 最後の章にジャンプしてください ...ただし、その章を本当に理解するには、この本を読む必要があります。

## 本書の読み方

Erlangランタイムシステム（ERTS）は多くの相互依存するコンポーネントからなる複雑なシステムです。ERTSは非常にポータブルな方法で書かれているので、ガムテープのようなコンピュータからテラバイトのメモリを持つ最大のマルチコアシステムまで、どんなものでも動作させることができます。このようなシステムの性能を自分のアプリケーションに最適化できるようにするためには、自分のアプリケーションを知るだけでなく、ERTSそのものを徹底的に理解する必要があります。

ERTS の動作に関するこの知識があれば、ERTS 上で実行したときにアプリケーションがどのように動作するかを理解することができ、また、アプリケーションのパフォーマンスに関する問題を発見して修正することができるようになります。本書の第 2 部では、ERTS アプリケーションの実行、監視、およびスケーリングを成功させる方法について説明します。

この本を読むのにErlangプログラマである必要はありませんが、Erlangが何であるかという基本的な理解は必要でしょう。この次の章ではErlangの背景を説明します。

## Erlangについて

このセクションでは、この本の他の部分を理解するのに重要な、いくつかの基本的なErlangの概念について見ていきます。

Erlangは特にErlangの作者の1人であるJoe Armstrongによって、並行処理指向の言語と呼ばれています。同時実行は間違いなくErlangの中心で、Erlangのシステムがどのように動くかを理解するためには、Erlangの同時実行モデルを理解する必要があります。

まず最初に、並行処理と並列処理の区別をする必要があります。この本では、並行処理とは、互いに独立して実行できる2つ以上のプロセスを持つという概念です。これは、最初に1つのプロセスを実行してからもう1つのプロセスを実行したり、実行をインターリーブしたり、プロセスを並行して実行することによって実現できます。並列実行とは、複数の物理的な実行ユニットを使って、プロセスが実際に全く同時に実行されることを意味します。並列実行は、さまざまなレベルで実現することができます。1つのコアにある実行パイプラインの複数の実行ユニット、1つのCPUの複数のコア、1つのマシンの複数のCPU、複数のマシンを介して行われます。

Erlangは並行処理を実現するためにプロセスを使います。概念的にはErlangプロセスはほとんどのOSプロセスと似ていて、並列に実行され、シグナルを通して通信することができます。実際には、ErlangプロセスはほとんどのOSプロセスよりもずっと軽量であるという大きな違いがあります。他の多くの並行処理プログラミング言語では、Erlangプロセスに相当するものをエージェントと呼んでいます。

ErlangはErlang仮想マシンであるBEAM上でプロセスの実行をインターリーブすることで並行処理を実現します。マルチコアプロセッサでは、BEAMはコアごとに1つのスケジューラを実行し、スケジューラごとに1つのErlangプロセスを実行することによっても並列性を実現することができます。Erlangシステムの設計者はシステムを複数のコンピュータに分散することで、さらなる並列性を実現することができます。

典型的なErlangシステム（Erlangで構築されたサーバーやサービス）は、ディスク上のディレクトリに対応するいくつかのErlangアプリケーションから構成されています。それぞれのアプリケーションはディレクトリの中のファイルに対応するいくつかのErlangモジュールから構成されています。それぞれのモジュールはいくつかの関数を含んでいて、それぞれの関数は式で構成されています。

Erlangは関数型言語なので、ステートメントはなく、式だけです。Erlangの式はErlang関数にまとめることができます。関数はいくつかの引数を取って値を返します。Erlangのコード例ではErlangの式と関数の例を見ることができます。

ErlangにはVMが実装する組み込み関数（BIFs）がたくさんあります。これは lists:append の実装のように、効率的な理由からです（これはErlangで実装できます）。また、list_to_atomのようにErlang自身では実装が難しい、または不可能な低レベルの機能を提供するためでもあります。

Erlang/OTP R13B03以降では、NIF（Native Implemented Functions）インターフェイスを使用して、C言語で実装された独自の関数を提供することもできます。

## 謝辞

まず最初に、EricssonのOTPチーム全員に感謝したいと思います。ErlangとErlangランタイムシステムを保守してくれて、さらに私の質問に辛抱強く答えてくれたことに。特にKenneth Lundin、Björn Gustavsson、Lukas Larsson、Rickard Green、Raimo Niskanenに感謝したいと思います。

また、本書に多大な貢献をいただいた田中義博氏、Roberto Aloi氏、Dmytro Lytovchenko氏、本書の制作を支援していただいたHappiHacking社とTubiTV社に感謝いたします。

最後に、編集や修正にご協力いただいた皆様に、心から感謝いたします。

```
Yoshihiro Tanaka
Dmytro Lytovchenko
Trevor Brown
Erick Dennis
Buddhika Chathuranga
Ramkumar Rajagopalan
Kim Shrier
Antonio Nikishaev
fred
yoshi
```

