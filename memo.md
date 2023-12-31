# ns-3 メモ


## DCQCN
https://github.com/bobzhuyb/ns3-rdma

### What did we add exactly?

- point-to-point/model/qbb-net-device.cc and all other qbb-* files:
  - **DCQCN and PFC implementation**. It also includes go-back-to-N and go-back-to-0 that handle packet drop due to corruption.
  - In 2013, we got a very basic NS-3 PFC implementation somewhere, and developed based on it. We cannot find the original repository anymore.
- network/model/broadcom-node.cc and .h:
  - This implements a **Broadcom ASIC switch model**, which is mostly doing all kinds of buffer threshold-related operations. These include deciding whether PFC should be triggered, ECN should be marked, buffer is too full so packets should be dropped, etc. It supports both static and dynamic thresholds for PFC.
  - Disclaim: this module is purely based on authors' personal understanding of Broadcom ASIC. It does not reflect any official confirmation from either Microsoft or Broadcom.
- network/utils/broadcom-egress-queue.cc and .h:
  - This is the actual **MMU buffering packets**. It also includes switch scheduler, i.e., when upper layer ask for a packet to send, it will decide which queue to be dequeued. Strategies like strict priority and round robin are supported.
- applications/model/udp-echo-client.cc:
  - We implement the RDMA client here, which aligns with the fact that RoCEv2 includes UDP header. In particular, original UDP client has troubles when PFC pause the link. Original UDP client keeps sending packets at line rate, soon it builds up huge queue and memory runs out. Here we throttle the sending rate if it gets pushed back by PFC.
- internet/model/seq-ts-header.cc and .h:
  - We didn't implement the full InfiniBand header. Instead, what we really need is just the sequence number (for detecting corruption drops, and also help us understand the throughput) and timestamp (required by TIMELY.) This is where we encode this information into packets.
- main/third.cc:
  - The main() function.
  - There may be other edits here and there, especially the trace generation is scattered among various network stacks. But above are the major ones.


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

- 上の例だと “Received 1024 bytes from 10.1.1.2” がどこから出てきたのかわからない。
- 次のように`prefix_func`をOR条件で指定して解決する。(クオートが必要)
```
$ export 'NS_LOG=UdpEchoClientApplication=level_all|prefix_func'
```
実行すると、指定されたログコンポーネントからのすべてのメッセージにコンポーネント名がプレフィックスとして付けられていることを確認できる。
```
UdpEchoClientApplication:UdpEchoClient(0xea8e50)
UdpEchoClientApplication:SetDataSize(0xea8e50, 1024)
UdpEchoClientApplication:StartApplication(0xea8e50)
UdpEchoClientApplication:ScheduleTransmit(0xea8e50, +0ns)
UdpEchoClientApplication:Send(0xea8e50)
UdpEchoClientApplication:Send(): At time +2s client sent 1024 bytes to 10.1.1.2 port 9
At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
UdpEchoClientApplication:HandleRead(0xea8e50, 0xea5b20)
UdpEchoClientApplication:HandleRead(): At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
UdpEchoClientApplication:StopApplication(0xea8e50)
UdpEchoClientApplication:DoDispose(0xea8e50)
UdpEchoClientApplication:~UdpEchoClient(0xea8e50)
```
- これで、UDP エコー クライアント アプリケーションからのすべてのメッセージがそのように識別されることがわかります。「10.1.1.2 から 1024 バイトを受信しました」というメッセージが、エコー クライアント アプリケーションからのものであることが明確に識別されるようになりました。

- 残りのメッセージはUDP echo server applicationのものと思われる。次のようにコロンでセパレートして設定する。

```
$ export 'NS_LOG=UdpEchoClientApplication=level_all|prefix_func:UdpEchoServerApplication=level_all|prefix_func'
```
実行結果。Serverのメッセージにもプレフィックスがつけられる。
```
UdpEchoServerApplication:UdpEchoServer(0x2101590)
UdpEchoClientApplication:UdpEchoClient(0x2101820)
UdpEchoClientApplication:SetDataSize(0x2101820, 1024)
UdpEchoServerApplication:StartApplication(0x2101590)
UdpEchoClientApplication:StartApplication(0x2101820)
UdpEchoClientApplication:ScheduleTransmit(0x2101820, +0ns)
UdpEchoClientApplication:Send(0x2101820)
UdpEchoClientApplication:Send(): At time +2s client sent 1024 bytes to 10.1.1.2 port 9
UdpEchoServerApplication:HandleRead(0x2101590, 0x2106240)
UdpEchoServerApplication:HandleRead(): At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
UdpEchoServerApplication:HandleRead(): Echoing packet
UdpEchoServerApplication:HandleRead(): At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
UdpEchoClientApplication:HandleRead(0x2101820, 0x21134b0)
UdpEchoClientApplication:HandleRead(): At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
UdpEchoClientApplication:StopApplication(0x2101820)
UdpEchoServerApplication:StopApplication(0x2101590)
UdpEchoClientApplication:DoDispose(0x2101820)
UdpEchoServerApplication:DoDispose(0x2101590)
UdpEchoClientApplication:~UdpEchoClient(0x2101820)
UdpEchoServerApplication:~UdpEchoServer(0x2101590)
```
- prefix_time を指定すると、時刻を出力できる。
```
$ export 'NS_LOG=UdpEchoClientApplication=level_all|prefix_func|prefix_time:UdpEchoServerApplication=level_all|prefix_func|prefix_time'
```
実行結果
```
+0.000000000s UdpEchoServerApplication:UdpEchoServer(0x8edfc0)
+0.000000000s UdpEchoClientApplication:UdpEchoClient(0x8ee210)
+0.000000000s UdpEchoClientApplication:SetDataSize(0x8ee210, 1024)
+1.000000000s UdpEchoServerApplication:StartApplication(0x8edfc0)
+2.000000000s UdpEchoClientApplication:StartApplication(0x8ee210)
+2.000000000s UdpEchoClientApplication:ScheduleTransmit(0x8ee210, +0ns)
+2.000000000s UdpEchoClientApplication:Send(0x8ee210)
+2.000000000s UdpEchoClientApplication:Send(): At time +2s client sent 1024 bytes to 10.1.1.2 port 9
+2.003686400s UdpEchoServerApplication:HandleRead(0x8edfc0, 0x936770)
+2.003686400s UdpEchoServerApplication:HandleRead(): At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
+2.003686400s UdpEchoServerApplication:HandleRead(): Echoing packet
+2.003686400s UdpEchoServerApplication:HandleRead(): At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
+2.007372800s UdpEchoClientApplication:HandleRead(0x8ee210, 0x8f3140)
+2.007372800s UdpEchoClientApplication:HandleRead(): At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
+10.000000000s UdpEchoClientApplication:StopApplication(0x8ee210)
+10.000000000s UdpEchoServerApplication:StopApplication(0x8edfc0)
UdpEchoClientApplication:DoDispose(0x8ee210)
UdpEchoServerApplication:DoDispose(0x8edfc0)
UdpEchoClientApplication:~UdpEchoClient(0x8ee210)
UdpEchoServerApplication:~UdpEchoServer(0x8edfc0)
```
- システム内のすべてのロギング コンポーネントをオンにするには、NS_LOG変数を次のように設定する。
```
$ export 'NS_LOG=*=level_all|prefix_func|prefix_time'
```
- アスタリスクは、ロギング コンポーネントのワイルドカード
- ログは大量に出力されるので、必要に応じて、ファイルにリダイレクトする。
```
$ ./ns3 run scratch/myfirst > log.out 2>&1
```
- 問題が発生し、どこで問題が発生しているのかわからないときに、この非常に詳細なバージョンのロギングを使用しています。
  - ブレークポイントを設定したり、デバッガーでコードをステップ実行したりする必要がなく、コードの進行状況を非常に簡単に追跡できます。お気に入りのエディターで出力を編集し、期待するものを探し回ったり、予想外のことが起こったりするだけです。
- 何が問題なのかについて大まかなアイデアが得られたら、デバッガーに移行して問題を詳細に調査します。
- この種の出力は、スクリプトがまったく予期しない動作をする場合に特に役立ちます。
  - デバッガーを使用してステップ実行している場合、予期しないエクスカーションを完全に見逃してしまう可能性があります。
  - エクスカーションを記録すると、すぐに確認できるようになります。

### コードへのロギングの追加

スクリプトのロギングコンポーネントを以下のように設定していた。
```
NS_LOG_COMPONENT_DEFINE("FirstScriptExample");
```
ファイルに次の行を追加する。
```
NS_LOG_INFO("Creating Topology");
```

次に、ns3 を使用してスクリプトを構築し、NS_LOG変数をクリアして、前に有効にしたログのトレントをオフにします。
```
$ ./ns3
$ export NS_LOG=""
```
ここで、スクリプトを実行すると、
```
$ ./ns3 run scratch/myfirst
```
ログ コンポーネント ( FirstScriptExample) が有効になっていないため、メッセージが表示されない。
次のコマンドで有効化する
```
$ export NS_LOG=FirstScriptExample=info
```
ここでスクリプトを実行すると、新しい「Creating Topology」ログ メッセージが表示されます。
```
Creating Topology
At time +2s client sent 1024 bytes to 10.1.1.2 port 9
At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
```

## コマンドライン引数の使用
### デフォルト属性の上書き

- コマンドライン引数を解析し、それらの引数に基づいてローカル変数とグローバル変数を自動的に設定するメカニズムを提供します。
- コマンド ライン引数システムを使用する最初の手順は、コマンド ライン パーサーを宣言することです。これは、次のコードのように非常に簡単に (メイン プログラム内で) 行われます。
```
int
main(int argc, char *argv[])
{
  ...

  CommandLine cmd;
  cmd.Parse(argc, argv);

  ...
}
```
- 次の方法でスクリプトにヘルプを求めます。
```
$ ./ns3 run "scratch/myfirst --PrintHelp"
```
- 次のように応答します。

```
myfirst [General Arguments]
General Arguments:
  --PrintGlobals:              Print the list of globals.
  --PrintGroups:               Print the list of groups.
  --PrintGroup=[group]:        Print all TypeIds of group.
  --PrintTypeIds:              Print all TypeIds.
  --PrintAttributes=[typeid]:  Print all attributes of typeid.
  --PrintVersion:              Print the ns-3 version.
  --PrintHelp:                 Print this help message.
```

- --PrintAttributes。オプションに焦点を当てる。
```
PointToPointHelper pointToPoint;
pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));
```
そして、それはDataRate実際には の であるとAttribute述べましたPointToPointNetDevice。Attributesコマンド ライン引数パーサーを使用して、 PointToPointNetDevice を見てみましょう。ヘルプ リストには、 を提供する必要があると記載されていますTypeId。が所属するクラスのクラス名に相当しますAttributes。この場合は となりますns3::PointToPointNetDevice。先に進んで入力してみましょう。
```
$ ./ns3 run "scratch/myfirst --PrintAttributes=ns3::PointToPointNetDevice"
```

```
--ns3::PointToPointNetDevice::DataRate=[32768bps]:
  The default data rate for point to point links
```
- これは、システムでPointToPointNetDevice が作成されるときに使用されるデフォルト値。
- このデフォルトをAttribute設定でオーバーライドしました。
- SetDeviceAttributeコールを削除して、ポイントツーポイント デバイスとチャネルのデフォルト値を使用する。
```
...

NodeContainer nodes;
nodes.Create(2);

PointToPointHelper pointToPoint;

NetDeviceContainer devices;
devices = pointToPoint.Install(nodes);

...
```
- 実行すると、時刻が変化する。(略)
- コマンドラインを使用してDataRateものを指定すると、シミュレーションを再び高速化できます
```
$ ./ns3 run "scratch/myfirst --ns3::PointToPointNetDevice::DataRate=5Mbps"
```
Ｃｈａｎｎｅｌのレイテンシのデフォルト
```
$ ./ns3 run "scratch/myfirst --PrintAttributes=ns3::PointToPointChannel"
```
Delay Attributeチャネルが次のように設定されていることがわかります。
```
--ns3::PointToPointChannel::Delay=[0ns]:
  Transmission delay through the channel
```
コマンド ライン システムを使用して、これらのデフォルト値を両方とも設定できます。
```
$ ./ns3 run "scratch/myfirst
  --ns3::PointToPointNetDevice::DataRate=5Mbps
  --ns3::PointToPointChannel::Delay=2ms"
```
この場合、スクリプト内でDataRateと を明示的に設定したときのタイミングを回復します。Delay
```
+0.000000000s UdpEchoServerApplication:UdpEchoServer(0x1df20f0)
+1.000000000s UdpEchoServerApplication:StartApplication(0x1df20f0)
At time +2s client sent 1024 bytes to 10.1.1.2 port 9
+2.003686400s UdpEchoServerApplication:HandleRead(0x1df20f0, 0x1de0250)
+2.003686400s At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
+2.003686400s Echoing packet
+2.003686400s At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
+10.000000000s UdpEchoServerApplication:StopApplication(0x1df20f0)
UdpEchoServerApplication:DoDispose(0x1df20f0)
UdpEchoServerApplication:~UdpEchoServer(0x1df20f0)
```
MaxPackets
```
$ ./ns3 run "scratch/myfirst
  --ns3::PointToPointNetDevice::DataRate=5Mbps
  --ns3::PointToPointChannel::Delay=2ms
  --ns3::UdpEchoClient::MaxPackets=2"
```
この時点で当然の疑問は、これらすべての属性の存在をどのようにして知るかということです。繰り返しますが、コマンド ライン ヘルプ機能にはこれに関する機能があります。コマンドラインのヘルプを求めると、以下が表示されるはずです。
```
$ ./ns3 run "scratch/myfirst --PrintHelp"
myfirst [General Arguments]

General Arguments:
  --PrintGlobals:              Print the list of globals.
  --PrintGroups:               Print the list of groups.
  --PrintGroup=[group]:        Print all TypeIds of group.
  --PrintTypeIds:              Print all TypeIds.
  --PrintAttributes=[typeid]:  Print all attributes of typeid.
  --PrintVersion:              Print the ns-3 version.
  --PrintHelp:                 Print this help message.
```
「PrintGroups」引数を選択すると、登録されているすべての TypeId グループのリストが表示されます。グループ名は、ソース ディレクトリ内のモジュール名と一致します (先頭が大文字になります)。すべての情報を一度に印刷すると多すぎるため、グループごとに情報を印刷するための追加のフィルターを使用できます。そこで、再びポイントツーポイント モジュールに焦点を当てます。
```
./ns3 run "scratch/myfirst --PrintGroup=PointToPoint"
TypeIds in group PointToPoint:
  ns3::PointToPointChannel
  ns3::PointToPointNetDevice
  ns3::PppHeader
```
ここから、--PrintAttributes=ns3::PointToPointChannel上記の例のように、属性を検索するための可能な TypeId 名を見つけることができます。

属性について調べるもう 1 つの方法は、ns-3 Doxygen を使用することです。シミュレータに登録されているすべての属性をリストするページがあります。


### 独自の値をフックする

- 独自のフックをコマンド ライン システムに追加することもできます。
- この機能を使用して、別の方法でエコーするパケットの数を指定してみましょう。
- nPackets関数に呼び出されるローカル変数を追加しましょう
  - 以前のデフォルトの動作と一致するように、これを 1 に初期化します。
  - コマンドラインパーサーがこの値を変更できるようにするには、値をパーサーにフックする必要があります。
  - これを行うには、 AddValueへの呼び出しを追加します。
```
int
main(int argc, char *argv[])
{
  uint32_t nPackets = 1;

  CommandLine cmd;
  cmd.AddValue("nPackets", "Number of packets to echo", nPackets);
  cmd.Parse(argc, argv);

  ...
```
- ス以下に示すように定数ではなくnPacketsをMaxPackets Attribute変数に設定されるように変更します。
```
echoClient.SetAttribute("MaxPackets", UintegerValue(nPackets));
```
- ここで、スクリプトを実行して--PrintHelp引数を指定すると、ヘルプ表示に新しいリストが表示されるはずです。
```
$ ./ns3 build
$ ./ns3 run "scratch/myfirst --PrintHelp"

[Program Options] [General Arguments]

Program Options:
  --nPackets:  Number of packets to echo [1]

General Arguments:
  --PrintGlobals:              Print the list of globals.
  --PrintGroups:               Print the list of groups.
  --PrintGroup=[group]:        Print all TypeIds of group.
  --PrintTypeIds:              Print all TypeIds.
  --PrintAttributes=[typeid]:  Print all attributes of typeid.
  --PrintVersion:              Print the ns-3 version.
  --PrintHelp:                 Print this help message.
```
--nPacketsエコーするパケットの数を指定したい場合は、コマンドラインで引数を設定することで指定できるようになりました。
```
$ ./ns3 run "scratch/myfirst --nPackets=2"
```
実行結果
```
+0.000000000s UdpEchoServerApplication:UdpEchoServer(0x836e50)
+1.000000000s UdpEchoServerApplication:StartApplication(0x836e50)
At time +2s client sent 1024 bytes to 10.1.1.2 port 9
+2.003686400s UdpEchoServerApplication:HandleRead(0x836e50, 0x8450c0)
+2.003686400s At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
+2.003686400s Echoing packet
+2.003686400s At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
At time +3s client sent 1024 bytes to 10.1.1.2 port 9
+3.003686400s UdpEchoServerApplication:HandleRead(0x836e50, 0x8450c0)
+3.003686400s At time +3.00369s server received 1024 bytes from 10.1.1.1 port 49153
+3.003686400s Echoing packet
+3.003686400s At time +3.00369s server sent 1024 bytes to 10.1.1.1 port 49153
At time +3.00737s client received 1024 bytes from 10.1.1.2 port 9
+10.000000000s UdpEchoServerApplication:StopApplication(0x836e50)
UdpEchoServerApplication:DoDispose(0x836e50)
UdpEchoServerApplication:~UdpEchoServer(0x836e50)
```
- 2 つのパケットがエコーされました。

## トレースシステムの使用

- このチュートリアルでは、いくつかの事前定義されたソースとシンクについて説明し、ユーザーの労力をほとんどかけずにそれらをカスタマイズする方法を示します。
- トレース名前空間の拡張や新しいトレース ソースの作成など、高度なトレース設定の詳細については、ns-3 マニュアルまたはハウツー セクションを参照してください。

### ASCII トレース

ns-3 は、低レベルのトレース システムをラップするヘルパー機能を提供し、理解しやすいパケット トレースの構成に関する詳細を支援します。この機能を有効にすると、出力が ASCII ファイルで表示され、それが名前になります。ns-2 の出力に詳しい人にとって、このタイプのトレースはout.tr多くのスクリプトによって生成されるトレースに似ています。

- ASCII トレース出力をスクリプトに追加してみましょう
- Simulator::Run()への呼び出しの直前に、次のコード行を追加します。
```
AsciiTraceHelper ascii;
pointToPoint.EnableAsciiAll(ascii.CreateFileStream("myfirst.tr"));
```
- 他の多くのns-3イディオムと同様に、このコードはヘルパー オブジェクトを使用して ASCII トレースの作成を支援します。
- 2 行目には 2 つのネストされたメソッド呼び出しが含まれています。
- 「内部」メソッドは、CreateFileStream()イディオムを使用してスタック上にファイル ストリーム オブジェクト (オブジェクト名なし) を作成し、それを呼び出されたメソッドに渡します。
- これについては今後さらに詳しく説明しますが、この時点で知っておく必要があるのは、「myfirst.tr」という名前のファイルを表すオブジェクトを作成し、それを ns-3 に渡すことだけです。

- EnableAsciiAll()シミュレーション内のすべてのポイントツーポイント デバイスで ASCII トレースを有効にすることをヘルパーに伝えます
- そして、(提供された) トレース シンクにパケットの移動に関する情報を ASCII 形式で書き出すようにしたいとします。。

これで、スクリプトをビルドしてコマンド ラインから実行できるようになります。
```
$ ./ns3 run scratch/myfirst
```
- 実行すると、プログラムは myfirst.tr という名前のファイルを作成します。
- ns3 の動作方法により、ファイルはローカル ディレクトリには作成されず、デフォルトではリポジトリの最上位ディレクトリに作成されます。
- トレースの保存場所を制御したい場合は、--cwdns3 のオプションを使用してこれを指定できます。
- myfirst.trまだそれを行っていないため、リポジトリの最上位ディレクトリに移動し、お気に入りのエディタでASCII トレース ファイルを確認する必要があります。

#### アスキートレースの解析
略

### PCAP トレース

- pcap形式でトレース ファイルを作成することもできます.
- この形式を読み取って表示できる最も一般的なプログラムは、Wireshark (以前は Ethereal と呼ばれていました) です。
- このチュートリアルでは、tcpdump を使用して pcap トレースを表示することに重点を置きます。

- pcap トレースを有効にするために使用されるコードはワンライナーです。
```
pointToPoint.EnablePcapAll("myfirst");
```
先に追加したばかりの ASCII トレース コードの後に​​、このコード行を挿入しますscratch/myfirst.cc。「myfirst.pcap」などの文字列ではなく、「myfirst」という文字列のみを渡していることに注意してください。これは、パラメータが完全なファイル名ではなく接頭辞であるためです。ヘルパーは実際に、シミュレーション内のすべてのポイントツーポイント デバイスのトレース ファイルを作成します。ファイル名は、プレフィックス、ノード番号、デバイス番号、および「.pcap」サフィックスを使用して構築されます。

このスクリプト例では、最終的に「myfirst-0-0.pcap」および「myfirst-1-0.pcap」という名前のファイルが表示されます。これらは、それぞれノード 0-デバイス 0 およびノー​​ド 1-デバイス 0 の pcap トレースです。

pcap トレースを有効にするコード行を追加したら、通常の方法でスクリプトを実行できます。
```
$ ./ns3 run scratch/myfirst
```
- ディストリビューションの最上位ディレクトリを見ると、3 つのログ ファイルが表示されるはずです。
  - myfirst.trは、以前に調べた ASCII トレース ファイルです。
  - myfirst-0-0.pcapと は、myfirst-1-0.pcap生成したばかりの新しい pcap ファイルです。

#### tcpdump による出力の読み取り

6.3.2.1. tcpdump による出力の読み取り

この時点でpcap解析を行う最も簡単な方法は、 tcpdumpを使用してファイルを確認することです。
```
$ tcpdump -nn -tt -r myfirst-0-0.pcap
reading from file myfirst-0-0.pcap, link-type PPP (PPP)
2.000000 IP 10.1.1.1.49153 > 10.1.1.2.9: UDP, length 1024
2.514648 IP 10.1.1.2.9 > 10.1.1.1.49153: UDP, length 1024

tcpdump -nn -tt -r myfirst-1-0.pcap
reading from file myfirst-1-0.pcap, link-type PPP (PPP)
2.257324 IP 10.1.1.1.49153 > 10.1.1.2.9: UDP, length 1024
2.257324 IP 10.1.1.2.9 > 10.1.1.1.49153: UDP, length 1024
```
- (クライアント デバイスの) myfirst-0-0.pcap ダンプを見ると、エコー パケットがシミュレーション開始の 2 秒目に送信されていることがわかります。
- 2 番目のダンプ (myfirst-1-0.pcap) を見ると、パケットが 2.257324 秒で受信されていることがわかります。
- 2 番目のダンプでは 2.257324 秒でパケットがエコー バックされていることがわかり、最後に、最初のダンプでは 2.514648 秒でパケットがクライアントで受信されていることがわかります。

#### Wireshark による出力の読み取り

- Wireshark は、これらのトレース ファイルを表示するために使用できるグラフィカル ユーザー インターフェイスです。
- Wireshark を使用できる場合は、各トレース ファイルを開いて、パケット スニファを使用してパケットをキャプチャしたかのように内容を表示できます。
- 
# トポロジの構築
## バスネットワークトポロジの構築
CSMA

```
// Default Network Topology
//
//       10.1.1.0
// n0 -------------- n1   n2   n3   n4
//    point-to-point  |    |    |    |
//                    ================
//                      LAN 10.1.2.0

```

## モデル、属性、現実
シミュレーションを使用する場合、モデル化されているものとされていないものを正確に理解することが重要です。たとえば、前のセクションで使用されたCSMAデバイスとチャネルを、実際のイーサネットデバイスであるかのように考え、シミュレーション結果が実際のイーサネットで起こることを直接反映するものであると期待することは誤りです。これは事実ではありません。

モデルは、定義上、現実の抽象化です。シミュレーションスクリプトの作成者は、シミュレーション全体およびその構成要素の「精度範囲」と「適用範囲」を決定する責任が最終的にあります。

Csmaなどの場合、モデル化されていないものを比較的簡単に特定することができる場合があります。モデルの説明（csma.h）を読むことで、CSMAモデルには衝突検出がないことがわかり、その使用がシミュレーションにどれだけ適用可能か、または結果に含めるべき注意事項を決定することができます。他の場合では、購入できる現実と一致しない動作を簡単に設定できる場合があります。いくつかの具体例を調査し、シミュレーションで現実の範囲外に簡単に逸脱できる方法を調べることは価値があるでしょう。

ご覧のように、ns-3では、ユーザーがモデルの動作を簡単に変更できる属性（Attributes）が提供されています。CsmaNetDeviceの2つの属性（MtuおよびEncapsulationMode）を考えてみましょう。Mtu属性はデバイスへの最大転送単位（MTU）を示します。これは、デバイスが送信できる最大プロトコルデータユニット（PDU）のサイズです。

CsmaNetDeviceではMTUのデフォルト値は1500バイトです。このデフォルト値はRFC 894「Ethernet Networks上でIP Datagramを転送するための標準」で見つかった数値に対応しています。この数値は実際には10Base5（フルスペックEthernet）ネットワーク用の最大パケットサイズから派生しています

## Building a Wireless Network Topology

```
// Default Network Topology
//
//   Wifi 10.1.3.0
//                 AP
//  *    *    *    *
//  |    |    |    |    10.1.1.0
// n5   n6   n7   n0 -------------- n1   n2   n3   n4
//                   point-to-point  |    |    |    |
//                                   ================
//                                     LAN 10.1.2.0
```
