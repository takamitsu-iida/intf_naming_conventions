
（このページはAdvent Calendar 2023向けに記載したものです）

<!--
20231130 書き始め

この投稿は、JP - FNETS - server-p Advent Calendar 2023の＊＊日目の記事です。

Topic #AdventCalender2023 #社内DX-SEデジタル革新
-->


# Cisco機器のインタフェース名を正規化するPythonコード

シスコ社のネットワーク機器に設定を流し込んだり、コマンドを叩くとき、短縮したインタフェース名を使っても解釈してくれる柔軟性があっていいですね。

たとえば、下記のコマンドは全て同じように解釈されます。

- show interfaces g0/1
- show interfaces gi0/1
- show interfaces gig0/1
- show interfaces Gigabit0/1

人間がコマンドを打つときには便利なのですが、プログラミングで文字列処理するときには少々厄介で、
g0/1とかgi0/1といった形式のデータを正式名称に変換して正規化したい、という場面もあることでしょう。



そもそも、昨今の高速なイーサネットのインタフェース名ってなんだっけ？という疑問もありますね。

サーバやストレージ機器の周辺でよく使われている2ギガとか5ギガみたいなマルチギガビットだったり、
アップリンクで使われる40ギガ、100ギガみたいな高速なLANのインタフェース名って、パッとは思い浮かばないですよね。
どっかにそんな情報まとまってるんじゃないの？と誰もが思うと思いますが、
シスコ社のドキュメントを探すのは意外に難しくて、未だに見つけられずにいます。

ですが、あるところにはあるんです。

Cisco社が公開しているpyATSというPythonで作られたツールのソースコードに、
convert_intf_nameという関数がありまして、
その中に埋め込まれている辞書型のデータが一番まとまってると思います。

https://github.com/CiscoTestAutomation/genieparser/blob/master/src/genie/libs/parser/utils/common.py

上記のリンクが切れたときに備えて、引用しておきます。

```python

class Common:
    '''Common functions to be used in parsers.'''

    @classmethod
    def convert_intf_name(self, intf, os='generic', ignore_case=False):
        '''return the full interface name

            Args:
                intf (`str`): Short version of the interface name
                os (`str`): picks what operating system the interface needs to be translated for.
                ignore_case (`bool`): Case in-sensitive matching of names

            Returns:
                Full interface name fit the standard

            Raises:
                None

            example:

                >>> convert_intf_name(intf='Eth2/1')
        '''

        # takes in the words preceding a digit e.g. the Ge in Ge0/0/1
        m = re.search(r'([-a-zA-Z]+)', intf)
        # takes in everything after the first encountered digit, e.g. the 0/0/1 in Ge0/0/1
        m1 = re.search(r'(\d[\w./]*)', intf)

        # checks if an interface has both Ge and 0/0/1 in the example of Ge0/0/1
        if hasattr(m, 'group') and hasattr(m1, 'group'):
            # fetches the interface type
            int_type = m.group(0)

            # fetch the interface number
            int_port = m1.group(0)

            # Please add more when face other type of interface
            convert = {
                'generic':
                # generic keys for when no OS detected
                {
                    'Eth': 'Ethernet',
                    'Lo': 'Loopback',
                    'lo': 'Loopback',
                    'Fa': 'FastEthernet',
                    'Fas': 'FastEthernet',
                    'Po': 'Port-channel',
                    'PO': 'Port-channel',
                    'Null': 'Null',
                    'Gi': 'GigabitEthernet',
                    'Gig': 'GigabitEthernet',
                    'GE': 'GigabitEthernet',
                    'Te': 'TenGigabitEthernet',
                    'Ten': 'TenGigabitEthernet',
                    'Tw': 'TwoGigabitEthernet',
                    'Two': 'TwoGigabitEthernet',
                    'Twe': 'TwentyFiveGigE',
                    'Fi': 'FiveGigabitEthernet',
                    'Fiv': 'FiveGigabitEthernet',
                    'Fif': 'FiftyGigE',
                    'Fifty': 'FiftyGigabitEthernet',
                    'mgmt': 'mgmt',
                    'Vl': 'Vlan',
                    'Tu': 'Tunnel',
                    'Fe': '',
                    'Hs': 'HSSI',
                    'AT': 'ATM',
                    'Et': 'Ethernet',
                    'BD': 'BDI',
                    'Se': 'Serial',
                    'Fo': 'FortyGigabitEthernet',
                    'For': 'FortyGigabitEthernet',
                    'Hu': 'HundredGigE',
                    'Hun': 'HundredGigE',
                    'TwoH': 'TwoHundredGigabitEthernet',
                    'Fou': 'FourHundredGigE',
                    'vl': 'vasileft',
                    'vr': 'vasiright',
                    'BE': 'Bundle-Ether',
                    'tu': 'Tunnel',
                    'M-E': 'M-Ethernet',  # comware
                    'BAGG': 'Bridge-Aggregation',  # comware
                    'Ten-GigabitEthernet': 'TenGigabitEthernet',  # HP
                    'Wl': 'Wlan-GigabitEthernet',
                    'Di': 'Dialer',
                    'Vi': 'Virtual-Access',
                    'Ce': 'Cellular',
                    'Vp': 'Virtual-PPP',
                    'pw': 'pseudowire'
                },
                'iosxr':
                # interface formats specific to iosxr
                {
                    'BV': 'BVI',
                    'BE': 'Bundle-Ether',
                    'BP': 'Bundle-POS',
                    'Eth': 'Ethernet',
                    'Fa': 'FastEthernet',
                    'Gi': 'GigabitEthernet',
                    'Te': 'TenGigE',
                    'Tf': 'TwentyFiveGigE',
                    'Fo': 'FortyGigE',
                    'Fi': 'FiftyGigE',
                    'Hu': 'HundredGigE',
                    'Th': 'TwoHundredGigE',
                    'Fh': 'FourHundredGigE',
                    'Tsec': 'tunnel-ipsec',
                    'Ti': 'tunnel-ip',
                    'Tm': 'tunnel-mte',
                    'Tt': 'tunnel-te',
                    'Tp': 'tunnel-tp',
                    'IMA': 'IMA',
                    'IL': 'InterflexLeft',
                    'IR': 'InterflexRight',
                    'Lo': 'Loopback',
                    'Mg': 'MgmtEth',
                    'Ml': 'Multilink',
                    'Nu': 'Null',
                    'POS': 'POS',
                    'Pw': 'PW-Ether',
                    'Pi': 'PW-IW',
                    'SRP': 'SRP',
                    'Se': 'Serial',
                    'CS': 'CSI',
                    'G0': 'GCC0',
                    'G1': 'GCC1',
                    'nG': 'nVFabric-GigE',
                    'nT': 'nVFabric-TenGigE',
                    'nF': 'nVFabric-FortyGigE',
                    'nH': 'nVFabric-HundredGigE'
                }
            }

            try:
                os_type_dict = convert[os]
            except KeyError as k:
                log.error((
                    "Check '{}' is in convert dict in utils/common.py, otherwise leave blank.\nMissing key {}\n"
                    .format(os, k)))
                return intf

            if ignore_case:
                mapping = {k.lower():v for k,v in os_type_dict.items()}
                name = int_type.lower()
                if name in mapping:
                    return mapping[name] + int_port
                else:
                    return intf[0].capitalize() + intf[1:].replace(
                        ' ', '').replace('ethernet', 'Ethernet')
            else:
                if int_type in os_type_dict.keys():
                    return os_type_dict[int_type] + int_port
                else:
                    return intf[0].capitalize() + intf[1:].replace(
                        ' ', '').replace('ethernet', 'Ethernet')

        else:
            return intf
```

IOS機器とXR機器でインタフェース名の扱いが微妙に違うことがわかります。

私は必要に応じてこのコードをコピーして使い回すことにしています。
