---
title: "メモ: BGP Graceful Restartの障害検出時間"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

[BGP Graceful Restart](https://datatracker.ietf.org/doc/html/rfc4724)は何らかの理由でBGPのセッションが切断された際, 本来であればPeerのルーティングテーブルから経路が削除されトラフィック断が発生するところを, Peerにパケットの転送を継続させることができる仕組み.

大変便利に見える一方で, 多くのBGPの実装ではGraceful RestartはデフォルトでOffになっている. なぜかというとGraceful RestartはBGPのセッションが切断されてもデータプレーンは生きているというケースには有効だが, データプレーンもBGPも両方落ちているというケースでは逆に到達不可能な経路に対してパケットを転送し続けてしまうという欠点があるからだと思う.

このポストでは, このようなリスクがあることを承知でGraceful Restartを使う場合に, どうすればこのような通信断を最小に抑えることができるのかをRFCによる理論とFRRによる実装を整理しながら考察する.

# スコープ

このポストで扱うRFCは以下

- Graceful Restart Mechanism for BGP ([RFC4724](https://datatracker.ietf.org/doc/html/rfc4724))
- Notification Message Support for BGP Graceful Restart ([RFC8538](https://datatracker.ietf.org/doc/html/rfc8538))

Long-Lived Graceful Restart ([RFC9494](https://datatracker.ietf.org/doc/html/rfc9494)) があるケースもやりたかったが調べる時間がなかったのでまた今度.

# RFC4724のみのケース

RFC8538は大抵の場合On/Offが切り替えられるようになっていて, FRRでもOn/Offが切り替えられるようになっている (FRR 10.0でのデフォルトはONだった). RFC8538のサポートがないルータを現役で使っていてかつGraceful Restartを使いたいケースが現場にあるのかどうかは分からないけれども, 多くのルータの設定的にはできる構成なので, まずはこのケースから考えてみる. 仕様的にはNotificationを送った場合はPeer側で即座に経路が破棄されるので, 通信断の時間はゼロ. Notificationを送らずにTCPのコネクションだけを切った場合の挙動は実装依存. Notificationも送らず, TCPのコネクションも正常に切断されなかった場合は普通にHold Timeが切れるまで通信断する. これはGraceful Restartは関係ない通常のBGPの挙動.

実際にこの挙動を以下のようなFRRの設定で検証してみた. これはRestartする側のルータの設定. Peer側もAS番号以外は同じ.

```
router bgp 64512
 no bgp ebgp-requires-policy
 ! Disable RFC8538 support
 no bgp graceful-restart notification
 neighbor PEERS peer-group
 neighbor PEERS remote-as external
 neighbor PEERS graceful-restart
 neighbor net0 interface peer-group PEERS
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
exit
```

色々試してみたが, `no bgp graceful-restart notification` な時にどうすればGraceful RestartがTriggerされるのかはよくわからなかった. コードをみる限りだとBGP NotificationなしでTCPコネクションのみが切断された場合にTriggerされそうだったが自分が試した限りではうまくいかなかった. 試した方法は以下.

- no neighborコマンド => Cease / Peer Deconfiguredが送信されてPeerが即座に経路を破棄
- bgp shutdownコマンド => Cease / Administrative Shutdownが送信されてPeerが即座に経路を破棄
- bgpdにSIGINT => TCPコネクションが切断, Peerが即座に経路を破棄
- bgpdにSIGTERM => TCPコネクションが切断, Peerが即座に経路を破棄
- bgpdにSIGKILL => TCPコネクションが切断, Peerが即座に経路を破棄

# RFC8538があるケース

RFC8538はRFC4724の仕様を拡張してBGP Notificationが送信された時とHold Timerが切れた時にGraceful RestartがTriggerできるようにしている. しかしこれだと経路をすぐに破棄して欲しい場合にPeer側にそれを伝える方法がないので, 新しいBGP Notification, Cease / Hard Resetを導入している. 他のNotificationとは違ってCease / Hard ResetはGraceful RestartをTriggerしないようになっている. ただしRFC8538はどういう時にCease / Hard Resetを送るべきなのかを厳密には定めておらず, Recommendationという形でとどめているため実質的には実装依存になっている.

このケースではVoluntaryな再起動の場合はCease / Hard Resetを送ることで即座に経路を破棄させることができるが, Involuntaryなケースでは不通になった経路にパケットを転送し続けるケースがRFC4724のみの場合と比べて増えてしまっている. Hold Timerが切れた時にGraceful RestartがTriggerされるということは最悪のケースで `Hold Time + Restart Time` だけ待たなければ経路が破棄されないということになる. これはHold TimeとRestart Timeを短くすることである程度は対処できるが, Hold Timeは最低3秒までしか縮めることができず, Restart Timeはあまり短くしすぎると正常なRestartの時にRestartが間に合わずに経路が破棄されてしまうので本末転倒になる. 結局このケースに対処するにはBFDなどのBGP以外の方法でデータプレーンの障害を検出してBGPの方にフィードバックするしかないように思える.

ともあれFRRでの検証をしてみた. 設定はさっきとほとんど変わらない.

```
router bgp 64512
 no bgp ebgp-requires-policy
 ! no bgp graceful-restart notification
 neighbor PEERS peer-group
 neighbor PEERS remote-as external
 neighbor PEERS graceful-restart
 neighbor net0 interface peer-group PEERS
 neighbor 10.0.1.2 peer-group PEERS
 address-family ipv4 unicast
  redistribute connected
exit
```

今回は概ね想定通りにGraceful RestartをTriggerすることができた. Cease / Hard Resetを送る方法も把握できた.

- no neighborコマンド => Cease / Hard Resetが送信されてPeerが即座に経路を破棄
- bgp shutdownコマンド => Cease / Hard Resetが送信されてPeerが即座に経路を破棄
