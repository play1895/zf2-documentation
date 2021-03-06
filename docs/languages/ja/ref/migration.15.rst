.. EN-Revision: none
.. _migration.15:

Zend Framework 1.5
==================

以前のバージョンから Zend Framework 1.5 またはそれ以降に更新する際は、
下記の移行上の注意点に注意すべきです。

.. _migration.15.zend.controller:

Zend_Controller
---------------

基本的な機能は同じでドキュメント化されている機能も変わりませんが、
ひとつだけ、 **ドキュメント化されていない**"機能" が変更されました。

*URL* の書き方としてドキュメント化されている方法は、 camelCased
形式の名前のアクションを使用するために
単語の区切り文字を使用するというものです。デフォルトの区切り文字は '.'
あるいは '-' ですが、ディスパッチャの設定で変更できます。
ディスパッチャは内部でアクション名を小文字に変換し、 単語の区切り文字をもとに
camelCasing 形式のアクションメソッド名を作成します。 しかし、 *PHP*
の関数名は大文字小文字を区別しないので、 *URL* 自体を camelCasing
形式で書くこともできます。 この場合でも、ディスパッチャは *URL*
を同じアクションメソッドに解決します。 たとえば 'camel-cased'
はディスパッチャによって 'camelCasedAction' になります。一方 'camelCased' は
'camelcasedAction' となります。 *PHP* では大文字小文字を細かく区別しないため、
これらはどちらも同じメソッドを実行することになります。

これは、ViewRenderer がビュースクリプトを解決する際に問題を引き起こします。
ドキュメントに記載されている正式な方法は、
単語の区切りをすべてダッシュに変換して単語は小文字にするというものです。
こうすればアクションとビュースクリプトの関連が明確になり、
小文字への正規化でスクリプトが見つかることが確実となります。
しかし、アクション 'camelCased' がコールされて解決された場合は、
単語の区切りはもう存在しません。そして ViewRenderer は ``camel-cased.phtml``
ではない別のファイル --``camelcased.phtml`` を探してしまうのです。

中にはこの "機能" を使用している開発者もいるようますが、
これは決して意図した機能ではありません。 1.5.0 のツリーでは、ViewRenderer
はこの方式の解決を行わなくなりました。
これでアクションとビュースクリプトの結びつきが確実になったわけです。
まず、ディスパッチャはアクション名の大文字小文字をきちんと区別するようになります。
つまり、camelCasing 形式を使用したアクションの解決先は、 単語の区切りを使用した
('camel-casing') 場合とは違うものになるということです。 これで、ViewRenderer
がビュースクリプトを解決する際には
区切り文字を使用したアクションのみを使用することになります。

今までこの "機能" に頼っていた人たちは、 以下のいずれかの方法で対応します。

- 一番いい方法: ビュースクリプトの名前を変更する。 利点: 前方互換性。欠点:
  もし対象となるビュースクリプトが多い場合は、
  多くのファイルの名前を変更しなければならなくなります。

- その次にいい方法: ViewRenderer はビュースクリプトの解決を ``Zend_Filter_Inflector``
  に委譲しています。 インフレクタのルールを変更し、
  アクションの単語間をダッシュで区切らないようにします。

  .. code-block:: php
     :linenos:

     $viewRenderer =
         Zend_Controller_Action_HelperBroker::getStaticHelper('viewRenderer');
     $inflector = $viewRenderer->getInflector();
     $inflector->setFilterRule(':action', array(
         new Zend_Filter_PregReplace(
             '#[^a-z0-9' . preg_quote(DIRECTORY_SEPARATOR, '#') . ']+#i',
             ''
         ),
         'StringToLower'
     ));

  上のコードは、インフレクタを変更して単語をダッシュで区切らないようにしています。
  もし実際のビュースクリプト名を camelCased にしたいのなら、さらに 'StringToLower'
  フィルタも削除することになるでしょう。

  ビュースクリプトの名前を変えるのが面倒だったり 時間がかかったりする場合は、
  もしあまり時間を割けないのならこの方法が最適です。

- あまりお勧めしない方法: ディスパッチャに camelCased
  形式のアクションをディスパッチさせるよう、フロントコントローラのフラグ
  ``useCaseSensitiveActions`` を設定します。

  .. code-block:: php
     :linenos:

     $front->setParam('useCaseSensitiveActions', true);

  これで camelCasing 形式の URL を使えるようになり、
  単語の区切り文字を使用した場合と同じアクションに解決されるようになります。
  しかし、もともと抱えていた問題も残ったままとなってしまいます。
  できれば先ほどのふたつのうちのいずれかを使用したほうがいいでしょう。

  このフラグを使用していると、 将来このフラグが廃止予定になったときに notice
  が発生することになります。


