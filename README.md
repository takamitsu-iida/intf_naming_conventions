（このページはAdvent Calendar 2023向けに記載したものです）

<!--
この投稿は、JP - FNETS - server-p Advent Calendar 2023の＊＊日目の記事です。

Topic
#AdventCalender2023 #社内DX-SEデジタル革新

python3 -m venv .venv
direnv allow
-->

# Cisco機器のインタフェース名を正規化するPythonコード

シスコ社のネットワーク機器に設定を流し込んだり、コマンドを叩くとき、短縮したインタフェース名を使っても解釈してくれる柔軟性があっていいですね。

たとえば、下記のコマンドは全て同じように解釈されます。

- show interfaces g0/1
- show interfaces gi0/1
- show interfaces gig0/1
- show interfaces Gigabit0/1

人間がコマンドを打つときには便利なのですが、プログラミングで文字列処理するときには少々厄介で、
省略形式のデータを正式名称に統一したい（正規化したい）、という場面も結構あります。

たとえば `show cdp neighbors` コマンドで隣接機器の情報を調べると、
下記のようにインタフェースの名称が `Gig 0/0` と省略されて表示されます。

```
R1#show cdp neighbors
Capability Codes: R - Router, T - Trans Bridge, B - Source Route Bridge
                  S - Switch, H - Host, I - IGMP, r - Repeater, P - Phone,
                  D - Remote, C - CVTA, M - Two-port Mac Relay

Device ID        Local Intrfce     Holdtme    Capability  Platform  Port ID
R2               Gig 0/0           171              R B             Gig 0/0

Total cdp entries displayed : 1
```

一方、コンフィグでは `GigabitEthernet0/0` という名称で扱われますので、
`Gig 0/0` のような省略されたインタフェース表記が登場した場合には、
正式な名称に正規化（統一化）したいわけです。

<BR><BR>

## そもそも正式なインタフェース名ってなんだっけ？

日頃使い慣れているギガビットイーサはともかくとして、
昨今の高速なイーサネットのインタフェース名ってなんだっけ？という場面にもよく遭遇します。

サーバやストレージ機器の周辺では2ギガとか5ギガみたいなマルチギガビットが使われていたり、
アップリンクのポートでよく使われる40ギガ、100ギガみたいな高速なLANのインタフェース名って、パッとは思い浮かばないですよね。

そんな情報どっかにまとまってるんじゃないの？と誰もが考えると思いますが、
シスコ社のドキュメントを探すのは意外に難しくて、未だに見つけられずにいます。

ですが、あるところにはあるんです。

Cisco社が公開しているpyATSというPythonで作られたツールのソースコードに、
convert_intf_nameという関数がありまして、
その中に埋め込まれている辞書型データが一番まとまってると思います。

Github上の[common.py](https://github.com/CiscoTestAutomation/genieparser/blob/master/src/genie/libs/parser/utils/common.pyhttps://github.com/CiscoTestAutomation/genieparser/blob/master/src/genie/libs/parser/utils/common.py "common.py")

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

<BR><BR>

## 使い方

オリジナルの `common.py` はクラスを使っていますが、私は必要な部分だけを抜き出して使い回しています。

このリポジトリから `cisco_intf_name.py` をコピーして使う場合、そのまま実行すれば `main()` 内のテストコードが走ります。

テストコードはこんな感じです。

```python
if __name__ == '__main__':
    import sys

    def main():

        test_intfs = [
            'Gig0/0',
            'Gig 0/0',
            'Te1/0/1',
            'Twe 1/0/1',
            'Hu1/0/49',
        ]

        for intf in test_intfs:
            normalized_intf_name = convert_intf_name(intf=intf)
            print(f'{intf} is normazlied as {normalized_intf_name}')

        return 0

    sys.exit(main())
```

これを実行すると、このように出力されます（↓）

```bash
iida@s400win:~/git/intf_naming_conventions/lib$ ./cisco_intf_name.py

Gig0/0 is normazlied as GigabitEthernet0/0
Gig 0/0 is normazlied as GigabitEthernet0/0
Te1/0/1 is normazlied as TenGigabitEthernet1/0/1
Twe 1/0/1 is normazlied as TwentyFiveGigE1/0/1
Hu1/0/49 is normazlied as HundredGigE1/0/49
```

`Gig0/0`は`GigabitEthernet0/0`に変換されます。

`Gig 0/0` のようにインタフェース名と番号の間にスペースが入っても大丈夫ですね。

`Twe` は `TwentyFiveGigE` です。先頭二文字 `Tw` だけにしまうと `TwoGigabitEthernet` になってしまいますので要注意です。

このコードを持っておけば、パラメータシートをYAMLで書いたりする場面で長い正式名称を使わずとも省略表記で書けるようになって便利です。

<BR><BR>

## おまけ

手っ取り早く試せるように Google Colaboratory にも作成しておきました。
下記の Open in Colab というアイコンをクリックすると、このコードをすぐに試せます。
`test_intfs = []` のところに正規化したいインタフェース名を書き並べてお試しください。

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/takamitsu-iida/intf_naming_conventions/blob/main/intf_naming_conventions.ipynb)
