<div align=center>
    <img src="./folia.png">
    <br /><br />
    <p>専用サーバーに範マルチスレッディングを追加した、<a href="https://github.com/PaperMC/Paper">Paper</a> のフォークです。</p>
</div>

## はじめに
このフォークは日本語翻訳を主としています。

## Overview

Foliaは、近くにあるロードされたチャンクをグループ化して「独立した地域」を形成します。Foliaが近くのチャンクをどのようにグループ化するかの詳細は、[PaperMCのドキュメント](https://docs.papermc.io/folia/reference/region-logic)を参照してください。各独立した地域はそれぞれ独自のティックループを持っており、通常のMinecraftのティックレート（20TPS）で処理されます。これらのティックループは、並行してスレッドプール上で実行されます。もはやメインスレッドは存在せず、各地域はそれぞれの「メインスレッド」を持ち、ティックループ全体を実行します。

プレイヤーが広範囲に散らばっているサーバーでは、Foliaは多くの散らばった地域を作成し、設定可能なサイズのスレッドプール上でそれらを並行してティックします。したがって、このようなサーバーではFoliaはうまくスケールするはずです。

Foliaは独自のプロジェクトでもあり、予見可能な将来、Paperに統合されることはありません。

より詳細だが抽象的な概要は、[プロジェクト概要](https://docs.papermc.io/folia/reference/overview)をご覧ください。

## よくある質問

## Foliaの利点を享受できるサーバータイプは何ですか？
プレイヤーが自然と散らばるようなサーバータイプ、例えばスカイブロックやSMPなどがFoliaの恩恵を最も受けられます。また、それなりの規模のプレイヤー数がいるサーバーであるべきです。

## Foliaはどのようなハードウェアで最も適切に動作しますか？
理想的には、少なくとも16 コア（スレッドではない）が必要です。

## Foliaを最適に設定する方法は？
まず、ワールドは事前に生成されていることが推奨されます。そうすることで、必要とされるチャンクシステムワーカースレッドの数を大幅に減らすことができます。

以下は、Foliaがテストサーバーでリリースされる前に行ったテストに基づく_非常に大雑把な_見積もりです。そのテストサーバーではピーク時に約330人のプレイヤーがいました。したがって、これは正確な数値ではなく、さらなる調整が必要です。あくまで出発点として考えてください。

マシンに利用可能なコアの総数を考慮に入れる必要があります。その後、以下のためにスレッドを割り当てます：

netty IO：200-300人のプレイヤーにつき約4スレッド
チャンクシステムIOスレッド：200-300人のプレイヤーにつき約3スレッド
事前に生成された場合のチャンクシステムワーカー：200-300人のプレイヤーにつき約2スレッド
事前に生成されていない場合のチャンクシステムワーカーについては最良の推測はありません。なぜなら、実際に実行したテストサーバーでは16スレッドを割り当てましたが、約300人のプレイヤーがいる時点でチャンクの生成は依然として遅かったからです。
GC設定：???? しかしながら、GC設定は並列スレッドを割り当てますし、正確な数を知る必要があります。これは通常 -XX:ConcGCThreads=n フラグを通じて設定されます。このフラグを -XX:ParallelGCThreads=n と混同しないでください。なぜなら、並列GCスレッドはGCによってアプリケーションが一時停止されたときのみ実行されるため、考慮に入れるべきではないからです。
それらの割り当てを行った後、システム上の残りのコアを80%までの割り当て（割り当てられた総スレッドが利用可能なCPUの80%未満）で、tickthreads（グローバル設定の下、threaded-regions.threads）に割り当てることができます。

80%以上のコアを割り当てない理由は、プラグインやサーバー自体が予測もしくは設定できない追加のスレッドを利用する可能性があるためです。

さらに、上記はすべてプレイヤー数に基づいた大雑把な推測であり、実際にはスレッド割り当てが理想的でない可能性が高く、実際のスレッド使用状況に基づいて調整する必要があるでしょう。

## プラグインの互換性
もはやメインスレッドは存在しません。Foliaで機能するためには、すべての プラグインが_ある程度の_ 改変が必要になると予想します。さらに、いかなる種類の マルチスレッディングはプラグインが保持するデータにおける競合状態を引き起こす可能性があるため、変更が必要になることは避けられないでしょう。

したがって、互換性についての期待はしないでください。

## プラグインの互換性

メインスレッドはもうありません。私は、存在する_すべての_単一のプラグインが、Foliaで機能するために_ある程度の_改変を必要とすると予想しています。さらに、_あらゆる種類の_マルチスレッディングは、プラグインが保持しているデータにおける競合状況を引き起こす可能性があります。そのため、変更が必要になることは避けられません。

ですので、互換性に関する期待はゼロに設定してください。

## APIの計画

現在、多くのAPIはメインスレッドに依存しています。
私は、Paperと互換性のあるプラグインがFoliaと互換性を持つものは基本的にゼロだと予想しています。しかし、FoliaのプラグインがPaperと互換性を持つようにAPIを追加する計画があります。

たとえば、Bukkitスケジューラです。Bukkitスケジューラは本質的に単一のメインスレッドに依存しています。FoliaのRegionSchedulerとFoliaのEntitySchedulerは、ある地域が「所有する」ロケーションまたはエンティティの「次のティック」にタスクのスケジューリングを可能にします。これらは通常のPaper上で実装することができますが、メインスレッドにスケジュールされるのを除いて、どちらの場合もタスクの実行はロケーションまたはエンティティを「所有する」スレッドで行われます。この概念は一般的に適用され、現在のPaper（シングルスレッド）は全ての世界の全てのチャンクを包含する巨大な「地域」のように見ることができます。

このAPIを直接Paper自体に追加するか、それともPaperlibに追加するかはまだ決定されていません。

## 新しいルール

まず、Foliaは多くのプラグインを破壊します。どのプラグインが動作するかをユーザーが判断できるようにするため、作者によって明示的にFoliaで動作するとマークされたプラグインのみがロードされます。プラグインのplugin.ymlに「folia-supported: true」と記述することで、プラグイン作者はそのプラグインが地域化マルチスレッディングと互換性があるとマークすることができます。

もう一つ重要なルールは、地域が_並列に_ティックすることであり、_同時に_ではないということです。彼らはデータを共有せず、データを共有することを期待しておらず、データの共有は_確実に_データの破損を引き起こします。
ある地域で実行されているコードは、どんな状況でも他の地域にあるデータにアクセスしたり変更したりしてはいけません。マルチスレッディングが名前に含まれているからといって、すべてがスレッドセーフになったわけではありません。実際には、これを実現するためにスレッドセーフにされたものは_わずか_です。時間が経つにつれて、スレッドコンテキストチェックの数は増え続けるでしょう。たとえそれがパフォーマンスのペナルティを伴うとしても、バグだらけのサーバープラットフォームを誰も使ったり開発したりすることはありませんし、これらのバグを防ぎ発見する唯一の方法は、不正なアクセスをその発生源で_厳しく_失敗させることです。

これは、Folia互換のプラグインがRegionSchedulerやEntitySchedulerのようなAPIを活用して、そのコードが正しいスレッドコンテキストで実行されていることを確実にする必要があることを意味します。

一般的に、ある地域がイベントの発生源から約8チャンクの範囲内のチャンクデータを所有していると考えるのは安全です（例えば、プレイヤーがブロックを破壊した場合、そのブロックの周りの8チャンクにアクセスできるでしょう）。しかし、これは保証されているわけではありません - プラグインは、正しい動作を保証するために、これから登場するスレッドチェックAPIを利用するべきです。

スレッドセーフティの唯一の保証は、特定のチャンクのデータを単一の地域が所有しているという事実から来ています - そしてその地域がティックしている場合、そのデータに完全にアクセスできます。このデータは具体的にはエンティティ/チャンク/POIデータであり、どのプラグインのデータとも全く関係ありません。

通常のマルチスレッディングのルールは、プラグインが自身のデータや他のプラグインのデータを保存/アクセスする場合に適用されます - イベント/コマンドなどは、地域が_並列に_ティックしているため_並列に_呼び出されます（同期的な方法で呼び出すことはできません。これはデッドロックの問題を引き起こし、パフォーマンスを制限するからです）。これには簡単な解決策はありません。それは完全にアクセスされているデータに依存します。時にはConcurrentHashMapのような並行コレクションが十分かもしれませんが、しばしば不注意に使用された並行コレクションはスレッディングの問題を_隠す_だけであり、その後のデバッグがほぼ不可能になります。

### 現在のAPI追加

API追加を正しく理解するためには、[プロジェクト概要](https://docs.papermc.io/folia/reference/overview)を読んでください。

- BukkitSchedulerの代わりとして機能するRegionScheduler、AsyncScheduler、GlobalRegionScheduler、およびEntityScheduler。 エンティティスケジューラーはEntity#getSchedulerを通じて取得され、その他のスケジューラーはBukkit/Serverクラスから取得できます。
- 現在ティックしている地域が位置やエンティティを所有しているかをテストするためのBukkit#isOwnedByCurrentRegion

### APIのスレッドコンテキスト

API追加を正しく理解するためには、[プロジェクト概要](https://docs.papermc.io/folia/reference/overview)を読んでください。

一般的なルール：

1. エンティティ/プレイヤーのコマンドは、そのエンティティ/プレイヤーを所有する地域で呼び出されます。コンソールコマンドはグローバル地域で実行されます。

2. 単一のエンティティに関連するイベント（例えばプレイヤーがブロックを破壊する/設置する）は、そのエンティティを所有する地域で呼び出されます。エンティティに対するアクションを伴うイベント（例えばエンティティのダメージ）は、対象エンティティを所有する地域で呼び出されます。

3. イベントのasync修飾子は非推奨になりました - 地域から、またはグローバル地域から発火されるイベントは、もはやメインスレッドがないにも関わらず_同期的_と考えられます。

### 現在壊れているAPI

- ポータルとのインタラクション、プレイヤーのリスポーン、一部のプレイヤーのログインAPIなど、多くのAPIが壊れています。
- スコアボードAPIはすべて壊れていると考えられます（これはまだ適切に実装する方法を見つけられていないグローバルステートです）。
- ワールドのロード/アンロード
- Entity#teleport。これは_どんな状況でも_戻ってこないでしょう。代わりにteleportAsyncを使用してください。
- その他多くのものが壊れている可能性があります。

### 予定されているAPI追加

- 適切な非同期イベント。これにより、イベントの結果を後で、異なるスレッドコンテキストで完了させることができます。これは、地域外のチャンクデータにアクセスする際に非同期のチャンクロードが必要なため、スポーン位置の選択などの実装に必要です。
- ワールドのロード/アンロード
- その他

### 予定されているAPI変更

- 徹底したスレッドチェックが全面的に行われます。これは、プラグイン開発者がサーバーのランダムな部分を完全に_診断不可能な_方法で壊すかもしれないコードを出荷することを防ぐために絶対に必要です。
- その他

### Maven情報
* Mavenリポジトリ（folia-api用）:
```xml
<repository>
    <id>papermc</id>
    <url>https://repo.papermc.io/repository/maven-public/</url>
</repository>
```
* アーティファクト情報：
```xml
<dependency>
    <groupId>dev.folia</groupId>
    <artifactId>folia-api</artifactId>
    <version>1.20.1-R0.1-SNAPSHOT</version>
    <scope>provided</scope>
</dependency>
 ```


## ライセンス
PATCHES-LICENSEファイルは、apiおよびサーバーパッチのライセンスについて説明しています。これらは./patchesおよびそのサブディレクトリにあり、特に注記されていない限りそれに該当します。

このフォークは、[こちら](https://github.com/PaperMC/paperweight-examples)で見つかるPaperMCのフォークの例に基づいています。したがって、このプロジェクトにはそれに対する変更が含まれていますので、変更されたファイルのライセンス情報についてはリポジトリをご覧ください。


## License
The PATCHES-LICENSE file describes the license for api & server patches,
found in `./patches` and its subdirectories except when noted otherwise.

The fork is based off of PaperMC's fork example found [here](https://github.com/PaperMC/paperweight-examples).
As such, it contains modifications to it in this project, please see the repository for license information
of modified files.
