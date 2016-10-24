#情報ネットワーク学演習II 10/19 レポート課題
===========
学籍番号 33E16011
提出者 田中 達也

## 課題 (マルチプルテーブルを読む)

OpenFlow1.3 版スイッチの動作を説明しよう。

スイッチ動作の各ステップについて、trema dump_flows の出力 (マルチプルテーブルの内容) を混じえながら動作を説明すること。

## 動作説明
各ステップについて動作を説明する。
### スイッチ接続
tremaを起動した直後のマルチプルテーブルを以下に示す。
```
cookie=0x0, duration=709.088s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=709.049s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=709.049s, table=0, n_packets=0, n_bytes=0, priority=1 actions=goto_table:1
cookie=0x0, duration=709.049s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=709.049s, table=1, n_packets=0, n_bytes=0, priority=1 actions=CONTROLLER:65535
```
1行目から3行目は、table=0であることからtable IDが0、すなわちフィルタリングテーブルである。
1行目は、宛先MACアドレスがマルチキャストアドレスである場合パケットをドロップすることを意味している。優先度は2である。
2行目は、宛先MACアドレスがipV6マルチキャストアドレスである場合パケットをドロップすることを意味している。優先度は2である。
3行目は上記のフィルタリングに引っかからなかった場合、table1（転送テーブル)に処理を移行することを意味している。優先度は1である。
4,5行目は、table=1であることから、転送テーブルである。
4行目は、宛先MACアドレスがブロードキャストアドレスである場合、パケットをフラッディングすることを意味している。優先度は3である。
5行目は、コントローラーにPacketInすることを意味している。優先度は1である。

これによりOpenFlow1.0では起動後に到着したパケットはすべてPacketInするデフォルトPacketInであったが、マルチプルテーブルにし、フィルタリングを導入することで、PacketInの数を減らすことができている。
### host1からhost2へパケット送信
host1からhost2へパケットを送信した後のマルチプルテーブルを以下に示す。
```
ensyuu2@ensyuu2-VirtualBox:~/learning-switch-Tatsu-Tanaka$ ./bin/trema send_packets --source host1 --dest host2
ensyuu2@ensyuu2-VirtualBox:~/learning-switch-Tatsu-Tanaka$ ./bin/trema dump_flows lsw
cookie=0x0, duration=754.48s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=754.441s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=754.441s, table=0, n_packets=1, n_bytes=42, priority=1 actions=goto_table:1
cookie=0x0, duration=754.441s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=754.441s, table=1, n_packets=1, n_bytes=42, priority=1 actions=CONTROLLER:65535
```
マルチプルテーブルに変化は生じていない。
host2宛てのパケットはフィルタリングに引っかからず、転送テーブルに処理が移行する。(3行目：n_packets=1)
転送テーブルでは、ブロードキャストアドレス宛のパケットではないのでフラッディングされず、コントローラーにPacketInされる。(5行目：n_packets=1)


### host2からhost1へパケット送信
host2からhost1へパケットを送信した後のマルチプルテーブルを以下に示す。
```
ensyuu2@ensyuu2-VirtualBox:~/learning-switch-Tatsu-Tanaka$ ./bin/trema send_packets --source host2 --dest host1
ensyuu2@ensyuu2-VirtualBox:~/learning-switch-Tatsu-Tanaka$ ./bin/trema dump_flows lsw
cookie=0x0, duration=773.711s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=773.672s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=773.672s, table=0, n_packets=2, n_bytes=84, priority=1 actions=goto_table:1
cookie=0x0, duration=773.672s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=5.584s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=60:c7:43:47:7b:3f,dl_dst=b3:ce:db:30:bd:71 actions=output:1
cookie=0x0, duration=773.672s, table=1, n_packets=2, n_bytes=84, priority=1 actions=CONTROLLER:65535
```
host1宛てのパケットはフィルタリングに引っかからず、転送テーブルに処理が移行する。(3行目：n_packets=2)
転送テーブルでは、ブロードキャストアドレス宛のパケットではないのでフラッディングされず、コントローラーにPacketInされる。(6行目：n_packets=2)
新たに5行目に`cookie=0x0, duration=5.584s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=60:c7:43:47:7b:3f,dl_dst=b3:ce:db:30:bd:71 actions=output:1`が追加されている。
これは送信元MACアドレスが60:c7:43:47:7b:3f、宛先MACアドレスがb3:ce:db:30:bd:71のパケットをポート1に転送するという意味である。優先度は2である。
host1からhost2へパケット送信のPacketInの際にコントローラーは今回のPacketInの宛先であるhost1のポート番号を学習しているのでフローテーブルを更新している。

##参考文献
デビッド・トーマス+アンドリュー・ハント(2001)「プログラミング Ruby」ピアソン・エデュケーション.  
[テキスト: 8章 "OpenFlow1.3版ラーニングスイッチ"](http://yasuhito.github.io/trema-book/#learning_switch13)



