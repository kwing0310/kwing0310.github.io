---
title: "電話機で Mastodon に投稿してみた"
slug: "sip-mastodon"
description: "OpenWrt ルータに Asterisk サーバを構築し，自動応答と Python を組み合わせて電話機からポケベル打ちで Mastodon に投稿できるようにしました．"
date: "2024-12-21"
---

> 🎄 この記事は [Fediverse (2) Advent Calendar](https://adventar.org/calendars/10064) 17日目の記事です．

みなさんこんにちは．Mastodon サーバ [Chillout Chat](https://chillout.chat) 管理人 (技術担当) の Karasuma です．今回はサーバの運営とは関係ありませんが，少しマニアックな話題をお届けします．

<details>
  <summary>Chillout Chat (略称: チルチャ) について</summary>
  <p>Chillout Chat は「音楽好きの人が集まる，chill out ≒ 落ち着き・くつろぎ の空間」をコンセプトにしている Mastodon サーバです．2021年から3年以上の運用実績があり，来年の春には4周年を迎えます．</p>
  <p>実装されている独自機能など，詳しい情報は <a href="https://info.chillout.chat">info.chillout.chat</a> に掲載しています．</p>
</details>

## 前置き

2006年生まれの私は，おそらくスマホ普及前の様子を覚えている最後の世代であり，公衆電話網を利用したサービスの衰退を見ながら育ってきました．その中でも2019年9月末に個人向け無線呼び出しサービス (いわゆる「ポケベル」) が終了したときのことは強く印象に残っています．サービス最終日には[「ポケベル終了を見守る配信」](https://blog.nicovideo.jp/niconews/119196.html) と題した番組がニコ生で放送されており，番組内で電話番号を公開していたポケベルにメッセージを送ったことが，自分にとって最初で最後のポケベルに触れた体験でした．

「2 タッチ入力」「ポケベル打ち」などと呼ばれるメッセージの記法は今でも一部の IME に搭載されているほか，NTT ドコモの電話から SMS を送るサービスにも採用されています．SMS だけでなく Mastodon や Misskey にもポケベル打ちで投稿できれば面白いかもしれません．

## 完成形

<iframe src="https://chillout.chat/@wing/113691014657231914/embed" class="mastodon-embed" style="max-width: 100%; border: 0" width="400" allowfullscreen="allowfullscreen"></iframe><script src="https://chillout.chat/embed.js" async="async"></script>

この動画のように，自宅にあるビジネス向け IP 電話機から自動応答システム (内線 `115`) に接続することで Mastodon に投稿を送信しています．

システムが応答したらポケベル打ちでメッセージ本文「ハヤヤッコ」を入力し，最後に `#` を押します．すると入力内容が自動的に投稿され，完了のメッセージとともに電話が切断されます．

(ガイダンス音声は [VOICEVOX ナースロボ_タイプT](https://voicevox.hiroshiba.jp/dormitory/nurserobo_typet/) で作成しました)

### システムの全体像

<img src="/images/sip-mastodon-system.png">

自宅の LAN 内に VoIP (Voice over IP) を利用した内線電話網を構築し，その中心となる内線交換ソフトウェア (IP-PBX) に自動応答の機能を付加することで実現しています．VoIP のプロトコルには SIP を採用しており，IP-PBX は OSS の [Asterisk](https://www.asterisk.org/) を使っています．

入力した数字は Python スクリプトに渡され，メッセージの変換と API を利用した Mastodon への投稿が行われます．

動画で登場したビジネスフォンは中古で入手した AVAYA 9611G ですが，LAN に接続しているスマホなどの一般的な端末でも，SIP に対応したソフトフォンがあれば同様の操作をすることができます． 

## システムの構築

### OpenWrt に Asterisk・Python を導入

Asterisk はリソースが若干余り気味になっている [OpenWrt](https://openwrt.org/) を導入したルータにインストールしました．OpenWrt はオープンソースのネットワーク機器向け Linux ディストリビューションであり，[GL.iNet](https://www.gl-inet.com/) などの製品には標準で搭載されているほか，ファームウェアの書き換えによって導入できる既存の機器もたくさんあります．

Asterisk・Python を扱う上で必要なパッケージをインストールします．(古いバージョンの OpenWrt では apk の代わりに opkg を使います)

```
$ apk add asterisk asterisk-pjsip asterisk-bridge-simple asterisk-codec-alaw asterisk-codec-ulaw asterisk-res-rtp-asterisk asterisk-format-gsm asterisk-format-pcm
```

自動応答を実装する上で必要になる Asterisk のアプリケーションも導入します．

```
$ apk add asterisk-app-verbose asterisk-app-system asterisk-app-read
```

次に Python をインストールします．Mastodon の API を簡単に利用するために [Mastodon.py](https://github.com/halcy/Mastodon.py) を，Mastodon の認証情報を `.env` に保存するために [python-dotenv](https://github.com/theskumar/python-dotenv) を使います．

```
$ apk add python3-light python3-pip
```

```
$ pip3 install Mastodon.py python-dotenv
```

機器のフラッシュメモリが 20MB ほどあれば展開できますが，使うパッケージをさらに吟味する・自前でビルドする (未検証) などといった工夫はできそうです．

### Asterisk サーバの設定

`/etc/asterisk` にある設定ファイル `pjsip.conf` と `extensions.conf` を編集します．前者の内容は自動応答を実装しない場合と変わらないため割愛します．以下は `extensions.conf` の自動応答に関する部分の抜粋です．

```
; 115: Fediverse Post Dial
exten => 115,1,Answer()
same => n,Verbose(2,Caller: ${CALLERID(num)})
same => n,Wait(3)
; こちらは Fediverse Post Dial です
same => n,Playback(/etc/asterisk/sounds/postdial1)
same => n,Wait(1)
; メッセージを入力し，最後にシャープを押してください
same => n,Read(Digits,/etc/asterisk/sounds/postdial2,0,e,1,0)
same => n,Verbose(2,Input: ${Digits})
same => n,System(python3 /root/asterisk-config/fedi-post.py ${Digits})
; ご利用ありがとうございました
same => n,Playback(/etc/asterisk/sounds/thanks)
same => n,Hangup()
```

自動で再生する音声ファイルはサンプリングレート 8kHz，モノラル，ulaw 形式で保存します．`extensions.conf` に音声ファイルのパスを書く際は拡張子を含めないことに注意が必要です．

### AVAYA 9611G の設定

LAN 内に立てた Web サーバに `96x1Supgrade.txt` `46xxsettings.txt`, を配置し，その中に Asterisk サーバの IP アドレス・ポートなどといった接続情報を書いて読み込ませるとスムーズに設定が進みます．詳しくは [公式ドキュメント](https://documentation.avaya.com/ja-JP/bundle/IPOfficeBranchMigrate/page/ConvertingAll96x1PhonesToSIP.html) を参照してください．

OpenWrt にはブラウザでアクセスする GUI の設定画面 ([LuCI](https://openwrt.org/docs/guide-user/luci/start)) が付属しており，LuCI を動かすための Web サーバとして [uhttpd](https://openwrt.org/docs/guide-user/services/webserver/uhttpd) がプリインストールされています．今回は設定ファイルを uhttpd のルートディレクトリである `/www` に配置しました．

### Python スクリプトの実装

`*2*2` が初めに入力された場合は文字に変換し，そうでない場合は数字をそのまま投稿するようにします．また，英小文字や「ッ」「ャ」「ュ」「ョ」などは直前に `*4` を入力することで識別しています．数字と文字との対応関係は `pokebell-code.json` に記載し，スクリプトから読み込むようにします．

`.env`

```
CLIENT_KEY=XXXXXXXXXXXXX
CLIENT_SECRET=XXXXXXXXXXXXX
ACCESS_TOKEN=XXXXXXXXXXXXX
```


`fedi-post.py`

```
#!/usr/bin/python

import os
import sys
import json
from mastodon import Mastodon
from dotenv import load_dotenv

digits = sys.argv[1]
text = ''
size = len(digits)
print('input:', digits)

load_dotenv()
client_key = os.getenv('CLIENT_KEY')
client_secret = os.getenv('CLIENT_SECRET')
access_token = os.getenv('ACCESS_TOKEN')

api = Mastodon(
    api_base_url  = 'https://nightly.chillout.chat',
    client_id     = client_key,
    client_secret = client_secret,
    access_token  = access_token
)

if digits[0:4] == '*2*2':
    i = 4
    with open('/root/asterisk-config/pokebell-code.json') as f:
        code = json.load(f)

    while i < size:
        if digits[i:i+2] == '*4':
            text += code['small'][digits[i+2:i+4]]
            i += 4
        else:
            text += code['normal'][digits[i:i+2]]
            i += 2
else:
    text = digits

print('output:', text)
api.toot(text)
```

`pokebell-code.json`

```
{
    "normal": {
        "11": "ア", "12": "イ", "13": "ウ", "14": "エ", "15": "オ", ... (省略)
    },
    "small": {
        "11": "ァ", "12": "ィ", "13": "ゥ", "14": "ェ", "15": "ォ", ... (省略)
    }
}
```

いずれも `/root/asterisk-config` に配置しました．

## 今後の展望

今回はあくまでデモンストレーションのため，単一アカウントで投稿するだけというシンプルな形になりましたが，タイムラインの読み上げなどを実装するとさらに実用的になるかもしれません．