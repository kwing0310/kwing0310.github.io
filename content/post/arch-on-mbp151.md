---
title: "MacBook Pro 15,1 (2018) に Arch Linux をインストールする手順"
slug: "arch-on-mbp151"
description: "Apple T2 セキュリティチップを搭載した MacBook Pro 15,1 (2018) に Arch Linux をシングルブートでインストールしてみました。"
date: "2023-09-04"
---

<img style="margin-bottom: 0;" src="/images/arch-on-mbp151-cover.jpg" alt="Running Arch Linux on MacBook Pro 15,1">

<p style="text-align: center; font-size: 0.8em;">Wallpaper: <a href="https://www.pixiv.net/artworks/103383813">Arch ちゃん by Ravimo</a> (CC-BY 4.0)</p>

## Introduction

2018 年から 2020 年夏までの間に発表された Intel Mac には Apple T2 セキュリティチップが搭載されており、サードパーティ製の OS が簡単に動作しないようになっています。T2 チップに対応した Linux カーネルやドライバーを開発するコミュニティ主導の [t2linux](https://t2linux.org/) プロジェクトが進み、十分に実用できるほどの段階になってきたため、私のメイン機である MacBook Pro 15,1 (2018) に Arch Linux をインストールしてみました。

### btw I use Arch

4 年以上使い続けた macOS が素晴らしい OS であることは言うまでもありませんが、今後のメイン機の利用目的を考えると Arch Linux をクリーンインストールしても良いのではないか、という考えに至りました。詳しい経緯については別の記事に書きたいと思いますが、今伝えたいのは「相当な理由が無い限り、T2 Mac に Linux をインストールすることは避けるべきである」ということです。ただでさえ大変な Arch Linux のインストール手順に T2 Mac 特有の作業が必要になるわけですから、覚悟が必要なのは当然と言ってもいいかもしれませんね。

### ⚠ Caution

- この記事は 2023 年 9 月現在の情報をもとに書かれています。インストールする前に [t2linux wiki](https://wiki.t2linux.org/) で最新の情報を確認することを**強く推奨**します。
- この記事に掲載された内容によって生じた損害等の一切の責任を負いかねますのでご了承ください。試す際は**自己責任**でお願いします。

## 下準備

基本的には [Pre-Install - t2linux wiki](https://wiki.t2linux.org/guides/preinstall/) に従って進めていく形となります。今回は実験的に **Arch Linux のみのシングルブート**を試してみましたが、Mac のファームウェアの更新 (ソフトウェアアップデート) が macOS 経由でしかできないため、macOS とのデュアルブートをおすすめします。

### macOS を最新の状態にアップデート

macOS の更新にはデバイスのファームウェアの更新も含まれています。後からファームウェアを更新するのは面倒なので、まずはじめに macOS を最新の状態にアップデートしておきます。

### 内蔵 SSD のバックアップ

インストールする段階でディスクのフォーマットを行うため、内蔵 SSD 内のデータはすべて失われることになります。そのため、あらかじめ安全な場所にバックアップを保管しておきます。外付けドライブにバックアップを保管する場合はフォーマット形式に注意しましょう。今回はシングルブートのため、下準備の段階ではパーティションの設定を行っていません。

### macOS インストールメディアの作成

インターネット経由で macOS を再インストールすることはできるようですが、念の為手元にもインストールメディア (USB メモリ) を用意しておきました。[t2linux wiki の FAQ](https://wiki.t2linux.org/roadmap/#can-i-completely-remove-macos/) でもこの方法が推奨されています。詳しい作り方については [Apple 公式の情報](https://support.apple.com/ja-jp/HT201372) を参照してください。

### Arch Linux インストールメディアの作成

今回は t2linux プロジェクトが公開している ISO イメージ [archlinux-t2-2023.07.21](https://github.com/t2linux/archiso-t2/releases/tag/2023.07.21) を使いました。他のディストリビューションについても [Selecting an ISO - t2linux wiki](https://wiki.t2linux.org/guides/preinstall/#selecting-an-iso) に T2 Mac に対応した ISO イメージの一覧が掲載されています。あとはいつも通り [USBImager](https://gitlab.com/bztsrc/usbimager/) や [balenaEtcher](https://etcher.balena.io/) などで USB メモリなどにイメージを書き込むだけです。

### Wi-Fi ファームウェアのコピー

Linux の Wi-Fi ドライバーは macOS の Wi-Fi ファームウェアと同じものを使っています。つまり、**この手順を忘れると Wi-Fi や Bluetooth が使えません。** 下準備の段階では [Wi-fi and Bluetooth - t2linux Wiki](https://wiki.t2linux.org/guides/wifi-bluetooth/#on-macos) の On macOS 部分のみの作業を行います。


### セキュアブートの無効化

最後にインストールメディアから Arch Linux が実行できるように、[Disable Secure Boot - t2linux wiki](https://wiki.t2linux.org/guides/preinstall/#disable-secure-boot) に従って Mac のセキュアブートを**無効化**します。当然のことながら、セキュアブートを切っている間は Mac のセキュリティに十分注意してください。

以上の作業がすべて終わったら、いよいよインストールメディアから Arch Linux をインストールしていきます。

## インストール

[インストールガイド - ArchWiki](https://wiki.archlinux.jp/index.php/%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%82%AC%E3%82%A4%E3%83%89) と [Installation (Arch) - t2linux wiki](https://wiki.t2linux.org/distributions/arch/installation/) を見ながらインストールを進めていきます。なお、一般的な Arch Linux のインストール手順と同じ部分については説明を省略しますのでご了承ください。

### パーティション

/dev/nvme0n1 が Mac の内蔵 SSD です。既に EFI 用の /dev/nvme0n1p1 が用意されているため、スワップとファイルシステム用のパーティションを準備していきます。EFI 用のパーティションもマウントが必要です。今回は以下のように設定しました。

|マウントポイント|パーティション|パーティションタイプ|容量|
|---|---|---|---|
|/mnt/boot|/dev/nvme0n1p1|EFI システムパーティション|300MiB|
|swap|/dev/nvme0n1p2|Linux swap|18GiB|
|/mnt|/dev/nvme0n1p3|Linux x86_64 root (/)|残り容量全て|

### 必須パッケージのインストール

インストールするカーネルが linux-t2 となるなど、必須パッケージが通常と異なるため、ArchWiki の「必須パッケージのインストール」部分は t2linux wiki に記載されている 5 番目の手順に置き換えます。

### ブートローダーの設定

t2linux wiki に記載されている 7 番目と 8 番目の手順に従います。今回は systemd-boot を使いましたが、おそらく GRUB も動くかと思います (未検証) 。

ブートローダーの設定まで終わったら再起動して、いつも通りネットワークの設定や GUI の有効化など、諸々の作業を行います。デスクトップ環境についてはディスプレイマネージャー [SDDM](https://wiki.archlinux.jp/index.php/SDDM) と Wayland コンポジタ [Sway](https://wiki.archlinux.jp/index.php/Sway) の組み合わせで運用しています。

## 1 週間使ってみて

メイン機に Arch Linux をシングルブートした状態で 1 週間ほど使い続けてみましたが、特に大きな問題などはありませんでした。現時点での各デバイスの動作状況については以下に記しておきます。

### スクリーン・キーボード・Web カメラ・USB ポート

問題なく使えます。

### Wi-Fi & Bluetooth

十分に使えますが、Wi-Fi と Bluetooth スピーカー・イヤホンを同時に使用したときに音の途切れが目立ちます。Wi-Fi を無効化して有線 LAN での接続に切り替えれば問題ありません。

### Touch Bar

Vimmer の私にとって Esc キーが使えるかどうかは死活問題でしたが…… **なんと動きました。** アプリケーションごとに表示を変えるなどの芸当は macOS 側の機能なのでできませんが、Esc キーや明るさ、音声などを制御するボタン類が常に表示されています。fn キーで F1 ~ F12 に切り替えることもできます。現状では Esc と F1 ~ F12 しか動きませんが、いろいろと工夫すれば他のボタンも機能しそうです。

### トラックパッド

こちらも動きました。深く押し込んだり、3 本指以上で操作したりすることはできませんが、十分使えるレベルにまで達しています。

### 内蔵スピーカー

一応再生はできますが、音質は macOS のときと比べて著しく悪いです。おそらく MacBook Pro 15,1 に合わせた DSP (Digital Signal Processing) の設定ができていないからであると思われます。外部のスピーカーに切り替えれば問題ありません。

デバイスの対応状況については [Device Support and State of Features - t2linux wiki](https://wiki.t2linux.org/state/) にも記載されています。

## 結び

先述したとおり、各種デバイスについては少ない設定で十分に動作しています。T2 Mac に Linux をインストールすることも実用的な選択肢としてあり得ることが分かりました。カーネルや各種ドライバーをメンテナンスしてくださっている t2linux コミュニティの方々に感謝の意を表し、記事の結びといたします。
