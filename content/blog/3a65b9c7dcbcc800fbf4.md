---
date: 2020-02-20T15:08:12.174Z
path: /3a65b9c7dcbcc800fbf4.md/index.html
template: BlogPost
title: スタンドアローンGORMをGroovyスクリプトからシュっと利用する
tags: Grails GORM Groovy SpringBoot
author: uehaj
slide: false
---
G* Advent Calendar 2017 20日目の記事です。

Grailsの[O/Rマッピングライブラリである「GORM」](http://gorm.grails.org/latest/)は、Grailsから切りはなして単体で使用することもできます。これをスタンドアローンGORMと呼びます。

非WebアプリやGrails以外の他のフレームワークから高機能なORMとして使うことも可能ですし、Grailsでのシステム開発の一部として、バッチやシェルスクリプトやsshでキックしたいとか、シェルアカウントの管理者権限者だけがデータベースにアクセスしたいときには、スタンドアローンGORMが有用なケースがあります。

というのも、そういうケースでは以下のような選択肢があるにはあるのですが、いずれも固有の課題を抱えているからです。

1. [grails scripts](http://docs.grails.org/latest/guide/commandLine.html#creatingCustomScripts)としてGrailsサービスやGrailsドメインクラスを呼び出す処理などを記述し、Grailsのプロジェクトフォルダ(GRAILS_PROJ/scriptsを含む)に格納して、grailsコマンドで起動。
問題は、プロジェクトソースコードを運用環境に展開しておく必要があること。
2. DB処理をWeb APIとして公開して、それを叩くクライアントを開発する。
転送速度がネックになったり、ポートを公開することでセキュリティの問題になるかもしれない。
3. Service層をGrailsの[Remoting Plugin](https://grails.org/plugin/remoting)を使って公開する。
問題は同上。
4. SQLを叩く別プログラムを開発する(Groovy SQL, ..)。
この場合の問題は、GrailsのドメインクラスからDDL生成されたデータベースに対して、生SQLを書くのはかったるいだけではなく、ドメインクラスの制約(constraint)の処理を別途しなければならないということ。

それぞれのケースで示した問題が許容できないとき、スタンドアローンGORMが有用です。

さて、具体的な使用方法としては、Spring Bootのアプリケーションから呼び出すこともできるでしょう。
本記事では、もっとストレートに素のGroovyスクリプトからGORMを呼び出す例を紹介します。

# 方法

たとえば、こんな感じでGroovyスクリプトを書いて、

```groovy
@Grab('org.grails:grails-datastore-gorm-hibernate5:6.1.0.RELEASE')
@GrabExclude('javax.transaction:jta')
@Grab('com.h2database:h2:1.4.192')
@Grab('org.apache.tomcat:tomcat-jdbc:8.5.0')
@Grab('org.slf4j:slf4j-log4j12')
import java.lang.String
import org.grails.orm.hibernate.HibernateDatastore
import grails.gorm.annotation.Entity

@Entity
class Book {
    String title
    static hasMany = [authors: Author]
    String toString() {"$id: $title"}
}

@Entity
class Author {
    String name
    static hasMany = [books: Book]
    static belongsTo = Book
    String toString() {"$id: $name"}
}

def hibernateDatastore = new HibernateDatastore([
        'hibernate.hbm2ddl.auto': 'create-drop',
        'dataSource.driverClassName': 'org.h2.Driver',
        'dataSource.url': 'jdbc:h2:mem:devDb;MVCC=TRUE;LOCK_TIMEOUT=10000;DB_CLOSE_ON_EXIT=FALSE',
        'dataSource.username': 'sa',
        'dataSource.password': '',
    ],
    Author)

Author.withTransaction {
    Author a = new Author(name:'上原 潤二');
    Book b1 = new Book(title: 'プログラミングGROOVY').addToAuthors(a).save()
    Book b2 = new Book(title: 'Grails徹底入門').addToAuthors(a).save()
    println Book.findByTitleLike('G%')
}
```

GroovyおよびJDKがインストールされていれば、以下で実行できます。

```
> groovy groovyスクリプトファイル名
```

GroovyやJDK以外の何かを事前ダウンロードしたりインストールしたりフォルダ構成を考えたりコンパイルしたりビルドしたりビルドスクリプトを書いたりデプロイしたりパッケージングしたりが一切不要なのが利点です✌。

なお、上記では@Entityでドメインクラスを定義していますが、データベースがGrailsアプリケーションの一部であるなら、Grailsアプリケーション本体のwar(やjar)に含まれるドメインクラスのクラスファイルのjarをクラスパスに含めれば別に書かなくても利用可能です(多分)。

