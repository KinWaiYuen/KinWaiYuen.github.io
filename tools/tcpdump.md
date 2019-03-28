tcpdump -c 10 -nn -i eth0 tcp dst port 22

分析场景
接收方:
```
➜  ~ sudo tcpdump -c 100 -nn -v -i  lo tcp dst port 9998
tcpdump: listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes
23:33:51.888788 IP (tos 0x0, ttl 64, id 64066, offset 0, flags [DF], proto TCP (6), length 40)
    127.0.0.1.9999 > 127.0.0.1.9998: Flags [R.], cksum 0x7700 (correct), seq 0, ack 3069588922, win 0, length 0
23:34:03.173790 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    127.0.0.1.9999 > 127.0.0.1.9998: Flags [S.], cksum 0xfe30 (incorrect -> 0xa764), seq 2706133401, ack 3245919350, win 43690, options [mss 65495,sackOK,TS val 17700438 ecr 17700438,nop,wscale 7], length 0
23:34:03.174247 IP (tos 0x0, ttl 64, id 18380, offset 0, flags [DF], proto TCP (6), length 52)
    127.0.0.1.9999 > 127.0.0.1.9998: Flags [.], cksum 0xfe28 (incorrect -> 0x75c9), ack 1001, win 334, options [nop,nop,TS val 17700438 ecr 17700438], length 0
23:34:03.174590 IP (tos 0x0, ttl 64, id 18381, offset 0, flags [DF], proto TCP (6), length 57)
    127.0.0.1.9999 > 127.0.0.1.9998: Flags [P.], cksum 0xfe2d (incorrect -> 0xb238), seq 1:6, ack 1001, win 334, options [nop,nop,TS val 17700439 ecr 17700438], length 5
23:34:04.176114 IP (tos 0x0, ttl 64, id 18382, offset 0, flags [DF], proto TCP (6), length 57)
    127.0.0.1.9999 > 127.0.0.1.9998: Flags [P.], cksum 0xfe2d (incorrect -> 0xa67f), seq 6:11, ack 2001, win 327, options [nop,nop,TS val 17701440 ecr 17701440], length 5
23:34:05.178091 IP (tos 0x0, ttl 64, id 18383, offset 0, flags [DF], proto TCP (6), length 57)
    127.0.0.1.9999 > 127.0.0.1.9998: Flags [P.], cksum 0xfe2d (incorrect -> 0x9ac5), seq 11:16, ack 3001, win 320, options [nop,nop,TS val 17702442 ecr 17702442], length 5
```
发送放
```
➜  ~ sudo tcpdump -c 100 -nn -v -i lo tcp dst port 9999
tcpdump: listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes
23:33:51.888756 IP (tos 0x0, ttl 64, id 57336, offset 0, flags [DF], proto TCP (6), length 60)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [S], cksum 0xfe30 (incorrect -> 0x7f13), seq 3069588921, win 43690, options [mss 65495,sackOK,TS val 17689153 ecr 0,nop,wscale 7], length 0
23:34:03.173728 IP (tos 0x0, ttl 64, id 55843, offset 0, flags [DF], proto TCP (6), length 60)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [S], cksum 0xfe30 (incorrect -> 0xb1bf), seq 3245919349, win 43690, options [mss 65495,sackOK,TS val 17700438 ecr 0,nop,wscale 7], length 0
23:34:03.173811 IP (tos 0x0, ttl 64, id 55844, offset 0, flags [DF], proto TCP (6), length 52)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [.], cksum 0xfe28 (incorrect -> 0x79a9), ack 2706133402, win 342, options [nop,nop,TS val 17700438 ecr 17700438], length 0
23:34:03.174224 IP (tos 0x0, ttl 64, id 55845, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x4387), seq 0:1000, ack 1, win 342, options [nop,nop,TS val 17700438 ecr 17700438], length 1000
23:34:03.175093 IP (tos 0x0, ttl 64, id 55846, offset 0, flags [DF], proto TCP (6), length 52)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [.], cksum 0xfe28 (incorrect -> 0x75ba), ack 6, win 342, options [nop,nop,TS val 17700439 ecr 17700439], length 0
23:34:04.175934 IP (tos 0x0, ttl 64, id 55847, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x9bef), seq 1000:2000, ack 6, win 342, options [nop,nop,TS val 17701440 ecr 17700439], length 1000
23:34:04.176303 IP (tos 0x0, ttl 64, id 55848, offset 0, flags [DF], proto TCP (6), length 52)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [.], cksum 0xfe28 (incorrect -> 0x69fb), ack 11, win 342, options [nop,nop,TS val 17701440 ecr 17701440], length 0
23:34:05.177921 IP (tos 0x0, ttl 64, id 55849, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x902f), seq 2000:3000, ack 11, win 342, options [nop,nop,TS val 17702442 ecr 17701440], length 1000
23:34:05.178195 IP (tos 0x0, ttl 64, id 55850, offset 0, flags [DF], proto TCP (6), length 52)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [.], cksum 0xfe28 (incorrect -> 0x5e3a), ack 16, win 342, options [nop,nop,TS val 17702442 ecr 17702442], length 0
23:34:06.179598 IP (tos 0x0, ttl 64, id 55851, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x846e), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 17703444 ecr 17702442], length 1000
23:34:06.381505 IP (tos 0x0, ttl 64, id 55852, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x83a5), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 17703645 ecr 17702442], length 1000
23:34:06.582796 IP (tos 0x0, ttl 64, id 55853, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x82db), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 17703847 ecr 17702442], length 1000
23:34:06.985895 IP (tos 0x0, ttl 64, id 55854, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x8148), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 17704250 ecr 17702442], length 1000
23:34:07.792695 IP (tos 0x0, ttl 64, id 55855, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x7e21), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 17705057 ecr 17702442], length 1000
23:34:09.403723 IP (tos 0x0, ttl 64, id 55856, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x77d6), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 17706668 ecr 17702442], length 1000
23:34:12.624313 IP (tos 0x0, ttl 64, id 55857, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x6b42), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 17709888 ecr 17702442], length 1000
23:34:19.071627 IP (tos 0x0, ttl 64, id 55858, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x5212), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 17716336 ecr 17702442], length 1000
23:34:31.951737 IP (tos 0x0, ttl 64, id 55859, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x1fc2), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 17729216 ecr 17702442], length 1000
23:34:57.744687 IP (tos 0x0, ttl 64, id 55860, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0xbb00), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 17755009 ecr 17702442], length 1000
23:35:49.328691 IP (tos 0x0, ttl 64, id 55861, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0xf17f), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 17806593 ecr 17702442], length 1000
23:37:32.495645 IP (tos 0x0, ttl 64, id 55862, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x5e7f), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 17909760 ecr 17702442], length 1000
23:39:32.815777 IP (tos 0x0, ttl 64, id 55863, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0x887d), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 18030080 ecr 17702442], length 1000
23:41:33.136689 IP (tos 0x0, ttl 64, id 55864, offset 0, flags [DF], proto TCP (6), length 1052)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [P.], cksum 0x0211 (incorrect -> 0xb27a), seq 3000:4000, ack 16, win 342, options [nop,nop,TS val 18150401 ecr 17702442], length 1000
23:41:34.622557 IP (tos 0x0, ttl 64, id 55865, offset 0, flags [DF], proto TCP (6), length 52)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [F.], cksum 0xfe28 (incorrect -> 0x7ea5), seq 4000, ack 16, win 342, options [nop,nop,TS val 18151887 ecr 17702442], length 0

```
分析握手过程

接收方
```
    127.0.0.1.9999 > 127.0.0.1.9998: Flags [S.], cksum 0xfe30 (incorrect -> 0xa764), seq 2706133401, ack 3245919350, win 43690, options [mss 65495,sackOK,TS val 17700438 ecr 17700438,nop,wscale 7], length 0
```
发送方
```
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [S], cksum 0xfe30 (incorrect -> 0xb1bf), seq 3245919349, win 43690, options [mss 65495,sackOK,TS val 17700438 ecr 0,nop,wscale 7], length 0
23:34:03.173811 IP (tos 0x0, ttl 64, id 55844, offset 0, flags [DF], proto TCP (6), length 52)
    127.0.0.1.9998 > 127.0.0.1.9999: Flags [.], cksum 0xfe28 (incorrect -> 0x79a9), ack 2706133402, win 342, options [nop,nop,TS val 17700438 ecr 17700438], length 0
```
[S.]是SYN,[.]是ACK [P.]是PUSH

### 握手
发送: 
[S.] seq 3245919349
接收: ack是发送的seq+1
[S.] seq 2706133401, ack 3245919350
发送: ack是接收seq+1
[.] ack 2706133402
建立连接.

### 发数据
**发送**:23:34:03.174224
[P.][P.], cksum 0x0211 (incorrect -> 0x4387), seq 0:**1000**, ack **1**, win 342, options [nop,nop,TS val 17700438 ecr 17700438], length **1000**

**接收**:23:34:03.174590
[P.][P.], cksum 0xfe2d (incorrect -> 0xb238), seq 1:**6**, ack **1001**, win 334, options [nop,nop,TS val 17700439 ecr 17700438], length **5**

**发送**:23:34:03.175093 
[.], cksum 0xfe28 (incorrect -> 0x75ba), ack **6**, win 342, options [nop,nop,TS val 17700439 ecr 17700439], length 0

================

**发送**:23:34:04.175934
[P.], cksum 0x0211 (incorrect -> 0x9bef), seq **1000**:**2000**, ack **6**, win 342, options [nop,nop,TS val 17701440 ecr 17700439], length **1000**

**接收**:23:34:04.176114 
[P.], cksum 0xfe2d (incorrect -> 0xa67f), seq **6**:**11**, ack 2001, win 327, options [nop,nop,TS val 17701440 ecr 17701440], length **5**

**发送**:23:34:04.176303
[.], cksum 0xfe28 (incorrect -> 0x69fb), ack **11**, win 342, options [nop,nop,TS val 17701440 ecr 17701440], length 0

=============

**发送**:23:34:05.177921
[P.], cksum 0x0211 (incorrect -> 0x902f), seq **2000**:**3000**, ack **11**, win 342, options [nop,nop,TS val 17702442 ecr 17701440], length **1000**

**接收**:23:34:05.178091
[P.], cksum 0xfe2d (incorrect -> 0x9ac5), seq **11**:**16**, ack **3001**, win 320, options [nop,nop,TS val 17702442 ecr 17702442], length **5**

**发送**:23:34:05.178195
[.], cksum 0xfe28 (incorrect -> 0x5e3a), ack **16**, win 342, options [nop,nop,TS val 17702442 ecr 17702442], length 0

===========

**发送**:23:34:06.179598 
[P.], cksum 0x0211 (incorrect -> 0x846e), seq **3000**:**4000**, ack 16, win 342, options [nop,nop,TS val 17703444 ecr 17702442], length **1000**


可以看到  接收端发送ack后,发送端会发一个确认包 告知接收方感知到接收方的接收

