---
title: トレースからコードのホットスポットを調査
kind: Documentation
further_reading:
  - link: トレーシング
    tag: Documentation
    text: APM 分散型トレーシング
  - link: tracing/profiler/getting_started
    tag: Documentation
    text: アプリケーションの継続的なプロファイラーを有効にします。
  - link: tracing/profiler/intro_to_profiling
    tag: Documentation
    text: プロファイリング入門
---
アプリケーションの本番環境でパフォーマンスの問題が発生し始めた場合、役立つ手法のひとつが APM からの分散型トレース情報をコードスタックのフルプロファイルに接続することです。APM 分散型トレーシングと Continuous Profiler の双方が有効化されたアプリケーションプロセスは自動的にリンクされるため、Code Hotspots タブでスパン情報からプロファイリングデータを直接開き、パフォーマンスの問題に関連する特定のコード行を見つけることができます。

{{< img src="tracing/profiling/code_hotspots_tab.gif" alt="Code Hotspots タブで APM トレーススパンのプロファイリング情報を確認">}}

## 前提条件

{{< programming-lang-wrapper langs="java,python" >}}
{{< programming-lang lang="java" >}}
[サービスのプロファイリングを起動する][1]と、コードのホットスポット識別がデフォルトで有効化されます。手動でインスツルメントされたコードの場合、Continuous Profiler はスパンのスコープアクティベーションを要求します。

```java
final Span span = tracer.buildSpan("ServicehandlerSpan").start();
try (final Scope scope = tracer.activateSpan(span)) { // mandatory for Datadog continuous profiler to link with span
    // worker thread impl
  } finally {
    // Step 3: Finish Span when work is complete
    span.finish();
  }

```

トレーシングライブラリのバージョン 0.65.0 以降が必要です。


[1]: /ja/tracing/profiler/getting_started
{{< /programming-lang >}}
{{< programming-lang lang="python" >}}

[サービスのプロファイリングを起動する][1]と、コードのホットスポット識別がデフォルトで有効化されます。

トレーシングライブラリのバージョン 0.44.0 以降が必要です。


[1]: /ja/tracing/profiler/getting_started
{{< /programming-lang >}}
{{< /programming-lang-wrapper >}}

## スパンからプロファイリングデータにリンクする

各トレースのビューから、選択したスパンの範囲内のプロファイリングデータが Code Hotspots タブに表示されます。

左側の詳細ビューは該当するスパンを実行している時間集計のタイプのリストです。このタイプリストの内容はランタイムと言語に応じて異なります。

- **メソッドの実行時間**は、コードの各メソッドが実行された全体の時間を示します。
- **CPU** は、CPU タスクを実行するのに費やした時間を示します。
- **同期**は、同期されたオブジェクトのロック待機に費やした時間を示します。
- **ガベージコレクション**は、ガベージコレクターの実行待ちに費やした時間を示します。
- **VM オペレーション** (Java のみ) は、ガベージコレクションに関連しない (ヒープダンプなど) VM のオペレーション待機に費やした時間を示します。
- **ファイル I/O** は、ディスクの読み取り/書き込み動作の実行待ちに費やした時間を示します。
- **ソケット I/O** は、ネットワークの読み取り/書き込み動作の実行待ちに費やした時間を示します。
- **オブジェクト待機**は、オブジェクト上の通知コール待機に費やした時間を示します。
- **その他**は、プロファイリングデータで説明不能なスパンの実行に費やした時間を示します。

これらのタイプのうちひとつをクリックし、時間を要したメソッドに対応するリスト (開始時間順) を確認します。プラス (`+`) マークをクリックすると当該メソッドのスタックトレースが**逆の順で**開きます。

**その他**の説明のつかない時間の記録が少ない (10% 未満) ことは通常ありません。その他の時間が記録される可能性は次の通りです。

  - 選択したスパンが実行に直接マッピングされていない場合。プロファイリングデータとスパンが特定のスレッドで実行される場合、この 2 つは一意の関係性を持ちます。たとえば、いくつかのスパンが作成され、一連の関連する処理ステップの仮想コンテナ一意に使用されたが、どのスレッドの実行とも直接関連していないといった場合が考えられます。
  - アプリケーションのプロセスが CPU リソースにアクセスしてそれを実行または一時停止できない場合。このとき、その他のプロセスやコンテナの競合するリソースについてプロファイラーに知らせる手段は存在しません。
  - アプリケーションがそれぞれ 10 分未満の同期または I/O イベントでロックされている場合: Java プロファイラーは一時停止された 10 分を超えるスレッドイベント (ロック、I/O、パーク) に関するデータを受信します。このしきい値を引き下げたい場合は、[セットアップのデフォルト値変更についてのドキュメント][1]を参照してください。
  - 選択したスパンが短い場合。プロファイリングは定期的にコードの状態をチェックするサンプリング機構であるため、スパンが 50ms より短い場合はそのスパンについての十分なデータが得られないことがあります。
  - インスツルメンテーションが欠けている場合: 詳細なプロファイリングを行うには、ScopeManager でスパンを有効化し、スパンを実行スレッドに関連させる必要があります。カスタムインスツルメンテーションの中にはこれらのスパンを適切に有効化しないものがあり、この場合はスパンを実行スレッドにマッピングできません。このスパンがカスタムインテグレーションに由来する場合は、[カスタムインスツルメンテーションのドキュメント][2]でその改善についての情報を参照してください。

## プロファイルをトレースから閲覧する

{{< img src="tracing/profiling/flamegraph_view.gif" alt="フレームグラフでプロファイルビューを開く">}}

詳細画面の各タイプについて、**View profile** をクリックするとそのデータをフレームグラフ形式で表示させることができます。**Span/Trace/Full profile** セレクタをクリックして、データのスコープを定義します。

- **Span** は、プロファイリングデータのスコープを以前に選択されたスパンに設定します。
- **Trace** は、プロファイリングデータのスコープを、以前に選択されたスパンとサービスプロセスが同じすべてのスパンに設定します。
- **Full profile** は、データのスコープを以前に選択されたスパンを実行したサービスプロセス全体の 60 秒に設定します。

## その他の参考資料

{{< partial name="whats-next/whats-next.html" >}}

[1]: /ja/tracing/profiler/profiler_troubleshooting#reduce-overhead-from-default-setup
[2]: /ja/tracing/setup_overview/custom_instrumentation/java#manually-creating-a-new-span