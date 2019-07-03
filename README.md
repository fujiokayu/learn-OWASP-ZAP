# Tutorial of OWASP ZAP

According to [第74回 脆弱性診断ええんやで(^^) for ルーキーズ](https://security-testing.connpass.com/event/134830/)

## Prerequisites

#### Download and Install [OWASP ZAP](https://github.com/zaproxy/zaproxy/wiki/Downloads)

## OWASP ZAP、ブラウザーの設定

- ZAP の運用では必ずプロテクトモードに設定する
    - 標準モードではあらゆる URL に対して攻撃が可能となるが、プロテクトモードでは自分が設定した診断対象にのみ攻撃、調査が可能となる

- ローカルプロキシの設定
    - [ツール][オプション][ローカル・プロキシ]
        - localhost という設定値は Typo しがちなのと、視認性が悪いので、127.0.0.1 を設定することを推奨
        - Port の 8080 は競合しがちなので避ける。あらゆる OS において予約されていない 49152-61000 を使用することを推奨
    - ZAP が動かないって時は疑うポイントでもある

- スパイダー
    - [ツール][オプション][スパイダー]
        - クロールする最大の深さはデフォルトで 5 となっており、取りこぼしが発生する可能性がある。10階層程度が推奨
        - 並列スキャンスレッド数は使用する端末のスペックや Web サーバー側の設定に依存することに注意。無闇にあげても効果が出ないことが多いので、１から初めて様子を見ながら増やしていくのが良い
        - 性能が上がって負荷がかからない範囲で作業するのが良い

- 動的スキャン
    - [ツール][オプション][動的スキャン]
        - 並列スキャンするホスト数について１にしておくのが無難
        - 並列スキャンスレッド数はスパイダーと同様
        - Max Result to list はデフォルトの1000だと足りないので、10000などに増やしておいた方が良い
            - ただしその分データベースの容量が大きくなるので、無闇に増やさない方が良い
            - 10000 を超えるようだと診断単位が不適切（より絞るべき）という懸念がある
        - スキャン中のミリ秒単位の遅延は、2.8から自由に設定できるようになった。スパンは長いほうが負荷が溜まりにくいが、診断時間とのトレードオフになる
            - 推奨は 1000 以上。自分のサーバーなど週末にゆっくり診断する場合は、10000 などで数日間起動したりしても良い

- HUD の切り方
    - [ツール][オプション][HUD][Enable whenEnable when using ZAP Enable when using the ZAP Desctop]のチェックをオフにする

- アドオン
    - [ヘルプ][アップデートのチェック]
    - [マーケットプレイス]タブでデフォルトでインストールされていない様々なツールがある

- ポリシー
    - [ポリシー][スキャンポリシー][追加]ボタンをクリック
    - ポリシーはエクスポートして保存できる。エクスポートした際、ポリシー名はそのままファイル名になる
    - [既定のアラートのしきい値]は「低」が推奨。この方がアラートが多く出る
    - [既定の攻撃の強度]も「低」が推奨。高くするとシステムへの負荷が高くなり、他のテストができなくなったりシステムの破壊が発生する恐れがあるため
        - 実際にこれを強にして実行したところ、データベースが空になったなどの話が出ている
    - [しきい値]を低にし、[ルールの開始]をクリックして適用する
    - [Strength to]を低にし、[ルールの開始]をクリックして適用する
    - 左側のツリーから、細かな診断種別ごとに設定することもできる

### ブラウザの設定
- 推奨は Firefox  
Chrome だと証明書やプロキシは OS の設定を見ているので、それらを変更しなければいけない。  
Firefox の設定は OS から独立しており、OS 側の環境を汚さずに作業ができるため、脆弱性診断ではよく使われる。

- [設定][ネットワーク設定][接続設定]
    - [手動でプロキシを設定する]をオンにし、ZAP に設定した IP とポートを設定する。
    - すべてのプロトコルでこのプロキシーを使用するにチェック
    - プロキシーなしで接続の欄は空白にしておく

- 上記の設定を行うと証明書のエラーでサイトが閲覧できなくなった。おそらく 2.8 で追加された HUD が影響しているのでは。 
これを回避するために ZAP の証明書を Firefox にインポートする
    1. ZAP で[ツール][オプション][ダイナミック SSL証明書] からルート CA 証明書を保存
    1. Firefox には以下の設定で追加する
    1. 設定→プライバシーとセキュリティ→証明書を表示
    1. 読み込むから、先ほどの OWASP の CA を選択し、「この認証局によるウェブサイトの識別を信頼する」をチェック

- Chrome の場合は以下の手順で OS のネットワーク設定を変更する。（Mac OSX での手順）  
    1. [設定][詳細設定][プロキシ設定を開く]→[ネットワーク設定]の[プロキシ]タブを表示する
    1. Web プロキシで IP/Port を設定する
    1. プロキシ設定を使用しないホストとドメインは空欄にする
    1. [保護されたWebプロキシ]の方も同じように設定する

## 診断演習

### 対象アプリの情報収集

- 調査対象は owaspbwa にある OWASP Mutillidae II を練習に使う。これを診断用のブラウザで開く
    - ZAP のサイトツリーに Mutillidae が追加されるので、これを右クリックして[コンテキストに含める][New Context]。これでプロテクトモードでもこのサイトを調査できるようになる
    - 対象サイトを右クリックし、[攻撃][スパイダー]を開き、[スキャンを開始]
- [スパイダー]タブに表示される結果を確認していく
    - 履歴が狭くて見辛いので、[スパイダー]タブをダブルクリックして広げると見やすい
    - Processed 列が赤くなっている場所はその理由が記載されている
    - 基本的に、ここで緑になっているものが診断対象になる

### 自動診断

- OWASP Mutillidae II の左メニューから OWASP 2013 -> OWASP 2013 -> A3 -> Reflected -> DNS Lookup から、適当に localhost などの Lookup をしておく
- 履歴に Lookup している POST リクエストが表示されるので、これを右クリックして[攻撃]を選択
    - 左上のターゲットマークをクリックすると履歴が診断対象に絞られる
- ポリシーがデフォルトのものになっているので、自分が設定したポリシーに変更する
- スキャン後、[アラート]タブを確認する
