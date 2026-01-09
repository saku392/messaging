# reputation に関する研究

### SPF レコード内容によるドメインレピュテーション

SPF レコードに記載している IP アドレス数に応じて，
そのドメイン名を評価してはどうか．
IP アドレス数 (幅) が大きいドメイン名は，
不適切な設定をしている可能性があり，
またメール送信出口を精密に管理していないと考えられることから，
ドメイン名が不正利用されている可能性もある．
特に末尾が "+all" のものは，
どの送信元からのメールでも pass となるため，
悪用目的で取得したか，
設定の不備となっており，いずれも悪用されている可能性が高い．
但し，
"spf.protection.outlook.com" (Microsoft 365) や Google, AmazonSES
などのサービスの SPF レコードを include している場合は IP アドレス数のカウントから除外すべきである (あまりにも多いため)．

さらに，
SPF レコードのに記載されている IP アドレスが，
IP Block List に登録されているかどうかを調べるのはどうだろうか．
IP アドレスによる DNSBL は，
長い歴史があり現在でも主要な迷惑メール対策手法として利用されており，
DNSBL にもよるが信頼性はある程度高いと考えて良い．
SPF レコードには，
メールの送信出口の IP アドレスおよびそれに紐づく情報 (mx や a など) が記載されている．
これらが既に DNSBL に登録されているとすれば，
その SPF レコードを持つドメイン名の評価も低くなるはずである．

ドメイン名の評価確認ツール

- [MxToolbox](https://mxtoolbox.com/blacklists.aspx)
- [Cisco TALOS](https://talosintelligence.com)

DNSBL として有名な [spamhaus](https://www.spamhaus.org) では，
IPアドレスについて数種類の Black List を提供している．
近年はドメイン名についても Black List を提供しているが，
その規模や網羅率については不明である．
IPアドレスの Block List には，
[ここ](https://www.spamhaus.org/blocklists/)で示されているように，
SBL (Spamhaus Block List), PBL (Policy Block List), XBL (Exploit Block List) があり，
それらの組み合わせで ZEN (SBL+CSS+XBL+PBL) と CSS (PBL + XBL) がある．
つまり一般的には，ZEN を使えば IP Block List に登録されているかどうかを確認することができる．

これまでは，
spamhaus.org ドメイン名での利用が可能であったが，
近年はサポート等を行う Spamhaus Technology にアカウントを作成し，
利用することが推奨されているようである．
Spamhaus Technology でも [free account](https://portal.spamhaus.com/auth/account-setup?ps=free_dqs_product) を提供している．
アカウントを作成すると Account Number が発行され，
portal に login できるようになり，
DQS (Data Query Service) key が生成できるようになる．

spamhaus の DNSBL の利用方法を以下に示す．
対象とするIPアドレスは，
1 octet 単位で逆向きに指定する．
つまり IP アドレス "73.97.249.63" を調べる場合は以下のようになる．
現状では，
まだ従来のアクセス方法 (spamhaus.org ドメイン名の利用) も可能なようである．

```
$ dig +short 63.249.97.73.<dqs-key>.zen.dq.spamhaus.net A
127.0.0.10

$ dig +short 63.249.97.73.zen.spamhaus.org A
127.0.0.10
```

DNSBLに登録されている場合は，
以下の応答 (IPアドレス) が返される．
上記の場合は，PBL に登録されている，ということを示している．

|  response | mean |
| :-------: | :--- |
| 127.0.0.2 | SBL  |
| 127.0.0.3 | CSS  |
| 127.0.0.4 | XBL (CBL)  |
| 127.0.0.9 | DROP |
| 127.0.0.10 | PBL (ISP Maintained)  |
| 127.0.0.11 | PBL (Spamhaus Maintained) |

DNSBL に登録されていない場合は，
NXDOMAIN が応答となる．

ドメイン名の block list も同様に，
spamhaus.org と spamhaus.net (spamhaus technology) の両方で利用可能である (現時点)．
ドメイン名の場合は，
右側から範囲が狭まるため，
一般的なドメイン名の並びを dbl ラベル以下に設定する．
例えば "jqvfqy.net" ドメイン名を調べる場合は，
DBL (Domain Block List) に対して以下のように を指定する．

```
$ dig +short jqvfqy.net.<dqs-key>.dbl.dq.spamhaus.net A
127.0.1.2

$ dig +short jqvfqy.net.dbl.spamhaus.org 
127.0.1.2
```

応答コードの意味は以下となっている．

|  response | mean |
| :-------: | :--- |
| 127.0.1.2 | spam domain  |
| 127.0.1.4 | phish domain  |
| 127.0.1.5 | malware domain |
| 127.0.1.6 | botnet C&C domain |
| 127.0.1.102 | abused legit spam |
| 127.0.1.103 | abused spammed redirector domain |
| 127.0.1.104 | abused legit phish |
| 127.0.1.105 |	abused legit malware |
| 127.0.1.106 |	abused legit botnet C&C |
| 127.0.1.255 |	IP queries prohibited! |

あるところから入手した spam 送信ドメイン名 192 を調べたところ，
spam domain が 52, phish domain が 9 という結果が得られた．
