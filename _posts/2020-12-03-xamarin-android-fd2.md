---
title: Xamarin Android Fd2
date: 2020-12-03 00:00:00 +09:00
published: false
---

---
title: "Xamarin.Android fast deployment v2について"
type: "tech"
topics: ["Xamarin", "Android", "MSBuild"]
published: false
---

## これは何?

[Xamarin Advent Calendar 2020](https://qiita.com/advent-calendar/2020/xamarin)、[田淵さん](https://blog.ytabuchi.dev/entry/2020/12/02/110000)から引き継いでのエントリーです。3日目は、2020年にxamarin-androidに追加されたFast Deployment V2についてかる〜く書きます。昔この辺の開発をやっていたので、現役のメンバーがどういう仕事をしたかを解説する感じです。

## 前提知識

Fast Deploymentというのは、AndroidアプリケーションであるところのAPKを可能な限り再ビルド・再インストールせずに、デバッグ対象のAndroidターゲット（デバイスまたはエミュレーター）に必要最小限のファイルのみをコピー（デプロイ）して実行する仕組みです。APKをビルドしてインストールする全工程は長くて待っていたくない仕事なのです。

ビルドとデプロイメントを高速化することで開発体験を良くしようというものは、有名どころではFlutterのHot Reloadなんかがそうですが、当然ながらFlutterのアレはそれまで培われてきた技術からなる巨人の肩の上に乗っかったものに過ぎません。これまで何度か自分がビルドとデプロイメントの高速化に関する記事を書いてきたのと、他の人が書いてきたのがあるのを、いくつか列挙するので、初めて知ったという方は参考にしてください。

- [インサイドXamarin (9) Xamarin.Androidのアプリケーションの作成、ビルド、実行とデバッグ](https://www.buildinsider.net/mobile/insidexamarin/09) (2014年)
- [Android Studio 2.0のInstant Runの仕組みを解読する](https://qiita.com/atsushieno/items/141f790a9bbaa93b62c1) (2016年)
- [Xamarin.AndroidでもInstant Run (cold swap) がしたい!](https://qiita.com/atsushieno/items/2da51b5614111c10c763) (2016年)
- [安定的で合理的なデバッグ作業を実現するApply Changes @ C96](https://atsushieno.hatenablog.com/entry/2019/08/01/232116) (2019年. このエントリ自体は同人誌の宣伝)
- [Xamarin.Forms でも HotReload がしたい！](https://qiita.com/amay077/items/150f484e68924468a2c3) (2019年 あめいさんのやつ。今気づいたけどこれタイトルパク(ry)((元ネタがかぶっただけだと思う))

そもそもXamarin.Androidのfast deploymentだって[Titanium Mobileで類似機能が実装されたほうが先](https://devblog.axway.com/mobile-apps/titanium-mobile-intro-series-fastdev-for-android/)だし、実行中のコードの差し替えなんてそれこそVisual Studio .NET（2001年）のEdit & Continueとか、それくらい昔からある技術ではあります。そして2020年の今でもAndroid StudioでJVMTIを拡張して[Structural Class Redefinition and Apply Changes](https://medium.com/androidde20velopers/structural-class-redefinition-and-apply-changes-30f96f1962e6)なんて実装してメジャーアップデート項目のひとつに列挙しているくらいhot topicです。

Xamarin.Androidでは、列挙した最初の記事を書いた2014年にはもうFast Deploymentと呼ばれる機能の初期版が出ていました。Xamarin.Androidでは仕組み上アプリケーションコードは主に.NETアセンブリであり、これはロードするパスさえ調整すればAPKを入れ替えずにそのDLLだけ一時的にデバッグ対象のAndroidターゲットにデプロイしてアプリケーションを再起動してしまえば、普通に新規インストールしたときと同じようにデバッグ出来たというわけです。

Instant Runが出た後、それに類似するBazelのmobile deploymentの仕組みを流用したImproved Fast Deploymentという機能も実装されました((というかわたしが作っていたのですが))。アプリの全面的な差し替えしか出来なかったJava言語のAndroid環境とは異なり、既にDLLの実行中差し替えが可能だったXamarin.Android上でのメリットはJavaコードの更新を伴う場合に限られたので、あんまし真面目に取り合われずにデフォルト無効のままだったのですが、機能としては存在していました。これは2019年頃にまたいろいろ書き換えられてデフォルト有効になったと理解しています。

## Android 11のscoped storage accessへの対応

2020年にはAndroid 11がリリースされたわけですが、Android 11では新たにアプリケーションからのローカルファイルへのアクセスが大きく制限されることになりました。Scoped Storage Enforcementという言い方で説明されることが多いです。

https://developer.android.com/about/versions/11/privacy/storage

どういうことになったか、ざっくり雑に説明すると、Android 11以降をtargetSdkVersionとして指定するアプリケーションでは、自ら作成したわけでもないファイルは読み書きできなくなり、scoped storageの流儀に従わないといけなくなりました。

これがXamarin.Androidのfast deploymentを直撃しました。これまでfast deploymentで更新されたアプリケーションのDLLはAndroidターゲット上の適当な一時ディレクトリにアップロードされていましたが、それが不可能になったのです。

これを解決するためには、アプリケーションのローカルスコープのディレクトリ（`/data/data/my.app.package`のようなディレクトリ）にファイルを作成しなければならなくなり、そのためにはadbのrun asを使うなどややこしい手間が必要になりました。API Level 20以降であればフツーはrun asが使えるはずです。

そういうわけで、この新しい制限があっても動作するような仕組みに書き換えられたのがfast deployment V2というわけです。大したことはないですね。FD v2の話自体は大した意味はないですが、scoped storageの変更のインパクトは割と大きいので、AndroidプラットフォームにKotlinやJavaで生きている人にはそこそこ知られていて、Xamairnで開発している人にとっても他人事ではないので、気をつけておいたほうが良いです。

ちなみに、多分古いfast deploymentではネイティブライブラリの更新などがちゃんと機能していなかったと思うのですが、V2はサポートしているようです。（ドキュメント上ではそうなっている）

参考資料: https://github.com/xamarin/xamarin-android/pull/4690

## What's next...

例年に比べて今年は格段に短い内容で終わらせました。明日はsecileさんです。

