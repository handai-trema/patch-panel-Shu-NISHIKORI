##解答
.confファイルは元から入っていたtrema.confを用いる．  
```
vswitch('lsw') {
  datapath_id 0xabc
}

vhost ('host1') {
  ip '192.168.0.1'
}

vhost ('host2') {
  ip '192.168.0.2'
}

link 'lsw', 'host1'
link 'lsw', 'host2'
```

初期状態のフローテーブルは以下のようになっていた．
```
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=231.67s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=231.51s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=231.51s, table=0, n_packets=0, n_bytes=0, priority=1 actions=goto_table:1
cookie=0x0, duration=231.51s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=231.51s, table=1, n_packets=0, n_bytes=0, priority=1 actions=CONTROLLER:65535
```
優先度3. フラッディングパケットであった場合，フラッディングする．  
優先度2. 宛先のMACアドレスがマルチキャストであった場合，IPv4, IPv6に関係なくドロップする．  
優先度1. 転送テーブルに遷移する．/コントローラにPacket_in()する． 

続いて，ホスト1からホスト2にパケットの送信を行った．
```
$ ./bin/trema send_packets --source host1 --dest host2
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=498.042s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=498.02s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=498.02s, table=0, n_packets=1, n_bytes=42, priority=1 actions=goto_table:1
cookie=0x0, duration=498.02s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=498.02s, table=1, n_packets=1, n_bytes=42, priority=1 actions=CONTROLLER:65535
```
この動作で条件を満足するのは，最も優先度の低い転送テーブル遷移とPacket_in()のみである．転送テーブルはまだ空で，宛先ホストであるhost2の情報も保持していないためフローテーブルテーブルに変化はない．

更に，ホスト2からホスト1にパケットの送信を行った．
```
$ ./bin/trema send_packets --source host2 --dest host1
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=657.393s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=657.24s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=657.24s, table=0, n_packets=2, n_bytes=84, priority=1 actions=goto_table:1
cookie=0x0, duration=657.24s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=120.565s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=3b:52:8a:3c:02:11,dl_dst=f5:38:0c:2b:d1:44 actions=output:1
cookie=0x0, duration=657.24s, table=1, n_packets=2, n_bytes=84, priority=1 actions=CONTROLLER:65535
```
「送信元ホストのポートが2，MACアドレスが3b:52:8a:3c:02:11，宛先のホストのMACアドレスがf5:38:0c:2b:d1:44である場合に，ポート1へ転送する．」というエントリ（優先度2）が加わった．これは，最初のhost1からhost2への送信の際に記録されたhost1の情報をもとに，host2からhost1への送信の際に転送テーブルが更新されたためである．