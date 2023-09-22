# ns-3 メモ

# Tutorial
[Tutorial Page](https://www.nsnam.org/docs/release/3.39/tutorial/singlehtml/index.html)

## Conceptual Overview
システム内のいくつかの中核的な概念と抽象化を説明する。

### Key Abstractions
このセクションでは、ネットワーキングで一般的に使用されるいくつかの用語を説明するが、ns-3内では特定の意味を持っている。

#### Node
- ns-3では、基本的なコンピュータデバイスの抽象化は「ノード」と呼ぶ。
- この抽象化は、C++でNodeクラスとして表される。
- Nodeクラスは、シミュレーション内のコンピュータデバイスの表現を管理するためのメソッドを提供する。

ノードは、機能を追加するコンピュータと考えられる。
アプリケーション、プロトコルスタック、および関連するドライバーを追加して、コンピュータが有用な作業を行えるようにする。

#### Application
- ns-3では、実際のオペレーティングシステムの概念は存在しないが、アプリケーションの概念は存在する。
- ソフトウェアアプリケーションがコンピュータ上で「実世界」でタスクを実行するように、ns-3アプリケーションはシミュレートされた世界でシミュレーションを駆動するためにns-3ノード上で実行される。
- この抽象化は、C++でApplicationクラスとして表される。
  - Applicationクラスは、シミュレーション内でユーザーレベルのアプリケーションの表現を管理するためのメソッドを提供する。
- このチュートリアルでは、Applicationクラスの特殊化であるUdpEchoClientApplicationとUdpEchoServerApplicationを使用する。
  - これらのアプリケーションは、シミュレートされたネットワークパケットを生成してエコーするクライアント/サーバーアプリケーションセットを構成する。

#### Channel
- ネットワークを通じてデータが流れるメディアは通常「チャネル」と呼ばれる。
- ns-3のシミュレートされた世界では、ノードを通信チャネルを表すオブジェクトに接続する。
- ここで、基本的な通信サブネットの抽象化は「チャネル」であり、C++ではChannelクラスで表される。
- Channelクラスは通信サブネットオブジェクトを管理し、ノードをそれに接続するためのメソッドを提供する。
- 開発者はオブジェクト指向プログラミングの意味でチャネルを特化させることもできます。チャネルの特化は、単なるワイヤのようなものから、Ethernetスイッチのような複雑なもの、無線ネットワークの場合の障害物がある三次元空間のようなものまで、さまざまなものをモデル化することができる。
- このチュートリアルでは、CsmaChannel、PointToPointChannel、WifiChannelといったチャネルの特化バージョンを使用する。たとえば、CsmaChannelは、キャリアセンスマルチアクセス通信媒体を実装した通信サブネットのバージョンをモデル化し、Ethernetのような機能を提供する。

#### Net Device
- ns-3では、ネットデバイスの抽象化はソフトウェアドライバーとシミュレートされたハードウェアの両方をカバーしています。
- ネットデバイスは、Nodeに「インストール」され、Nodeがチャネルを介してシミュレーション内の他のNodeと通信できるようにします。
- 実際のコンピュータと同様に、Nodeは複数のNetDeviceを介して複数のチャネルに接続される可能性があります。
- ネットデバイスの抽象化は、C++でNetDeviceクラスで表されます。
  - NetDeviceクラスはNodeとChannelオブジェクトへの接続を管理するためのメソッドを提供し、オブジェクト指向プログラミングの意味で開発者によって特化されることがあります。
- このチュートリアルでは、CsmaNetDevice、PointToPointNetDevice、WifiNetDeviceなど、NetDeviceの特化バージョンをいくつか使用します。
  - Ethernet NICがEthernetネットワークと連携するように、CsmaNetDeviceはCsmaChannelと連携するように設計され、PointToPointNetDeviceはPointToPointChannelと連携し、WifiNetDeviceはWifiChannelと連携するように設計されています。

#### Topology Helpers
- NetDevicesをノードに接続したり、NetDevicesをチャネルに接続したり、IPアドレスを割り当てたりするなどは、ns-3では非常に一般的なタスクです。
- そのため、これらをできるだけ簡単に行うために、私たちはこれを「トポロジヘルパー」と呼ばれるものを提供しています。
  - 例えば、NetDeviceを作成し、MACアドレスを追加し、そのネットデバイスをノードにインストールし、ノードのプロトコルスタックを設定し、NetDeviceをチャネルに接続するには、多くの異なるns-3の基本操作が必要かもしれません。
  - さらに、複数のデバイスをマルチポイントチャネルに接続し、個々のネットワークをインターネットワークに接続するにはさらに多くの操作が必要です。
- これらの多くの異なる操作を、利便性のために使いやすいモデルに組み合わせたトポロジヘルパーオブジェクトを提供しています。

### A First ns-3 Script
#### Copyright
#### Module includes
#### Ns3 Namespace
```
using namespace ns3;
```

#### Logging
```
NS_LOG_COMPONENT_DEFINE("FirstScriptExample");
```
#### Main Function

```
int
main(int argc, char* argv[])
{
    CommandLine cmd(__FILE__);
    cmd.Parse(argc, argv);
```
- 時間の解像度を設定
```
    Time::SetResolution(Time::NS);
```
- ログ設定
```
    LogComponentEnable("UdpEchoClientApplication", LOG_LEVEL_INFO);
    LogComponentEnable("UdpEchoServerApplication", LOG_LEVEL_INFO);
```

#### Topology Helpers
##### NodeContainer
```
    NodeContainer nodes;
    nodes.Create(2);
```
- 主要な抽象化の1つはノードです。これは、プロトコルスタック、アプリケーション、周辺カードなどを追加するコンピュータを表します。
- NodeContainerトポロジヘルパーは、シミュレーションを実行するために作成したNodeオブジェクトを作成、管理、アクセスする便利な方法を提供します。
- 最初の行は、NodeContainerを宣言していますが、これをnodesと呼んでいます。
- 2行目は、nodesオブジェクト上のCreateメソッドを呼び出し、コンテナに2つのノードを作成するように要求しています。
  - コンテナは内部的に2つのNodeオブジェクトを作成し、それらのオブジェクトへのポインタを格納します。
- スクリプト内のノードは、そのままでは何もしません。
- トポロジを構築する次のステップは、ノードをネットワークに接続することです。
- サポートする最も単純なネットワーク形態は、2つのノード間の単一のポイントツーポイントリンクです。ここで、そのようなリンクを構築します。

##### PointToPoint Helper

- リンクを組み立てる低レベルの作業を行うためにトポロジヘルパーオブジェクトを使用する。
- NetDeviceとChannelという2つの主要な抽象化があることを思い出してください。
- このスクリプトでns-3のPointToPointNetDeviceとPointToPointChannelオブジェクトを構成および接続するために、単一のPointToPointHelperを使用する。

```
    PointToPointHelper pointToPoint;
    pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
    pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));
```

- 最初の行でPointToPoint Helperのオブジェクトを作成
- 次の行では、PointToPointHelperオブジェクトに対して、PointToPointNetDevice オブジェクトを作成する際に「DataRate」を「5Mbps」（1秒あたり5メガビット）として使用するように指示しています。
  - 詳細な観点から見ると、文字列「DataRate」はPointToPointNetDeviceの「属性」と呼ばれるものに対応しています。
  - ns3::PointToPointNetDeviceクラスのDoxygenを見て、GetTypeIdメソッドのドキュメンテーションを見つけると、デバイス用に定義された属性のリストが表示されます。
  - その中に「DataRate」という属性も含まれています。
  - ほとんどのユーザーに見えるns-3オブジェクトには類似した属性のリストがあります。
  - このメカニズムを使用して、後続のセクションで見るように、再コンパイルせずにシミュレーションを簡単に設定できます。
- PointToPointNetDevice の「DataRate」に類似して、PointToPointChannel には「Delay」属性が関連付けられています。
- 最後の行では、PointToPointHelperに対して、その後に作成するすべてのポイントツーポイントチャネルの伝搬遅延値として「2ms」（2ミリ秒）を使用するよう指示しています。


##### NetDeviceContainer

- このスクリプトのこの時点で、2つのノードを含むNodeContainerがあります。
- PointToPointHelperは、PointToPointNetDeviceを作成し、それらの間にPointToPointChannelオブジェクトを接続するために準備が整っています。
- シミュレーションのノードを作成するためにNodeContainerトポロジヘルパーオブジェクトを使用したように、PointToPointHelperにデバイスを作成、設定、インストールする作業を行うように指示します。
- 作成されるすべてのNetDeviceオブジェクトのリストを持っている必要があるため、ノードを保持するためにNodeContainerを使用したのと同様に、NetDeviceを保持するためにNetDeviceContainerを使用します。

```
NetDeviceContainer devices;
devices = pointToPoint.Install(nodes);
```
- このコードはデバイスとチャネルの設定を完了します。
- 最初の行は上記で言及したデバイスコンテナを宣言し、2行目は重要な作業を行います。
- PointToPointHelperのInstallメソッドはNodeContainerをパラメーターとして受け取ります
- 内部的には、NetDeviceContainerが作成されます。
- NodeContainer内の各ノード（ポイントツーポイントリンクの場合、ちょうど2つ必要です）に対してPointToPointNetDeviceが作成され、デバイスコンテナに保存されます
- PointToPointChannelが作成され、2つのPointToPointNetDeviceが接続されます
- PointToPointHelperによってオブジェクトが作成されると、ヘルパーに以前に設定された属性が作成されたオブジェクト内の対応する属性を初期化するために使用されます。
- pointToPoint.Install(nodes)の呼び出しを実行した後、2つのノードがあり、それぞれにインストールされたポイントツーポイントネットデバイスと、それらの間に単一のポイントツーポイントチャネルがあります。
- 両方のデバイスは、2ミリ秒の伝送遅延を持つチャネルを介して5メガビット毎秒でデータを送信するように設定されます。

##### InternetStackHelper

- ノードとデバイスが設定されましたが、ノードにはまだプロトコルスタックがインストールされていません。次の2行のコードがそれを処理します。

```
InternetStackHelper stack;
stack.Install(nodes);

```
- InternetStackHelperは、ポイントツーポイントネットデバイスに対するPointToPointHelperのように、インターネットスタックに対するトポロジヘルパーです。
- InstallメソッドはNodeContainerをパラメーターとして受け取ります。
- 実行されると、ノードコンテナ内の各ノードにインターネットスタック（TCP、UDP、IPなど）をインストールします。

##### Ipv4AddressHelper

- 次に、ノード上のデバイスをIPアドレスに関連付ける必要があります。
- IPアドレスの割り当てを管理するトポロジヘルパーを提供します。
- ユーザーが見えるAPIは、実際のアドレス割り当てを実行する際に使用するベースIPアドレスとネットワークマスクを設定することだけです（これはヘルパー内部で低レベルで行われます）。

```
    Ipv4AddressHelper address;
    address.SetBase("10.1.1.0", "255.255.255.0");
```

- このコードは、まずアドレスヘルパーオブジェクトを宣言し、ネットワーク10.1.1.0およびマスク255.255.255.0を使用してIPアドレスを割り当てるためのベースを指定します。
- デフォルトでは、割り当てられたアドレスは1から始まり、単調に増加します。
- したがって、このベースから割り当てられる最初のアドレスは10.1.1.1であり、それに続いて10.1.1.2などが続きます。
- 実際には、ns-3の低レベルシステムは割り当てられたすべてのIPアドレスを記憶しており、同じアドレスが誤って2回生成されると致命的なエラーを生成します（とてもデバッグが難しいエラーです）。

```
Ipv4InterfaceContainer interfaces = address.Assign(devices);
```
- このコード行、は実際のアドレス割り当てを実行します。
- ns-3では、IPアドレスとデバイスの関連付けをIpv4Interfaceオブジェクトを使用して行います。
- ヘルパーによって作成されたネットデバイスのリストが将来の参照のために必要なことがあるように、Ipv4Interfaceオブジェクトのリストが必要な場合もあります。
- Ipv4InterfaceContainerはこの機能を提供します。
- これで、スタックがインストールされ、IPアドレスが割り当てられたポイントツーポイントネットワークが構築されました。
- この時点で必要なのはトラフィックを生成するアプリケーションです。

#### Applications

- ns-3システムのもう一つの中核的な抽象化の1つがApplicationです。
- このスクリプトでは、UdpEchoServerApplicationとUdpEchoClientApplicationという、ns-3の核となるクラスApplicationの2つの特殊化を使用しています。
- 以前の説明と同様に、基礎となるオブジェクトを構成および管理するのにヘルパーオブジェクトを使用します。
- ここでは、UdpEchoServerHelperとUdpEchoClientHelperオブジェクトを使用して、作業を簡略化します。

##### UdpEchoServerHelpe
以下のコードは、前に作成したノードの1つにUDPエコーサーバーアプリケーションをセットアップするために使用されます。

```
UdpEchoServerHelper echoServer(9); // Port 9

ApplicationContainer serverApps = echoServer.Install(nodes.Get(1)); // Node #1 にAppをインストール
serverApps.Start(Seconds(1.0));
serverApps.Stop(Seconds(10.0));
```

- 上記のスニペットの最初のコード行は、UdpEchoServerHelperを宣言しています。
  - 通常通り、これはアプリケーションそのものではなく、実際のアプリケーションを作成するのに使用するオブジェクトです。
  - 私たちの慣習の1つは、ヘルパーコンストラクタに必要な属性を配置することです。
  - この場合、ヘルパーはクライアントも知っているポート番号が提供されない限り、有用なことは何もできません。
  - 単に適当なポート番号を選んでうまくいくことを期待するのではなく、コンストラクタにパラメータとしてポート番号を指定する必要があります。
  - コンストラクタは、渡された値でSetAttributeを単純に実行します。
  - 必要であれば、後でSetAttributeを使用して「Port」属性を別の値に設定することもできます。
- 多くの他のヘルパーオブジェクトと同様に、UdpEchoServerHelperオブジェクトにはInstallメソッドがあります。
  - 実際には、このメソッドの実行によって基盤となるエコーサーバーアプリケーションがインスタンス化され、ノードにアタッチされます。興味深いことに、Installメソッドは、私たちが見てきた他のInstallメソッドと同様に、NodeContainerをパラメータとして受け取ります。
  - 実際には、この場合でもそう見えないものの、nodes.Get(1)の結果（ノードオブジェクトへのスマートポインタであるPtr<Node>を返す）が使用され、名前のないNodeContainerのコンストラクタに渡され、その後Installに渡されます。
  - C++コード内で特定のメソッドシグネチャを見つけるのに迷った場合は、この種の暗黙の変換を探してみてください。
- これにより、echoServer.Installが、ノードを管理するために使用したNodeContainer内のインデックス番号1のノードにUdpEchoServerApplicationをインストールすることがわかります。
  - Installは、ヘルパーによって作成されたすべてのアプリケーションへのポインタを保持するコンテナを返します（この場合、1つのアプリケーションを含むNodeContainerを渡したので、1つのアプリケーションが返されます）。
- アプリケーションはトラフィックを生成するための「開始」時間と、オプションの「停止」時間が必要です。
  - これらの時間は、ApplicationContainerのStartおよびStopメソッドを使用して設定されます。
  - これらのメソッドはTimeパラメータを受け取ります。
  - この場合、C++のdouble型1.0をSecondsキャストを使用してns-3 Timeオブジェクトに変換するための明示的なC++変換シーケンスを使用しています。
  - 変換ルールはモデル作成者によって制御される可能性があり、C++には独自のルールがあるため、パラメータが自動的に変換されることを常に前提とすることはできません。


```
serverApps.Start(Seconds(1.0));
serverApps.Stop(Seconds(10.0));
```

  - これにより、エコーサーバーアプリケーションはシミュレーションの1秒後に開始（有効化）し、シミュレーションの10秒後に停止（無効化）します。
  - アプリケーションの停止イベントを10秒で宣言したことから、シミュレーションは少なくとも10秒間続行されます。

##### UdpEchoClientHelper

- エコークライアントアプリケーションは、サーバーの場合と大部分が類似した方法で設定されます。
- UdpEchoClientHelperによって管理される基本的なUdpEchoClientApplicationが存在します。

- 以下のコードでは、エコークライアントを設定していますが、サーバーの場合と同様の手順です。
```
    UdpEchoClientHelper echoClient(interfaces.GetAddress(1), 9); // Remote Server (Node #1) の IP アドレス、Port 9
    echoClient.SetAttribute("MaxPackets", UintegerValue(1));
    echoClient.SetAttribute("Interval", TimeValue(Seconds(1.0)));
    echoClient.SetAttribute("PacketSize", UintegerValue(1024));

    ApplicationContainer clientApps = echoClient.Install(nodes.Get(0)); // Node #0 に App をインストール
    clientApps.Start(Seconds(2.0));
    clientApps.Stop(Seconds(10.0));
```
- 最初にUdpEchoClientHelperを宣言し、次に必要な5つの異なる属性を設定します。
- 最初の2つの属性は、UdpEchoClientHelperの構築中に設定されます。
- ヘルパーのコンストラクタにパラメータを渡し、ヘルパー内部で「RemoteAddress」と「RemotePort」属性を設定するための慣例に従っています。

- Ipv4InterfaceContainerを使用して、デバイスに割り当てたIPアドレスを追跡しました。
  - interfacesコンテナの最初のインターフェースは、nodesコンテナの最初のノードに割り当てられたIPアドレスに対応します。
  - したがって、上記のコードの最初の行では、ヘルパーを作成し、クライアントのリモートアドレスを、サーバーが存在するノードに割り当てられたIPアドレスに設定するように指定しています。
- また、ポート9にパケットを送信するように指示しています。
- 
- 「MaxPackets」属性は、シミュレーション中にクライアントが送信できる最大パケット数をクライアントに伝えます。
- 「Interval」属性は、クライアントにパケット間の待機時間を伝え、 「PacketSize」属性は、パケットのペイロードサイズをクライアントに伝えます。
- 特定の属性の組み合わせでは、クライアントに1つの1024バイトのパケットを送信するよう指示しています。

- エコーサーバーの場合と同様に、エコークライアントにStartとStopを指示しますが、ここではクライアントをサーバーが有効になった1秒後（シミュレーションの2秒目）に開始します。

#### Simulator
##### 起動
この段階で、実際にシミュレーションを実行する必要があります。これは、グローバル関数Simulator::Runを使用して行います。
```
Simulator::Run();
```

##### シミュレーションの流れ
- 次のメソッドを呼び出し、1.0秒、2.0秒、10.0秒の3つの時刻に予約されたイベントをシミュレータにスケジュールした。

```
serverApps.Start(Seconds(1.0));
serverApps.Stop(Seconds(10.0));
...
clientApps.Start(Seconds(2.0));
clientApps.Stop(Seconds(10.0));
```

- Simulator::Runが呼び出されると、システムは予約されたイベントのリストを調べて実行を開始します。
- まず、1.0秒のイベントが実行され、エコーサーバーアプリケーションが有効になります（このイベントはさらに多くのイベントを予約する可能性があります）。
- その後、t=2.0秒に予約されたイベントが実行され、エコークライアントアプリケーションが開始されます。
- 再び、このイベントはさらに多くのイベントを予約するかもしれません。
- エコークライアントアプリケーションの開始イベントの実装により、パケットをサーバーに送信することでシミュレーションのデータ転送フェーズが開始されます。

- サーバーにパケットを送信する行為は、裏で自動的に予約される一連のイベントの連鎖をトリガーし、スクリプトで設定したさまざまなタイミングパラメータに従ってパケットエコーのメカニズムを実行します。

- 最終的に、1つのパケットしか送信しない（MaxPackets属性が1に設定されていることを思い出してください）クライアントのエコーリクエストによってトリガーされるイベント連鎖は終息し、シミュレーションはアイドル状態になります。
- これが発生したら、残りのイベントはサーバーとクライアントの停止イベントです。これらのイベントが実行されると、さらに処理するイベントがなくなり、Simulator::Runが戻ります。シミュレーションは完了です。

##### クリーンアップ
- 残る作業はクリーンアップです。
- これは、グローバル関数Simulator::Destroyを呼び出すことで行います。
  - ヘルパー関数（または低レベルのns-3コード）が実行されると、作成されたすべてのオブジェクトを破棄するためのフックがシミュレータに挿入されるようになっています。
  - あなたはこれらのオブジェクトを自分で追跡する必要はありませんでした。
  - Simulator::Destroyを呼び出して終了するだけ。

```
Simulator::Destroy();
return 0;
```

### Tweaking
#### ログモジュールの使用

ns-3のログモジュールについては、最初の.ccスクリプトを説明する際に簡単に説明しましたが、今度はより詳しく見て、ログサブシステムがどのようなユースケースをカバーするために設計されたかを見てみましょう。

### ログの概要

- 多くの大規模なシステムは、メッセージログの機能をサポートしており、ns-3も例外ではありません。一部の場合では、エラーメッセージのみが「オペレーターコンソール」に記録されます（通常、Unixベースのシステムではstderrです）。他のシステムでは、警告メッセージだけでなく、詳細な情報メッセージも出力されることがあります。一部の場合では、デバッグメッセージを出力するためにログ設備が使用され、出力をすぐに見づらくすることがあります。

- ns-3は、これらの詳細度レベルがすべて役立つと考えており、メッセージログへの **選択可能なマルチレベルアプローチを提供** しています。
  - ログを完全に無効にすることも、コンポーネントごとに有効にすることも、グローバルに有効にすることもでき、選択可能な冗長度レベルを提供します。
  - ns-3のログモジュールは、シミュレーションから有用な情報を比較的簡単に取得するためのわかりやすい方法を提供しています。

- シミュレーションからデータを取得するための一般的なメカニズムであるトレース（トレーシング）も提供していますが、 **シミュレーション出力にはトレースを優先すべきです**（トレーシングシステムの詳細については、トレーシングシステムの使用チュートリアルセクションを参照してください）。
  - ログは、デバッグ情報、警告、エラーメッセージ、またはスクリプトやモデルから簡単にメッセージを取得したい場合に使用すべきです。

- 現在、システムで定義されている増加する冗長性の7つのログメッセージレベルがあります。
  - LOG_ERROR — エラーメッセージを記録します（関連するマクロ: NS_LOG_ERROR）。
  - LOG_WARN — 警告メッセージを記録します（関連するマクロ: NS_LOG_WARN）。
  - LOG_DEBUG — 比較的まれなアドホックなデバッグメッセージを記録します（関連するマクロ: NS_LOG_DEBUG）。
  - LOG_INFO — プログラムの進行状況に関する情報メッセージを記録します（関連するマクロ: NS_LOG_INFO）。
  - LOG_FUNCTION — 呼び出される各関数を説明するメッセージを記録します（メンバー関数用の関連するマクロであるNS_LOG_FUNCTIONと、静的関数用のNS_LOG_FUNCTION_NOARGSの2つの関連するマクロ）。
  - LOG_LOGIC – 関数内の論理フローを説明するメッセージを記録します（関連するマクロ: NS_LOG_LOGIC）。
  - LOG_ALL — 上記で説明したすべてを記録します（関連するマクロはありません）。

各 LOG_TYPE には LOG_LEVEL_TYPE もあり、これを使用すると、そのレベルに加えてその上のすべてのレベルのロギングが有効になります。(この結果、LOG_ERROR と LOG_LEVEL_ERROR、および LOG_ALL と LOG_LEVEL_ALL は機能的に同等です。) たとえば、LOG_INFO を有効にすると、NS_LOG_INFO マクロによって提供されるメッセージのみが有効になりますが、LOG_LEVEL_INFO を有効にすると、NS_LOG_DEBUG、NS_LOG_WARN、および NS_LOG_ERROR マクロによって提供されるメッセージも有効になります。

また、ログ レベルやコンポーネントの選択に関係なく、常に表示される無条件のログ マクロも提供します。
```
    NS_LOG_UNCOND – 関連するメッセージを無条件にログに記録します (関連するログ レベルはありません)。
```

各レベルは、単独または累積的にリクエストできます。また、ログ記録は、シェル環境変数 (NS_LOG) を使用するか、ログ記録システム関数呼び出しによって設定できます。チュートリアルの前半で説明したように、ロギング システムには Doxygen のドキュメントがあり、まだロギング モジュールのドキュメントを熟読していない場合は、この機会に熟読してください。

ドキュメントを詳しく読んだので、その知識を活用して、scratch/myfirst.ccすでに構築したサンプル スクリプトから興味深い情報を取得してみましょう。

### 
- 実行すると、次のログが出る。
```
$ ./ns3 run scratch/myfirst

At time +2s client sent 1024 bytes to 10.1.1.2 port 9
At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
```

UdpEchoClientApplication上に表示される「送信」メッセージと「受信」メッセージは、実際にはとからのメッセージを記録していることがわかりますUdpEchoServerApplication。たとえば、NS_LOG 環境変数を介してログ レベルを設定することで、クライアント アプリケーションに詳細情報を出力するように依頼できます。

ここからは、「VARIABLE=value」構文を使用する sh に似たシェルを使用していると仮定します。csh のようなシェルを使用している場合は、私の例をそれらのシェルで必要な「setenv VARIABLE value」構文に変換する必要があります。

最初は LOG_LEVEL_INFOログのレベルが有効になっている。

```
LogComponentEnable("UdpEchoClientApplication", LOG_LEVEL_INFO);
```

- NS_LOG 環境変数を設定することで、スクリプトを変更したり再コンパイルしたりすることなく、ログ レベルを上げてより多くの情報を取得できる。

```
$ export NS_LOG=UdpEchoClientApplication=level_all
```
実行結果
```
UdpEchoClientApplication:UdpEchoClient(0xef90d0)
UdpEchoClientApplication:SetDataSize(0xef90d0, 1024)
UdpEchoClientApplication:StartApplication(0xef90d0)
UdpEchoClientApplication:ScheduleTransmit(0xef90d0, +0ns)
UdpEchoClientApplication:Send(0xef90d0)
At time +2s client sent 1024 bytes to 10.1.1.2 port 9
At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
UdpEchoClientApplication:HandleRead(0xef90d0, 0xee7b20)
At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
UdpEchoClientApplication:StopApplication(0xef90d0)
UdpEchoClientApplication:DoDispose(0xef90d0)
UdpEchoClientApplication:~UdpEchoClient(0xef90d0)
```