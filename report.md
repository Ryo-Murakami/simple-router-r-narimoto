#課題レポート(ルータのCLIを作ろう)
更新日：(2016.11.02)

##課題
```
課題内容：
ルータのコマンドラインインターフェース(CLI)を作る．
次の操作ができるコマンドを作成
  ・ルーティングテーブルの表示
  ・ルーティングテーブルエントリの追加と削除
  ・ルータのインターフェース一覧の表示
  ・そのほか，あると便利な機能
```

##目次  
1. [ルーティングテーブルの表示](#printRT)  
1. [ルーティングテーブルエントリの追加と削除](#addRT)  
1. [ルータのインターフェース一覧の表示](#printInterface)  

<a id="printRT"></a>
##1.ルーティングテーブルの表示  
下記のようにメソッドを追加した．  

###print_routing_tableメソッド  
RoutingTableクラスに実装したgetDBメソッドの戻り値をそのまま返す．  
```ruby
#./lib/simple_router.rb
def print_routing_table()
    return @routing_table.getDB()
end
```  

###getDBメソッド  
RoutingTableクラス内で保持している@db変数の中身を，すべてipアドレスを示す文字列へと変換した状態のハッシュの配列として生成しなおして返している．  
@db変数そのままの状態では，ハッシュのキーとなる宛先ipのプレフィックスが整数型に変換されてしまっている為，IPv4Addressクラスを利用して可読性の高い文字列へと変換するようにしている．
```ruby
#./lib/routing_table.rb
def getDB()
  ret = Array.new()
  @db.each do |each|
    tmp = Hash.new()
    each.each do |key, value|
      tmp[IPv4Address.new(key).to_s] = value.to_s
    end
    ret << tmp
  end
  return ret
end
```  

###printRTコマンド  
新たにbinフォルダ内にコマンド実行用のバイナリsimple_routerを作成した．  
ルーティングテーブルを表示するときのコマンドは以下の通り．
```
./bin/simple_router printRT
```  
実装は以下の通り．  
```ruby
#./bin/simple_router
desc 'Print routing table'
arg_name '@routing_table'
command :printRT do |c|
  c.desc 'Location to find socket files'
  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

  c.action do |_global_options, options, args|
    @routing_table = Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      print_routing_table()
    print "\"destination/netmask\" \"next hop\"\n"
    @routing_table.each do |each|
      mask = @routing_table.find_index each
      each.each do |key, value|
        print sprintf("%-21s %s\n", key+"/"+mask.to_s, value)
      end
    end
  end
end
```  

###動作確認  
起動時の初期状態のルーティングテーブルは以下の通り．  
```
#./simple_router.conf
ROUTES = [
  {
    destination: '0.0.0.0',
    netmask_length: 0,
    next_hop: '192.168.1.2'
  }
]
```
実行結果は以下のようになった．  
```
$ ./bin/simple_router printRT
"destination/netmask" "next hop"
0.0.0.0/0             192.168.1.2
$
```  
結果より，正しくルーティングテーブルが表示されていることがわかる．  
また，以降の実装の動作確認においても使用しており，そちらも含めて実装が正しいことを改めて確認している．

<a id="addRT"></a>
##2.ルーティングテーブルエントリの追加と削除    
下記のようにメソッドを追加した．  

###add_routing_table_entryメソッド  
RoutingTableクラスに実装されているaddメソッドを利用する．  
渡す引数だけ，コードを確認して合うように用意している．
```ruby
#./lib/simple_router.rb
def add_routing_table_entry(destination_ip, netmask_length, next_hop)
    options = {:destination => destination_ip, :netmask_length => netmask_length, :next_hop => next_hop}
    @routing_table.add(options)
end
```  
###delete_routing_table_entryメソッド  
RoutingTableクラスにdeleteメソッドを実装し，追加と同様に利用している．  
渡す引数のフォーマットを追加と同様にしている．
```ruby
#./lib/simple_router.rb
def delete_routing_table_entry(destination_ip, netmask_length)
  options = {:destination => destination_ip, :netmask_length => netmask_length}
  @routing_table.delete(options)
end
```  

###deleteメソッド  
既に実装されていたaddメソッドを参考に，optionsに含まれる情報を元に，
配列から削除するメソッドを下記の通り実装した．
```ruby
#./lib/routing_table.rb
def delete(options)
  netmask_length = options.fetch(:netmask_length)
  prefix = IPv4Address.new(options.fetch(:destination)).mask(netmask_length)
  @db[netmask_length].delete(prefix.to_i)
end
```  

###addRT/delRTコマンド  
./bin/simple_routerにaddRTとdelRTコマンドを実装し，下記のように使えるようにした．  
```
(追加)
./bin/simple_router addRT [destination ip] [netmask length] [next hop]
(削除)
./bin/simple_router delRT [destination ip] [netmask length]
```  
実装は以下の通り．  
```ruby
#./bin/simple_router
desc 'Add routing table entry'
arg_name 'destination_ip netmask_length next_hop'
command :addRT do |c|
  c.desc 'Location to find socket files'
  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

  c.action do |_global_options, options, args|
    destination_ip = args[0]
    netmask_length = args[1].to_i
    next_hop = args[2]
    Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      add_routing_table_entry(destination_ip, netmask_length, next_hop)
  end
end

desc 'Delete routing table entry'
arg_name 'destination_ip netmask_length'
command :delRT do |c|
  c.desc 'Location to find socket files'
  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

  c.action do |_global_options, options, args|
    destination_ip = args[0]
    netmask_length = args[1].to_i
    next_hop = args[2]
    Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      delete_routing_table_entry(destination_ip, netmask_length)
  end
```  

###動作確認  
次の手順で動作確認を行った．  
```
1.現在のルーティングテーブルを表示
2.いくつか適当にエントリを追加
3.追加したエントリをすべて削除
この際，各アクションの前にルーティングテーブルを表示して確認
```  
実行結果は以下の通りとなった．  
```
$ bin/simple_router printRT
"destination/netmask" "next hop"
0.0.0.0/0             192.168.1.2
$ addRT 192.168.1.0 24 192.168.1.1
$ printRT                         
"destination/netmask" "next hop"
0.0.0.0/0             192.168.1.2
192.168.1.0/24        192.168.1.1
$ addRT 192.168.2.0 24 192.168.2.1
$ addRT 192.168.1.0 24 192.168.1.1
$ printRT                         
"destination/netmask" "next hop"
0.0.0.0/0             192.168.1.2
192.168.1.0/24        192.168.1.1
192.168.2.0/24        192.168.2.1
$ addRT 128.64.0.0 16 128.64.0.1  
$ printRT                       
"destination/netmask" "next hop"
0.0.0.0/0             192.168.1.2
128.64.0.0/16         128.64.0.1
192.168.1.0/24        192.168.1.1
192.168.2.0/24        192.168.2.1
$ delRT 128.64.0.0 16        
$ printRT            
"destination/netmask" "next hop"
0.0.0.0/0             192.168.1.2
192.168.1.0/24        192.168.1.1
192.168.2.0/24        192.168.2.1
$ delRT 192.168.2.0 24
$ printRT             
"destination/netmask" "next hop"
0.0.0.0/0             192.168.1.2
192.168.1.0/24        192.168.1.1
$ delRT 192.168.1.0 24
$ printRT             
"destination/netmask" "next hop"
0.0.0.0/0             192.168.1.2
$
```  
結果より，ルーティングテーブルの追加と削除が正しく行えていることを確認した．  
また，ルーティングテーブルの表示も正しく行えていることも同時に確認している．  

<a id="printInterface"></a>
##3.ルータのインターフェース一覧の表示  
下記のようにメソッドを追加した．  

###print_interfaceメソッド  
保持しているインターフェースを表示する際に扱いやすいように文字列や数値のみで表されるハッシュにし，それをインターフェースごとの要素として配列に格納して返すように実装している．  
```ruby
#./lib/simple_router.rb
def print_interface()
  ret = Array.new()
  Interface.all.each do |each|
    ret << {:port_number => each.port_number, :mac_address => each.mac_address.to_s, :ip_address => each.ip_address.value.to_s, :netmask_length => each.netmask_length}
  end
  return ret
end
```  

###printInterfaceコマンド  
./bin/simple_routerにprintInterfaceコマンドを実装し，下記のように使えるようにした．  
```
./bin/simple_router printInterface
```  
実装は以下の通り．  
```ruby
#./bin/simple_router
desc 'Print interface'
arg_name 'interfaces'
command :printInterface do |c|
  c.desc 'Location to find socket files'
  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

  c.action do |_global_options, options, args|
    interfaces = Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.print_interface()
    print sprintf("%s %-17s %s", "\"port number\"", "\"mac address\"", "\"ip address/netmask\"\n")
    interfaces.each do |each|
      print sprintf("%-13s %-17s %s\n", each[:port_number].to_s, each[:mac_address], each[:ip_address]+"/"+each[:netmask_length].to_s)
    end
  end
end
```  

###動作確認  
起動時の初期状態におけるインターフェースは以下の通り．  
```
#./simple_router.conf
INTERFACES = [
  {
    port: 1,
    mac_address: '01:01:01:01:01:01',
    ip_address: '192.168.1.1',
    netmask_length: 24
  },
  {
    port: 2,
    mac_address: '02:02:02:02:02:02',
    ip_address: '192.168.2.1',
    netmask_length: 24
  }
]
```
実行結果は以下のようになった．  
```
$ bin/simple_router printInterface                  
"port number" "mac address"     "ip address/netmask"
1             01:01:01:01:01:01 192.168.1.1/24
2             02:02:02:02:02:02 192.168.2.1/24
$
```  
結果より，正しくインターフェースが表示されていることがわかる．  

##ソースコード
* [./lib/simple_router.rb](https://github.com/handai-trema/simple-router-r-narimoto/blob/master/lib/simple_router.rb)  
* [./lib/routing_table.conf](https://github.com/handai-trema/simple-router-r-narimoto/blob/master/lib/routing_table.rb)  
* [./bin/simple_router](https://github.com/handai-trema/simple-router-r-narimoto/blob/master/bin/simple_router)  
