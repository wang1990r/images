## I. 背景

新機能開発とPR修正を同時に進めているため、頻繁にブランチを切り替えらなければならないことになりました。

ただ、毎回ブランチを変えると、ビルドが走って、数分間が何にもできない状態になりまして、非常にイライラしました。

ビルドの時間を削減するため、諸々試しました。


## II. 試し方

以下のコマンドで、gradleがビルドのレポートを生成してくれるので、コマンドで試すことにしました。

```
./gradlew app:assembleDebug --profile
```

生成されたのレポート位置は `project/build/reports` になります。

![report.png](https://recruit-mp.qiita.com/files/2104708c-d51a-1dff-667e-17413030c6d5.png)

以上のコマンドは実際のタスクのビルドが走ります。もし実際のタスクを走りたくない場合、 `--dry-run` を付けてできます。
今回は実際にどのぐらいの時間を削減できるのを実感したく、 `dry-run` をやりませんでした。
以下の手順となります。

- 1. プロジェクトcleanする
- 2. プロジェクトbuildする

## III. 現状

- プロジェクトは複数のモジュールを含む
- Android Studio バージョンが `2.2.2`
- Gradle バージョンが `2.14.1`

早速ですが、現在一つのビルドがどのぐらいかかるのかを見てみましょう。

```
./gradlew app:assembleDebug --profile
```

![original](https://raw.githubusercontent.com/wang1990r/images/master/gradle/optimize/original.png)

5分26秒かかりましたね。実感はこれよりかかった気がします＞＜

## IV. ステップ1

`--configure-on-demand` というオプションがありまして、主に必要だけなタスクをビルドする機能です。

現在はまだデフォルトではないため、手動でONにしてみました。

```
./gradlew app:assembleDebug --profile --configure-on-demand
```

![configure-on-demand](https://raw.githubusercontent.com/wang1990r/images/master/gradle/optimize/configure-on-demand.png)

結構20秒の差がありましたね！

- 参考: https://docs.gradle.org/current/userguide/multi_project_builds.html#sec:configuration_on_demand

## V. ステップ2

GradleがJVMで走るため、毎回ビルドして初期化すると、時間がかかります。なので、毎回初期化する時間を削減するため、デーモンプロセス(daemon)を使って、初期化は一回だけで済むことができます。このデーモン機能は `--daemon` というオプションです。

*このオプションは `Gradle 3.0` 以降デフォルトONになりました。*

二回目以降効きますので、こちらのスクリーンショットをスッキプします。

- 参考：https://docs.gradle.org/current/userguide/gradle_daemon.html

## VI. ステップ3

現在使っているGradleのバージョンは結構古いので、最新版の `3.4.1` にしてみました。（ `--daemon` がデフォルトです）

```
[gradle-wrapper.properties]

distributionUrl=https\://services.gradle.org/distributions/gradle-3.4.1-all.zip
```

変更後、↓を走りました。

```
./gradlew app:assembleDebug --profile --configure-on-demand --daemon
```

![gradle-version](https://raw.githubusercontent.com/wang1990r/images/master/gradle/optimize/gradle-version.png)

更に20秒削減できました！

## VII. ステップ4

ステップ2で紹介したデーモンプロセスが使用するメモリーを `1536m` から `2048m` に増やしてみました。
実は `3G` まで増やしたいのですが、PCのメモリがきつくて、やめました。
PCにより、最適なメモリ数値が変わりらしく、加えて、一定の値の以上に設定してもパーフォーマンスが変わらないらしいです。
今回の設定は `gradle.properties` で直接に指定しました。

```
org.gradle.jvmargs=-Xmx2048m
```

![heap-size](https://raw.githubusercontent.com/wang1990r/images/master/gradle/optimize/heap%20size.png)

3分以内に収まりました！

## VIII. ステップ５（最強兵器）

現状のプロジェクトですが、複数のモジュールが持っていて、モジュール同士の依頼関係はちゃんと `build.gradle` に声明していて、 `Decoupled Projects` と呼ばれるでしょう。それで、並列に走ることができます。

```
./gradlew app:assembleDebug --profile --configure-on-demand --daemon --parallel
```

![parallel](https://raw.githubusercontent.com/wang1990r/images/master/gradle/optimize/parallel.png)

タスクが並列に走るため、 `Task Execution` は全てのタスクの時間の総合で、 `Total Build Time` より長いと考えておりました。

結果、２分以内に収まりました！:tada::tada::tada:

- 参考：https://docs.gradle.org/current/userguide/multi_project_builds.html#sec:parallel_execution

## `gradle.properties` の設定

以上のコマンドラインについて説明しましたが、直接に `gradle.properties` を設定した方がオススメします！

```
org.gradle.daemon=true
org.gradle.configureondemand=true
org.gradle.parallel=true
org.gradle.jvmargs=-Xmx2048m
```

## 感想
一日14回をビルドすると、１時間の作業時間を減らすことができますね！
まさに、**最＆高！！！**

## 追記（もっと速くする）


- `run dex in process` はビルドを速くします。ONにする方法はヒープサイズを `2G` 以上設定することです。

```
org.gradle.jvmargs=-Xmx2048m
```

---------
- ↓を設定する場合、-Xmx には `javaMaxHeapSize + 512mb以上` 設定した方がいい

```
dexOptions {
    javaMaxHeapSize "2g"
}
```

  - 参照
    - https://android.googlesource.com/platform/tools/base/+/028ba07/build-system/builder/src/main/java/com/android/builder/core/DexByteCodeConverter.java#72
    - https://developer.android.com/studio/build/optimize-your-build.html#increase_gradle_heap

---------

- cleanを頻繁にしないなら `preDexLibraries=true` 、cleanするなら `preDexLibraries=false` 。なので、CIビルドはfalseにした方が速い。
ドキュメントの参照は↓

> preDexLibraries: Declares whether to pre-dex library dependencies so that your incremental builds are faster. Because this feature may slow down your clean builds, you may want to disable this feature for your continuous integration server.

  - 参照：https://developer.android.com/studio/build/optimize-your-build.html#dex_options

---------
- ライブラリを変更してないなら `--offline` オプションを使う　←　めっちゃ速くなる

![image](https://recruit-mp.qiita.com/files/aeb5e305-bbba-a0ff-d864-00d05f1ed7cb.png)

---------

- build cache を使うと、clean buildが速くなります。キャッシュフォルダーは `~/.android/build-cache` になります。

```
android.enableBuildCache=true
```

ただ、注意点がありまして、build cache を使うなら、preDexLibraries は未設定やtrueの状態でないといけません。

Android Studio 2.3 の場合、デフォルトが `true` になります。
Android plugin 2.3.0の場合、デフォルトが `true` になります。

  - 参照：http://tools.android.com/tech-docs/build-cache


## 質問

https://developer.android.com/studio/releases/gradle-plugin.html#revisions

- Android plugin for Gradle, revision 2.1.0 (April 2016)

incremental jvmargs

- Android plugin for Gradle, revision 2.0.0 (April 2016)
maxProcessCount


## More
- multiDexEnabled false minsdk:21
has pre-dex

- multiDexEnabled true minsdk:21
no pre-dex

为啥multidex花时间？
https://developer.android.com/studio/build/multidex.html#keep

```
When building each DEX file for a multidex app, the build tools perform complex decision-making to determine which classes are needed in the primary DEX file so that your app can start successfully. If any class that's required during startup is not provided in the primary DEX file, then your app crashes with the error java.lang.NoClassDefFoundError.

This shouldn't happen for code that's accessed directly from your app code because the build tools recognize those code paths, but it can happen when the code paths are less visible such as when a library you use has complex dependencies. For example, if the code uses introspection or invocation of Java methods from native code, then those classes might not be recognized as required in the primary DEX file.

So if you receive java.lang.NoClassDefFoundError, then you must manually specify these additional classes as required in the primary DEX file by declaring them with the multiDexKeepFile or the multiDexKeepProguard property in your build type. If a class is matched in either the multiDexKeepFile or the multiDexKeepProguard file, then that class is added to the primary DEX file.
```

为啥minsdkversion21以上变快了？
```
These settings cause the Android plugin for Gradle to do the following:

1. Perform pre-dexing: Build each app module and each dependency as a separate DEX file.
2. Include each DEX file in the APK without modification (no code shrinking).
3. Most importantly, the module DEX files are not combined, and so the long-running calculation to determine the contents of the primary DEX file is avoided.
```

Reference: https://developer.android.com/studio/build/multidex.html#dev-build

- multiDexEnabled true minsdk:16
empty pre-dex folder

- multiDexEnabled false minsdk:16
has pre-dex

- multiDexEnabled true minsdk:22
no pre-dex

