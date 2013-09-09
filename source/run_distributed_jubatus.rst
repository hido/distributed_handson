Jubatusを動かしてみる。
==========================

さっそく分散環境で動かしてみましょう。


AMI
-----------------

用意されたVMでは、EC2公式のubuntu 12.04 serverイメージに追加して、以下の様な設定がされています。

::

    ubuntu@[manager]:~$ history 
        1  sudo vi /etc/apt/sources.list
        2  sudo apt-get update
        3  sudo apt-get install build-essential git zookeeper rabbitmq-server python-pip
        4  git clone https://github.com/jubatus/jubatus-installer.git
        5  jubatus-installer/install.sh -D
        6  vi /home/ubuntu/.bashrc 
        7  git clone https://github.com/odasatoshi/jubatus_distributed_handson.git
        8  sudo pip install jubatus pika==0.9.8
        9  exit
       10  history 
    
       ubuntu@[manager]:~$ cd jubatus_distributed_handson/

最初に、 ``jubatus_distributed_handson`` ディレクトリに移動します。
以後、すべてこのディレクトリ内で作業をします。


初期設定
-------------

サンプルに更新があるかもしれません。まずは最新化しましょう。

::

    ubuntu@[manager]:~$ git pull


次に、 ``manager`` 内で、MessageQueueを起動しておきます。

::

    ubuntu@[manager]:~$ sudo sh init_mq.sh 

これは、最初の一回だけで一度起動すれば、マシンを再起動しない限り、ログアウトしても有効です。

``manager`` は、QueueとZookeeperの役割をさせるので、IPアドレス（プライベート）を調べておきましょう。

::

    ifconfig
    eth0      Link encap:Ethernet  HWaddr 12:31:43:01:fc:bc  
              inet addr:10.152.251.74  Bcast:10.152.251.255  Mask:255.255.254.0
              inet6 addr: fe80::1031:43ff:fe01:fcbc/64 Scope:Link
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:1951 errors:0 dropped:0 overruns:0 frame:0
              TX packets:1639 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000 
              RX bytes:147791 (147.7 KB)  TX bytes:235389 (235.3 KB)
              Interrupt:27 

上記のうち、このマシンの場合は、10.152.251.74、今後は、10.X.X.Xと表記します。
サーバのIP（zookeeper, rabbitmq）が必要なのは、このアドレスだけです。

.. raw:: html

    <script language="javascript" type="text/javascript">
        function OnButtonReplaceClick() {
            newip = document.getElementById("privateip");
            document.body.innerHTML=document.body.innerHTML.replace(new RegExp('10.X.X.X', 'g'), newip.value);
            alert(newip.value);
        }
    </script>
    プライベートIPを入れると置換します: <input id="privateip" type="text" /><input type="button" value="置換" onclick="OnButtonReplaceClick();"/>

一台構成
----------------

まずは、簡単に一台構成で挙動を確認しましょう。

.. image:: _static/single.png
   :scale: 80%

これから、近傍探索のタスクを題材にハンズオンを行います。センサーからストリーム的にデータが入ってくることを想定してください。これらのデータを一旦Queueで受けながら、逐次データを学習させます。任意の時点で、ある点の近傍探索を行うことが出来、その時点における近傍にある点を見つけることがこのタスクの目的です。

なお、それぞれの役割は以下のようになっています。

- source.py

  センサーデータが次々にやってくる。それを受けてenqueueする。

- jubatus_update.py 10.X.X.X

  10.X.X.Xにあるキューserverから、dequeueして、localhost のjubanearest_neighborに学習させる。
  jubatus_update.pyからみるとjubanearest_neighborは必ずlocalhostにあるように見える。
  
- jubatus_analyze.py [id]

  idの近傍を取得する。

一台構成で動かしてみます。
その前に、SSHの接続を増やすか、screen, byobuなどで4つのセッションを確保してください。
よくわからなければ、ターミナルを4つ開いて、それぞれからSSHで接続してください。
各シェルで、以下のコマンドを実行します。

* shell1

::

    ubuntu@[manager]:~$ jubanearest_neighbor -f config.json

* shell2

::

    ubuntu@[manager]:~$ python source.py

* shell3

::

    ubuntu@[manager]:~$ python jubatus_update.py 10.X.X.X

* shell4

::

    ubuntu@[manager]:~$ python jubatus_analyze.py 0
    ubuntu@[manager]:~$ python jubatus_analyze.py 1

最後のshell4に近傍が出力されているかと思います。
[('0', -0.0), ('5', -0.025126328691840172), ('9', -0.389676570892334), ('2', -0.6407973170280457), ('19', -1.0131890773773193)]
1つ目の'0'は、'0'に近い点を探しているので当然として、'5', '9'の順に近いとしています。
これは、学習している途中なので、結果はタイミングによって変わります。
なお、もし"WARNING:root:Connect error on fd 7: [Errno 99] Cannot assign requested addressc msgpackrpc.error.TransportError: Retry connection over the limit"のようなエラーが出る場合は、

::

    sudo /sbin/sysctl -w net.ipv4.tcp_tw_recycle=1

を設定しておいてください。一気に多くのクエリーが発行された時に起こります。

分散構成
-----------------

次に分散構成を取ります。
まずは、manager上にzookeeperプロセスを立てます。
jubatusは、サーバ同士、およびプロキシプロセス同士の発見、死活監視をzookeeperを介して行っています。
本来、zookeeperをSPoFにしないように3台以上で構成しますが、今回は簡易的に行っています。

::

    ubuntu@[manager]:~$ sudo /usr/share/zookeeper/bin/zkServer.sh start

これまで起動時に指定していたconfigファイルをzookeeperに登録します。

``sensor_nn`` というのが、このタスクの名前です。このタスクは、zookeeper上に一意である必要があります。
jubatusは、この名前が同じもの同士、MIXを行おうとします。

::

    ubuntu@[manager]:~$ jubaconfig -c write -f config.json -t nearest_neighbor -n sensor_nn -z 10.X.X.X:2181
    ubuntu@[manager]:~$ jubaconfig -c list -z 10.X.X.X:2181

最終的には以下のプロセス構成になります。

.. image:: _static/multi.png
   :scale: 60%

::

    ubuntu@[manager]:~$ python source.py

    ubuntu@[s1]:~$ jubanearest_neighbor --zookeeper 10.X.X.X:2181 -n sensor_nn
    ubuntu@[s2]:~$ jubanearest_neighbor --zookeeper 10.X.X.X:2181 -n sensor_nn

これで、サーバ二台待ち受けている状態になっているはずです。正しくサーバが待ち受けられているかを確認するために、jubactrlを使ってstatusを確認してみましょう。

::

    ubuntu@[manager]:~$ jubactl -z 10.X.X.X:2181 -s jubanearest_neighbor -t nearest_neighbor -c status -n sensor_nn
    active jubaproxy members:
    active jubavisor members:
    active sensor_nn members:
    10.122.23.69_9199
    10.122.23.72_9199

``sensor_nn members`` に二台のマシンが登録されているでしょうか？ここで表示されているprivate IPアドレスは、 ``s1`` , ``s2`` のものです。
jubatusはzookeeperを介して自動的にサーバのIPアドレス、ポートを管理します。利用者はzookeeperの場所を意識するだけでよいようになります。
この後、proxyを立ち上げます。

::

    ubuntu@[c1]:~$ jubanearest_neighbor_proxy --zookeeper 10.X.X.X:2181
    ubuntu@[c2]:~$ jubanearest_neighbor_proxy --zookeeper 10.X.X.X:2181

    ubuntu@[c1]:~$ python jubatus_update.py 10.X.X.X
    ubuntu@[c2]:~$ python jubatus_update.py 10.X.X.X

ここまでで分散できていることを確認しましょう。

::

    ubuntu@[c1]:~$ python jubatus_analyze.py 0


MIXの影響を見る
-------------------

jubatusのMIXは、最後にMIXが行われてからinterval_countで指定された回数updateを受けるか、
interval_secで指定された時間経過するかのどちらかが契機となって始まります。例えば、下記の設定では5分に一度MIXされます。

::

    jubanearest_neighbor --zookeeper 10.X.X.X:2181 --name sensor_nn --interval_sec 300

source.pyは、seedオプションで、乱数の制御が出来ます。また、speedは毎秒最大していされた個数をenqueueします。countで、
何個投入したら止めるかを指定します。

::

    ubuntu@[manager]:~$ python source.py --seed 1 --speed 5 --count 10000

MIXが起きる前と、起きた後で、結果が変わることを確認して下さい。

::

    ubuntu@[c1]:~$ python jubatus_analyze.py 0

ここまでで、本日のハンズオンは終了です。
時間的に余裕がある方は、以下のプログラムを試してみてください。


性能アップを体験する
---------------------
Jubatusのスケールアウトを体験してみましょう。

::

    ubuntu@[manager]:~$ python source.py --silent --count 30000
    ubuntu@[manager]:~$ git checkout meter
    ubuntu@[manager]:~$ python jubameter.py &
    ubuntu@[manager]:~$ python jubameter.py &


``jubameter.py`` は、5秒間辺りのanalyzeを実行できた回数を表示します。
``jubanearest_neighbor`` の台数を変えて変化することを体験してみてください。
