---
title: .sbt ビルド定義
layout: default
---

[Keys]: http://www.scala-sbt.org/release/sxr/sbt/Keys.scala.html

# `.sbt` ビルド定義

[前](../running) _始める sbt 6/14 ページ_ [次](../scope)

このページでは、多少の「理論」も含めた sbt のビルド定義 (build definition) と `build.sbt` の構文を説明する。
君が、[sbt の使い方](../running)を分かっていて、「始める sbt」の前のページも読んだことを前提とする。

## `.sbt` vs. `.scala` 定義

sbt のビルド定義はベースディレクトリ内の `.sbt` で終わるファイルと、
`project` サブディレクトリ内の `.scala` で終わるファイルを含むことができる。

どちらか一つだけを使うこともできるし、両方使うこともできる。
普通の用途には `.sbt` を使って、`.scala` を使うのは以下のような `.sbt` で出来ないことに限定する、というのが良い方法だ:

 - sbt をカスタマイズする（新しい設定値やタスクを加える）
 - サブプロジェクトを定義する(sbt0.13.0以降はbuild.sbtでも可能)

このページでは `.sbt` ファイルの説明をする。`.scala` ファイルの詳細と、それがどう `.sbt` に絡んでくるかに関しては、
（このガイドの後ほどの）[.scala ビルド定義](../full-def)を参照。

## ビルド定義って何？

** ここは読んで下さい **

プロジェクトを調べ、全てのビルド定義ファイルを処理した後、sbt は、ビルドを記述した不可変マップ（キーと値のペア）を最終的に作る。

例えば、`name` というキーがあり、それは文字列の値、つまり君のプロジェクト名に関連付けられる。


_ビルド定義ファイルは直接には sbt のマップに影響を与えない。_

その代わり、ビルド定義は、型が `Setting[T]` のオブジェクトを含んだ巨大なリストを作る。
`T` はマップ内の値の型だ。（Scala の `Setting[T]` は Java の `Setting<T>` と同様。）
`Setting` は、新しいキーと値のペアや、既存の値への追加など、_マップの変換_を記述する。
（関数型プログラミングの精神に則り、変換は新しいマップを返し、古いマップは更新されない。）

`build.sbt` では、プロジェクト名の `Setting[String]` を以下のように作る:

    name := "hello"

この `Setting[String]` は `name` キーを追加（もしくは置換）して `"hello"` という値に設定することでマップを変換する。
変換されたマップは新しい sbt のマップとなる。

マップを作るために、sbt はまず、同じキーへの変更が一緒に起き、かつ他のキーに依存する値の処理が依存するキーの後にくるように `Setting` のリストをソートする。
次に、sbt はソートされた `Setting` のリストを順番にみていって、一つづつマップに適用する。

まとめ: _ビルド定義は `Setting[T]` のリストを定義し、`Setting[T]` は sbt のキー・値ペアへの変換を表し、`T` は値の型を指す。_

## `build.sbt` はどう設定値を定義するか

以下に具体例で説明しよう:

<pre>
name := "hello"

version := "1.0"

scalaVersion := "2.9.1"
</pre>

`build.sbt` は、空行で分けられた `Setting` のリストだ。それぞれの `Setting` は Scala の式で表される。

`build.sbt` 内の式は、それぞれ独立しており、完全な Scala 文ではなく、式だ。
そのため、`build.sbt` 内ではトップレベルでの `val`、`object`、クラスやメソッドの定義は禁止されている。
(sbt0.13.0以降、val、lazy val、メソッド定義は可能になった。objectやclass、varの定義は引き続き不可能)

左辺値の `name`、`version`、および `scalaVersion` は _キー_だ。
キーは `SettingKey[T]`、`TaskKey[T]`、もしくは `InputKey[T]` のインスタンスで、
`T` はその値の型だ。キーの種類に関しては後で説明しよう。

キーには `:=` メソッドがあり、`Setting[T]` を返す。
Java 的な構文でこのメソッドを呼び出すこともできる:

    name.:=("hello")

だけど、Scala では `name := "hello"` と書ける（Scala では全てのメソッドがどちらの構文でも書ける）。

`name` キーの `:=` メソッドは `Setting` を返すが、特に `Setting[String]` を返す。
`String` は、`name` の型にもあらわれ、これは、`SettingKey[String]` となっている。
この場合、返された `Setting[String]` は、キーを追加（もしくは置換）して `"hello"` という値に設定するマップの変換だ。

間違った型の値を使うと、ビルド定義はコンパイルしない:

    name := 42  // コンパイルしない

## キーは Keys オブジェクトで定義される

組み込みのキーは [Keys] と呼ばれるオブジェクトのフィールドにすぎない。
`build.sbt` は、自動的に `import sbt.Keys._` するため、
`sbt.Keys.name` は `name` として呼ぶことができる。

カスタムのキーは [.scala ファイル](../full-def)か[plugin](../using-plugins) で定義することができる。

## 設定を変換する他の方法

`:=` による置換は、最も単純な変換だけど、他にもいくつかある。
例えば、`+=` を用いて、リスト値に追加することができる。

他の変換は[スコープ](../scope)の理解が必要なため、
[次のページ](../scope)がスコープで、
[次の次のページ](../more-about-settings)で設定の詳細に関して説明する。

## タスクキー

キーには三種類ある:

 - `SettingKey[T]`: 値が一度だけ計算されるキー（値はプロジェクトの読み込み時に計算され、保存される）。
 - `TaskKey[T]`: 毎回再計算され、副作用を伴う可能性のある値のキー。
 - `InputKey[T]`: コマンドラインの引数を受け取るタスクキー。
 　「初めての sbt」では `InputKey` を説明しないので、このガイドを終えた後で、
   [[Input Tasks]] を読んでみよう。

`TaskKey[T]` は、_タスク_を定義しているといわれる。タスクは、`compile` や `packae` のような作業だ。
タスクは `Unit` を返すかもしれないし（`Unit` は、Scala での `void` だ）、
タスクに関連した値を返すかもしれない。例えば、`package` は作成した jar ファイルを値として返す `TaskKey[File]` だ。

例えばインタラクティブモードの sbt プロンプトに `compile` と打ち込むなどして、
タスクを実行するたびに、sbt は関連したタスクを一回だけ再実行する。

プロジェクトを記述した sbt のマップは、`name` のようなセッティング (setting) ならば、その文字列の値をキャッシュすることができるけど、
`compile` のようなタスク（task）の場合は実行可能コードを保存する必要がある
（たとえその実行可能コードが最終的に同じ文字列を返したとしても、それは毎回再実行されなければいけない）。

_あるキーがあるとき、それは常にタスクか素のセッティングかのどちらかを参照する。_
つまり、キーの「タスク性」（毎回再実行するかどうか）はキーの特性であり、値にはよらない。

`:=` を使うことで、タスクに任意の演算を代入することができ、その演算は毎回再実行される:

    hello := { println("Hello!") }

型システムの視点から考えると、タスクキー (task key) から作られた `Setting` は、セッティングキー (setting key) から作られたそれとは少し異なるものだ。
`taskKey := 42` は `Setting[Task[T]]` の戻り値を返すが、`settingKey := 42` は `Setting[T]` の戻り値を返す。
タスクが実行されるとタスクキーは型`T` の値を返すため、ほとんどの用途において、これによる影響は特にない。

`T` と `Task[T]` の型の違いによる影響が一つある。
それは、セッティングキーはキャッシュされていて、再実行されないため、タスキキーに依存できないということだ。
このことについては、後ほどの[他の種類のセッティング](../more-about-settings)にて詳しくみていく。

## sbt インタラクティブモードにおけるキー

sbt のインタラクティブモードからタスクの名前を打ち込むことで、どのタスクでも実行することができる。
それが `compile` と打ち込むことでコンパイルタスクが起動する仕組みだ。つまり、`compile` はタスクキーなのだ。

タスクキーのかわりにセッティングキーの名前を入力すると、セッティングキーの値が表示される。
タスクキーの名前を入力すると、タスクを実行するが、その戻り値は表示されないため、
タスクの戻り値を表示するには素の `<タスク名>` ではなく、`show <タスク名>` と入力する。

Scala の慣例にのっとり、ビルド定義ファイル内ではキーはキャメルケース（`camelCase`）で命名されているけども、
sbt コマンドラインではハイフン分けされて（`hyphen-separated-words`）命名されている。
sbt で使われているハイフン分けされた文字列はキーの定義とともに宣言されている（[Keys] 参照）。
例えば、`Keys.scala` に以下のキーがある:

    val scalacOptions = TaskKey[Seq[String]]("scalac-options", "Options for the Scala compiler.")

sbt では `scalac-options` と打ち込むけど、ビルド定義ファイルでは `scalacOptions` を使う。

あるキーについてより詳しい情報を得るためには、sbt インタラクティブモードで `inspect <キー名>` と打ち込む。
`inspect` が表示する情報の中にはまだ分からないこともあると思うけど、一番上にはセッティングの値の型と、セッテイングの簡単な説明がある。

## `build.sbt` 内の import 文

`build.sbt` の一番上に import 文を書くことができ、それらは空行でわけなくてもいい。

自動的に以下のものがデフォルトでインポートされる:

<pre>
import sbt._
import Process._
import Keys._
</pre>

（さらに、[.scala ファイル](../full-def)がある場合は、それらの全ての `Build` と `Plugin` の内容もインポートされる。
これに関しては、[.scala ビルド定義](../full-def)でさらに詳しく。）

## ライブラリへの依存性を加える

サードパーティのライブラリに依存するには二つの方法がある。
第一は `lib/` に jar ファイルを入れてしまう方法で（アンマネージ依存性、unmanged dependency）、
第二はマネージ依存性（managed dependency）を加えることで、`build.sbt` ではこのようになる:

    libraryDependencies += "org.apache.derby" % "derby" % "10.4.1.3"

これで Apache Derby ライブラリのバージョン 10.4.1.3 へのマネージ依存性を加えることができた。

`libraryDependencies` キーは二つの複雑な点がある:
`:=` ではなく `+=` を使うことと、`%` メソッドだ。
後で[他の種類のセッティング](../more-about-settings)で説明するけど、`+=` はキーの古い値を上書きするかわりに新しい値を追加する。
`%` メソッドは文字列から Ivy モジュール ID を構築するのに使われ、これは[ライブラリ依存性](../library-dependencies)で説明する。

ライブラリ依存性に関しては、このガイドの後ほどまで少しおいておくことにする。
後で、[一ページ分](../library-dependencies)をさいてちゃんと説明する。

## 続いては

[スコープ](../scope)について。
