#解答
コードを以下に示す．  
* [./lib/patch_panel.rb](https://github.com/handai-trema/learning-switch-Shu-NISHIKORI/blob/develop/reports/20161019/patch_panel.rb)  
* [./bin/patch_panel](https://github.com/handai-trema/learning-switch-Shu-NISHIKORI/blob/develop/reports/20161019/patch_panel)  

###ミラーリング
patch_panel.rbには以下のコードを追加した．  
```
  def create_mirror(dpid, port_a, port_b)  
    add_mirror_entries dpid, port_a, port_b  
    @mirror[dpid] += [port_a, port_b].sort  
  end
``` 
```
  def add_mirror_entries(dpid, port_a, port_b)	#a is mirrored, b is mirroring
    send_flow_mod_add(dpid,
                      match: Match.new(in_port: port_a),
                      actions: SendOutPort.new(port_b))
    send_flow_mod_add(dpid,
                      match: Match.new(ip_source_address: '192.168.0.1'),
                      actions: SendOutPort.new(port_b))
  end
```
create_mirror()はpatch_panelから呼ばれる関数である．  
ミラーリングの一覧を表示するために，mirror[]という配列にミラーリングの状況を格納している．そのため，スイッチの起動時にこの配列を確保し，初期化している．  
また，ミラーリングの設定の本体であるadd_mirror_entries()は，送信フレームと受信フレームの転送を設定する．  
port_aに入っていくパケットは，ミラーリングによりport_bに転送される．さらに，パケットの送信元IPアドレスが被ミラーリング対象のホストであれば，port_bに転送される．  
（被ミラーリングホストのIPアドレスが既知であるという少しありえなさそうな想定ですが，他に方法が思いつきませんでした．）  

また，patch_panelには以下のコードを追加した．  
```
  desc 'Creates a new mirror'
  arg_name 'dpid port#1 port#2'	#1 is mirrored, 2 is mirroring
  command :create_mirror do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      dpid = args[0].hex
      port1 = args[1].to_i
      port2 = args[2].to_i
      Trema.trema_process('PatchPanel', options[:socket_dir]).controller.
        create_mirror(dpid, port1, port2)
    end
  end
```
既にあった他のコマンドを参考に作った．

###パッチパネルとミラーリングの一覧
patch_panel.rbには以下のコードを追加した．  
```
  def print_patch(dpid)
    return @patch[dpid]
  end

  def print_mirror(dpid)
    return @mirror[dpid]
  end
```
また，patch_panelには以下のコードを追加した．  
```
  desc 'Print existing patches and mirrors'
  arg_name 'dpid'
  command :print_list do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      dpid = args[0].hex
      puts("Patches")
      puts Trema.trema_process('PatchPanel', options[:socket_dir]).controller.
        print_patch(dpid)
      puts("Mirrors (Mirrored, Mirroring)")
      print Trema.trema_process('PatchPanel', options[:socket_dir]).controller.
        print_mirror(dpid)
    end
  end
```
patch_panel中で配列の中身を表示させるよう使用することを想定して，patch_panel.rbを作成した．  
patch，mirrorそれぞれを別の関数として動作させているので，実装はしていないがpatch，mirrorそれぞれのみを表示させるコマンドを作成することも容易である．  

###動作確認
```
$ ./bin/patch_panel create 0xabc 1 2
$ ./bin/patch_panel m_create 0xabc 1 3
$ ./bin/patch_panel print_list 0xabc
Patches
[1 2]
Mirrors (Mirrored, Mirroring)
[1 3]
$ ./bin/trema send_packets --source host1 --dest host2
$ ./bin/trema send_packets --source host2 --dest host1
$ ./bin/trema show_stats host1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 1 packet
Packets received:
  192.168.0.2 -> 192.168.0.1 = 1 packet
$ ./bin/trema show_stats host2
Packets sent:
  192.168.0.2 -> 192.168.0.1 = 1 packet
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ ./bin/trema show_stats host3
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
  192.168.0.2 -> 192.168.0.1 = 1 packet
```


