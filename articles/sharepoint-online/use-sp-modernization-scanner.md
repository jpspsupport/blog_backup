---
title: SharePoint Modernization Scanner を使用して、移行対象のワークフローを抽出する
date: 2020-07-09
tags:
  - SharePoint Online
---

# SharePoint Modernization Scanner を使用して、移行対象のワークフローを抽出する

今回の投稿では、SharePoint Online における [SharePoint 2010 形式ワークフローの廃止](https://support.microsoft.com/ja-jp/office/1ca3fff8-9985-410a-85aa-8120f626965f?ui=ja-jp&rs=ja-jp&ad=jp) (MC217999) に際しての計画段階において、移行対象のワークフローを抽出する方法を詳細に記載します。

そのためのツールとして、SharePoint Modernization Scanner (SharePoint モダン化スキャナー) が公開されています。

最初に、移行する SharePoint 2010 形式ワークフローの全体像をつかむことは必須の作業となります。これをもとに、技術的な課題項目を整理したり、工数を見積もるという作業が派生してくると思います。
本投稿では、できる限りスムーズに移行検討作業に移れるよう、現バージョンの SharePoint モダン化スキャナーによる操作イメージを画面付きで紹介します。

## 1. 最初に下記サイトからツールをダウンロードします。

タイトル : SharePoint モダン化スキャナーを使い始める
アドレス : https://docs.microsoft.com/ja-jp/sharepoint/dev/transform/modernize-scanner

![DownloadSMS](use-sp-modernizationscanner/SMS01.png)

## 2. ダウンロードしたツールを実行します。

![RunSMS](use-sp-modernizationscanner/SMS02.png)

### 2-1. クライアント ID とクライアント シークレットを用意します。
Selected Authentication Option ですが、テナント全体に対して実行する場合は、あらかじめテナント内の全サイト コレクションにアクセス可能なユーザーが用意できない場合は、アプリ専用トークンを利用することが現実的です。
アプリ専用トークンの種類は 2 種類あります。一般的にはライフサイクルを考慮した場合、継続的なアプリケーションの開発や利用を想定した場合は Azure AD App Only が推奨されますが、どちらでも現時点での機能面に変わりはないため、今回の一時的な利用においては操作が簡単な Azure ACS App Only のシナリオを進めていただくことで問題はありません。

別途以下のサイトなどの手順で Azure ACS クライアント ID と クライアント シークレットを入手します。

タイトル : PnP PowerShell でテナントのフルコントロール権限を使用して接続する方法について
アドレス : https://jpspsupport.github.io/blog/sharepoint-online/pnp-powershell-with-app-only-token/

なお、上記サイトはテナントに対してフルコントロール権限でアプリを払い出しておりますので、適宜権限要求 XML を変更してサイトに対するフルコントロール等に変更していただいてもかまいません。

### 2-2. クライアント ID とクライアント シークレットを入力して [Next] をクリックします
### 2-3. 対象を指定して [Next] をクリックします。

![RunSMS2](use-sp-modernizationscanner/SMS03.png)

## 3. Scan mode には Classic workflow usage を指定ください。

![RunSMS3](use-sp-modernizationscanner/SMS04.png)

## 4.	最後に細かな設定が可能ですが、[Start Scan] を押すと開始します。

![RunSMS3](use-sp-modernizationscanner/SMS04-2.png)

## 5. 実行時にはコマンド プロンプトに進捗が書かれます。

![RunSMS4](use-sp-modernizationscanner/SMS05.png)

## 6. コマンド実行が完了すると、フォルダーが実行プログラム (*.EXE) と同一階層に現れます。

![FILES](use-sp-modernizationscanner/SMS06.png)

## 7. フォルダー内には以下のようなファイルが入っています。

![FILES1](use-sp-modernizationscanner/SMS07.png)

## 8. ModernizationWorkflowScanResults.csv を開きます。

日本語が含まれる場合は、一度ファイルをメモ帳で開いて、名前を付けて保存し、UTF8 BOM 付きで保存しなおし、その後 Excel で開きます。

![FILES2](use-sp-modernizationscanner/SMS08.png)

## 9. シート上のセルをテーブルに変換し、フィルターを使用して、置き換え対象のワークフローを絞り込んでいきます。

![EXCEL1](use-sp-modernizationscanner/SMS09.png)

例
* SharePoint 2010 形式ワークフロー (Version = 2010)
* リストで新規インスタンスを許可されている (Enabled = True) 

![EXCEL2](use-sp-modernizationscanner/SMS10.png)

この時点で、使用中のワークフロー一覧については抽出完了です。
サイト コレクション管理者にコンテンツを委任している場合は、各サイト コレクション管理者に MC217999 の内容や告示内容、各サイト コレクションごとの抽出データとあわせ、Admins 列に格納されたユーザーに共有して連携してください。


