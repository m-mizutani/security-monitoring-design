# セキュリティ監視の設計

**目次**

- なぜセキュリティ「監視」が必要なのか
- セキュリティ監視に必要な3つの機能
    - セキュリティアラートが検出できる
    - アラートに関する調査ができる
    - アラートの管理ができる
- 設計で考えるべき4つのポイント
    - どのログを集めるか
    - どのようにログを検索するか
    - どのようにアラートを検出するか
    - どのようにアラートを管理するか

## なぜセキュリティ「監視」が必要なのか

- 「セキュリティ」という分野は破茶滅茶に広いが、この文書ではサイバーセキュリティ（主にコンピュータのソフトウェア、ネットワークを中心としたセキュリティ）について述べる
- セキュリティ対策は大別すると「防止」と「検出・対応」になる
    - 定義は規格や人によってブレるがそれぞれが大きくずれているわけではない
    - 防止：予防・抑止に分かれる
        - 予防：あらかじめ脆弱性をなる部分を潰しておいて攻撃されても被害がでないようにする対策
        - 抑止：攻撃者を牽制して攻撃をさせないようにする対策。サイバーセキュリティだと個人や一組織でできる対策はあまり多くない
    - 検出と対応（あるいは復旧）
        - 検出：セキュリティに影響があると思われる事象を見つけるための対策
        - 対応：セキュリティ上影響を及ぼす事象が確定した場合に影響範囲を把握し、被害を極小化する。さらには事後報告や法執行機関
        - 「検出・対応」の対策はそれぞれが密接に絡み合っているものが多い。
        - また、検出と対応ははっきりとした区切りがあるわけではなく、現実には徐々に対応へと移行していくイメージになる
            - セキュリティ侵害の可能性がある事象が検出されても、それが誤検知であったり、実際には影響のない事象の可能性もあるので調査が必要
            - この調査は一部のセキュリティ侵害が確定しても、影響範囲などを改めて把握するために継続することになる
        - よってここで説明する「監視」は検出を中心とするが、一部対応の話も含む
- セキュリティ侵害を事前に防ぐ「防止」は高めるほどトレードオフが厳しくなるため検出も必要
    - 「防止」の方が安心感はあるが、現実には防止だけに頼ると運用のコストが無視できなくなってくる
    - 最も導入が簡単な防止（予防）策のは「できることを制限する」こと
        - アクセスできるリソースの種類を制限する
        - アクセスできるネットワークを限定する
        - アクセスできる人を制限する
        - チェックを厳しくする
    - これは導入は楽だが運用のコストは少なくない
        - はじめる分には「やるぞ！」と言うだけ
        - 実際には完全な定型業務でない限り不便が発生しがち。利便性の低下
            - 事業の視点で言えばビジネスのスピードを低下させる
        - 制限できるものを管理するためのコストも考える必要がある
            - 対象となるリソースや人、組織は常に入れ替わるのでルールなどを継続的に見直さないといけない
            - strictなルールを運用すればするほど複雑化して大変なだけではなくミスも発生しがち
    - 検出を絡めたほうがトータルのコストは安くなりがち
        - 問題が発生したときにそれを発見し、速やかに対応に繋げられる体制を整えておく
    - そのためセキュリティ対策は防止と検出を両輪で考え、実装できるとバランスが良くなりやすい
- 詳しくはBruce Schneier先生の[「セキュリティはなぜやぶられたのか」](https://www.amazon.co.jp/dp/4822283100)を読んでくれよな



## セキュリティ監視に必要な3つの機能

- 用語定義
    - `ログ`: セキュリティに関連する事象（イベント）を記録したデータ全般
        - これ自身を「イベント」と呼ぶこともある
        - 基本的には1つの事象に対する記録が1つのログ、ということにしておきたい
        - 一般化して説明しようとするとかなり難しいというか抽象化されすぎてしまうので、ここではあまり突き詰めない
    - `アラート`: セキュリティ侵害が発生している可能性がある、とラベル付けされたログ（複数の場合もある）
        - 誤検知や影響のないイベントを検出したものも含む
        - 検出方法（ラベルの付け方）については後述
        - 同じ概念だが "Incident", "Detection", "Finding", "Event of Interest", "Offense" など、製品や規格によって呼び方はバラバラ
        - この文書では「アラート」で統一する
    - `セキュリティ監視`: ログを集め、その中からアラートを見つけ、そのアラートは現実の被害や影響がある事象を捉えていたのかを調査し、もし影響があったら対応に必要な情報を収集する、という業務
- ほぼ定義のままだが、セキュリティ監視に必要な要件はアラートの検出・調査・管理

### アラートが検出できる

- アラートの検出方法は概ね以下の4通りある
    - 監視装置からあがってくるアラート
        - セキュリティ監視のためのデバイス、あるいはその機能を併せ持つデバイスが「アラートです」と言って送ってくるアラート
        - NGFW (Next Generation Firewall)、IDS (Intrusion Detection System)、WAF (Web Application Firewall) アンチウィルスソフトなど、そもそもアラートを見つける機能を持っているデバイス（もちろんソフトウェアベースのものも含む）
        - 監視デバイス側で判定してくれるので楽だが、ロジックの調整はする必要がある
            - 過剰にアラートを検出している場合はアラートを集約した先でも調整できる
            - 感度が低いという場合はデバイス側を調整するしかない
    - メトリクス型のアラート
        - 特定の種類のログが一定時間内にN回発生したら発火するタイプのアラート
        - 例：「ログイン失敗が10分以内に10回発生するのを検知」
        - 監視デバイス側で同等のことをやってくれる場合と、ログを集約した先でそういったロジックを組む場合がある
        - インフラ監視のプロダクト・ツールで実現できる場合もある（nagios、zabbixなど）
            - ただし、回数カウントのためのキーが爆発的に多い場合があるので注意が必要
            - 例えばアクセスしてくるIPアドレスがキーだと1,000〜1,000,000というようなオーダーになる
        - 事前にしきい値を決めておく
    - 相関ルールによるアラート
        - 複数のログを組み合わせて発火するアラート
        - 例「ポートスキャンの発生を示すログのあとにExploitコードの送信を示すログが出現したら攻撃と判定する」みたいなやつ
        - SIEM (Security Informaion & Event Manager) が得意とする機能（詳しくは後述）
        - 単体のログでは危険性が認識できない場合でも複数組み合わせることでリスクのあるアラートとして検出できる（かもしれない）
        - ただし、ログとログの組み合わせをルールとして管理するのはとても大変
            - 特に異なる監視装置・製品・サービス間のログをつなげるのは大変
            - ログの種類を抽象化し、汎用的なルールだけ管理するというアプローチをしている某SIEM製品があるが、ログの抽象化が残念で厳しい様子だった
        - ちゃんとやるならしっかり自組織内でルールを管理・運用していく心構えが必要
    - アナリスト（人間）がログから検出するアラート
        - アラートとは直接関係ないログ（たいだい大量）からアナリストが知識・経験・直感をもとに見つけ出したアラート
        - 現実にはまだシステムに取り込まれていないようなIoC (Indicator of Compromise) 情報をログから検索したりした結果の場合が多い
        - でもたまに本気で人間が（一部のカテゴリだけだったりするけど）目視でログを見て分析しているサービスもあったりする
        - IoCを使うものはともかく人間がログを日々見るのは結構大変（神経が削れる）なので、定常運用はあまりオススメできない
    - 一応、他にも異常検知のカテゴリはあるが、…まあ頑張りたい人は頑張ってみてくれ、という気持ち
        - アラートに直接関係しないようなログはセキュリティ以外の要素に影響されやすい
            - 利用している・提供しているサービスそのものの異常ないしトラブル
            - 外的要因。極端な例で言えばYahoo砲で、内部の状況に関係なくいつもと挙動が変わったりするケース
- これらのアラートの検出を管理・調整できる運用が必要
    - アラートの検出結果の善し悪しは必ず組織によって異なるので画一的な検出ルールで運用するのは非常に厳しい
        - 事業会社ならやっているビジネスが違う
        - 組織ごとの温度感の違い・体制
        - 「防止」の対策をどれくらい＆どのように導入・運用しているのかにも影響してくる
    - なので導入前だけでなく継続的にアラート検出のロジックを修正してく必要がある
        - どちらかというと最終的にアラートとして管理されるフェイズに届く前にフィルタするような仕組みのほうが便利
            - ルールを一箇所にまとめて管理できるから
            - 監視装置毎の条件の設定方法などを（あまり）覚える必要がなく手間が省けるから
            - ただしどちらにしてもログに含まれるデータ構造などは把握してく必要がある
    - さらに出力も統一されているのが望ましい
        - 各監視装置内でアラートを管理する機能を持っているものもあるが、そこで管理しようとすると監視装置の数だけ操作やデータのあり方を覚える必要が出てくる
            - 教育などの負担増
            - いちいち監視装置にログインし直したりするのはだいぶ手間
        - また、アラートの情報を横断的に見れなくなる
        - 統一された形式で管理できるのであれば、監視装置などの特性について把握しておく必要はあるものの、だいぶ負担が減る


### アラートに関する調査ができる

### アラートの管理ができる

### 具体例：SOC (Security Operation Center)

## 設計で考えるべき4つのポイント

### どのログを集めるか
### どのようにログを検索するか
### どのようにアラートを検出するか
### どのようにアラートを管理するか

## 本文章について

### 著者

- Masayoshi Mizutani < mizutani@sfc.wide.ad.jp >, [@m_mizutani](https://twitter.com/m_mizutani)

### ライセンス

- [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.en)
