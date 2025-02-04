# Trickle ICE: Incremental Provisioning of Candidates for the Interactive Connectivity Establishment (ICE) Protocol

draft-ietf-ice-trickle-21

この文書は[draft-ietf-ice-trickle-21](https://tools.ietf.org/html/draft-ietf-ice-trickle-21)の非公式な日本語訳です。この翻訳は参考のために提供しており、その品質に責任を負いません。正確な情報を得るためには、原文を参照してください。

---

## 抄録

この文書では、ICEエージェントがまだ候補を集めている間に、一度にすべての候補を交換するのではなく、時間をかけて候補をインクリメンタルに交換することで、接続性チェックを開始することを可能にする、対話型接続確立(ICE)プロトコルの拡張である「Trickle ICE」について説明する。 この方法により、通信セッションの確立プロセスを大幅に高速化することができる。

## このメモの状況

このインターネットドラフトは、以下の内容に完全に準拠して提出されます。BCP 78およびBCP 79の規定を参照のこと。

インターネットドラフトは、インターネット工学の作業文書である。タスクフォース（IETF）。 他のグループが配布している場合もあることに注意してください。作業文書をインターネットドラフトとして提供しています。 現在のインターネットドラフトは http://datatracker.ietf.org/drafts/current/ にあります。

インターネットドラフトは、最大6ヶ月間有効なドラフトであり、いつでも他の文書に更新されたり、置き換えられたり、陳腐化したりする可能性があります。 インターネットドラフトを参考資料として使用したり、「進行中の作業」以外で引用したりすることは不適切です。

このインターネットドラフトは2018年10月17日に有効期限が切れる。

## 著作権について

Copyright (c) 2018 IETF Trust and the persons identified as the document authors.  All rights reserved.

本文書は、本文書の発行日に有効なBCP 78およびIETF信託のIETF文書に関する法的規定(http://trustee.ietf.org/license-info) の対象となります。 これらの文書には、この文書に関するあなたの権利と制限事項が記述されているので、注意深く検討してください。 この文書から抽出されたコードコンポーネントは、トラストの法的規定の第4.e項に記載されている簡易BSDライセンスのテキストを含まなければならず、簡易BSDライセンスに記載されているように保証なしで提供されています。

## 1.  序章

インタラクティブ接続確立(ICE)プロトコル[rfc5245bis]は、ICEエージェントがどのようにして候補を集め、相手のICEエージェントと候補を交換し、候補ペアを作成するかを記述しています。 ペアが集められると、ICEエージェントは接続性チェックを実行し、最終的には通信セッション内でデータを送受信するために使用されるペアを指名して選択します。

rfc5245bis]の手順に従うと、通信セッションの確立時間が多少長くなることがある。なぜなら、候補の収集には、STUNサーバー[RFC5389]への問い合わせと、TURNサーバー[RFC5766]を使用した中継された候補の割り当てが含まれることが多いからである。 多くのICE手順は並行して完了することができるが、[rfc5245bis]のペーシング要件に従う必要がある。

このドキュメントでは、"Trickle ICE" を定義しています。これはICE操作の補助的なモードで、候補が利用可能になるとすぐに(他の候補の収集と同時に)インクリメンタルに候補を交換することができます。 また、候補ペアが作成されるとすぐに接続性チェックを開始することができます。 Trickle ICEでは、候補の収集と接続性チェックを並行して行うことができるため、通信セッションの確立プロセスを大幅に高速化することができる。

このドキュメントでは、Trickle ICEのサポートを発見する方法、Trickle ICEを使用する際に[rfc5245bis]の手順がどのように変更または補足されるか、Trickle ICEエージェントが[rfc5245bis]に準拠したICEエージェントとどのように相互運用できるかについても定義している。

このドキュメントは、Trickle ICEのプロトコル固有の使用法を定義していない。 代わりに、Trickle ICEのプロトコル固有の詳細は、別の使用法文書で定義されている。 そのようなドキュメントの例として、[I-D.ietf-mmusic-trickle-ice-sip] (セッション開始プロトコル(SIP) [RFC3261]とセッション記述プロトコル(Session Description Protocol) [RFC3261]での使用法を定義している)と[XEP-0176] (XMPP [RFC6120]での使用法を定義している)がある。 ただし、このドキュメントのいくつかの例では、SDPとオファー/アンサーモデル(offer/answer model [RFC3264])を使用して、基本的な概念を説明している。

次の図は、オファー/アンサーモデルに従うプロトコルを使用したTrickle ICE交換の成功例である。

```

           Alice                                            Bob
             |                     Offer                     |
             |---------------------------------------------->|
             |            Additional Candidates              |
             |---------------------------------------------->|
             |                     Answer                    |
             |<----------------------------------------------|
             |            Additional Candidates              |
             |<----------------------------------------------|
             | Additional Candidates and Connectivity Checks |
             |<--------------------------------------------->|
             |<========== CONNECTION ESTABLISHED ===========>|
```

図1：フロー

Trickle ICE の背後にある技術については、2005年(XMPP Jingle 拡張モジュールが [XEP-0176] で指定された "ドリブルモード" を定義したとき)までさかのぼって、多くの運用経験があります。

1. Trickle ICEのサポートの決定
2. 初期ICE記述の生成
3. 初期ICE記述の処理と初期ICE応答の生成
4. 初期ICE応答の処理
5. チェックリストの作成、候補の剪定、接続性チェックなど。
6. 最初のICE記述と対応後の候補の収集と伝達
7. インバウンドで引っかかった候補者への対応
8. End-of-Candidates表示の生成と処理
9. ICEの再起動の処理

Trickle ICE の背後にある技術については、2005年(XMPP Jingle 拡張モジュールが [XEP-0176] で指定された "ドリブルモード" を定義したとき)までさかのぼって、多くの運用経験があります。

## 2.  用語解説

本文書中のキーワード「MUST」、「MUST NOT」、「REQUIRED」、「SHALL」、「SHALL NOT」、「SHOULD」、「SHOULD NOT」、「RECOMMENDED」、「MAY」、「OPTIONAL」は、[RFC2119]に記述されているように解釈されるべきである。

この仕様では、[rfc5245bis]で定義されているインタラクティブ接続確立のためのすべての用語を使用する。 さらに、以下の用語を定義している。

Full Trickle: Trickle ICEエージェントの典型的な動作モードで、最初のICE記述は任意の数の候補(ゼロ候補であっても)を含むことができ、Half Trickleのように候補の完全な世代を含む必要はない。

Generation: 1つのICEセッション内で伝達されるすべての候補。

Half Trickle: Trickle ICE の動作モードで、イニシエータが最初の ICE 記述を作成して伝達する前に、完全な世代の候補を厳密に収集する。 一度伝達されると、この候補情報は通常のICEエージェントによって処理され、Trickle ICEのサポートを必要としない。 また、Trickle ICEが可能なレスポンダは、まだ候補を収集し、ノンブロッキングの方法で接続性チェックを実行することができ、Trickle ICEの利点のおよそ「半分」を提供することができます。 このHalf Trickleメカニズムは、最初の ICE 説明を伝える前に、レスポンダが Trickle ICE をサポートしているかどうかを確認できない場合に使用することを目的としています。

ICE Description: ICE エージェントを設定するために必要な ICE セッション (候補ではない) に関連する属性。 これらには、ユーザ名フラグメント、パスワード、その他の属性が含まれるが、これらに限定されない。

Trickled Candidates: Trickle ICEエージェントが、最初のICE記述を伝達した後、または最初のICE記述に応答した後に、同じICEセッション内で伝達する候補。   Trickled候補は、候補の収集や接続性チェックと並行して伝えることができる。

Trickling: Trickled Candidatesを段階的に伝達すること。

Empty Check List: 最初は候補ペアが含まれていないチェックリストで、Tricklingされていくうちに候補ペアが追加されていくため。 (エージェントがチェックリストセットを作成するときに、すべての候補ペアが既知であるため、このシナリオは通常のICEエージェントでは発生しません)。

## 3.  Trickle ICEサポートの決定

Trickle ICEを完全にサポートするために、プロトコルを使用して、実装がTrickle ICEがサポートされているかどうかを判断できるように、以下のメカニズムのいずれかを組み込むべきである[SHOULD]。

1. セッションを開始する前にエージェントがTrickle ICEのサポートを検証できるように、能力発見の方法を提供する(XMPPの Service Discovery [XEP-0030]はそのようなメカニズムの1つである)。
2. ユーザーエージェントがサポートを想定できるように、Trickle ICEのサポートを必須にする。

使用プロトコルがTrickle ICEがサポートされているかどうかを事前に判断する方法を提供していない場合、 エージェントはセクション16で述べられているHalf Trickle手順を使用することができる。

最初のICE記述を伝える前に、能力発見をサポートするusingプロトコルを実装するエージェントは、リモートパーティがTrickle ICEをサポートしているかどうかの検証を試みることができる。 エージェントがリモートパーティがTrickle ICEをサポートしていないと判断した場合、エージェントは通常のICEの使用にフォールバックするか、セッション全体を放棄しなければならない[MUST]。

使用プロトコルが能力発見メソッドを含まない場合でも、ユーザーエージェントは、ICEオプションの'trickle'を通信することで、ICE記述の中でTrickle ICEをサポートしていることを示すことができる。 このトークンは、セッションレベルで提供されなければならない[MUST]か、 データストリームレベルで提供される場合、すべてのデータストリームに対して提供されなければならない[MUST NOT]。 注意: 「Trickle」ICEオプションのエンコーディングと、それを相手に伝えるために使用されるメッセージはプロトコル固有である。例えば、セッション記述プロトコル(SDP) [RFC4566]のエンコーディングは、[I-D.ietf-mmusic-trickle-ice-sip]で定義されている。

専用のディスカバリーセマンティクスとHalf Trickleは、ICEセッションの開始前にのみ必要である。 ICEセッションが確立され、両当事者のTrickle ICEサポートが確認された後、どちらかのエージェントは、その後の交換にFull Trickleを使用することができる(セクション15も参照のこと)。

## 4.  初期ICE記述の生成

ICEエージェントは、通信が差し迫っていることを示す兆候(例えば、ユーザインタフェースの合図や通信セッションを開始するための明示的な要求)があれば、すぐに候補の収集を開始することができる。 通常のICEとは異なり、Trickle ICEの実装では、ブロッキングの方法で候補を収集する必要はない。 したがって、Half Trickleが使用されていない限り、イニシエータが初期のICE記述を可能な限り早く生成して送信すると、ユーザーエクスペリエンスが改善される(したがって、リモートパーティが候補の収集とTricklingを開始することを可能にする)。

イニシエータは、最初のICE記述を伝えるときに、候補の任意の組み合わせを含めてもよい[MAY]。 これには、イニシエータが使用する予定のすべての候補を伝える(Half Trickleのように)、 公開可能なIPアドレスのみを伝える(例えば、ファイアウォールの後ろにいないことが知られているデータリレーの候補)、または候補をまったく伝えない(この場合、イニシエータはレスポンダの初期候補リストをより早く取得でき、レスポンダはより早く候補の収集を開始できる) 可能性が含まれる。

最初のICE記述に含まれる候補については、優先度と基盤の計算、候補の冗長性の決定などの方法は、通常のICE [rfc5245bis]と同様に動作する。

## 5.  初期ICE記述の処理と初期ICEレスポンスの生成

レスポンダは最初の ICE 記述を受信すると、まず、セクション 3 で説明したように ICE 記述またはイニシエータが Trickle ICE をサポートしているかどうかをチェックする。 そうでない場合、レスポンダは通常の ICE 手順 [rfc5245bis]に従って初期 ICE 記述を処理しなければならない[MUST] (または、ICE サポートが全く検出されない場合は、オファー/アンサー処理ルール [RFC3264]などの使用プロトコルに関連する処理ルールに従って)。 ただし、Trickle ICE のサポートが確認された場合、レスポンダは自動的に通常の ICE もサポートしているものとする。

最初の ICE 記述が Trickle ICE のサポートを示している場合、レスポンダは自分の役割を決定し、候補の収集と優先順位付けを開始し、その間、イニシエータとレスポンダの両方がチェックリストを作成して接続性のチェックを開始できるように、最初の ICE 応答を伝えることで応答します。

レスポンダは、候補を収集している間、どの時点でも初期 ICE 記述に応答することができる。 最初の ICE レスポンスは、すべての候補を含む候補のセットを含んでもよいし、候補なしのセットを含んでもよい[MAY]。 (no candidatesを含める利点は、最初のICE応答をできるだけ早く伝えることである。これにより、両当事者は ICEセッションをできるだけ早くアクティブネゴシエーション中であると考えることができる)。

セクション3で述べたように、SDPを使用するプロトコルを使用する場合、最初のICE応答は、ice-options属性に 「trickle」のトークンを含めることで、Trickle ICEのサポートを示すことができる。

## 6.  初期ICEレスポンスの処理

最初のICE応答を処理する際、イニシエータは通常のICE手順に従って役割を決定し、その後、チェックリストを作成し（セクション7）、接続性チェックを行う（セクション8）。

## 7.  チェックリストの作成

通常のICE手順[rfc5245bis]によれば、候補のペアリングを可能にし、冗長な候補を剪定するためには、最初のICE記述と最初のICE応答で候補を提供する必要がある。 対照的に、Trickle ICEの下では、候補が伝達されるか受信されるまでチェックリストは空になる可能性がある。したがって、Trickle ICEエージェントは、通常のICEエージェントとは少し異なる方法でチェックリストの形成と候補者のペアリングを処理する: エージェントは依然としてチェックリストを形成するが、エージェントは、そのチェックリストの候補者ペアが実際に存在した後にのみ、与えられたチェックリストを生成する。 すべてのチェックリストは、たとえチェックリストが空であっても、最初はRunning状態に置かれます(これは[rfc5245bis]のセクション6.1.2.1と一致しています)。

## 8.  接続性チェックの実行

[rfc5245bis]で指定されているように、タイマーTaが切れるときはいつでも、候補ペアの接続性チェックをスケジューリングするときに、実行中状態のチェックリストだけがピックされる。 したがって、Trickle ICEエージェントは、候補ペアがチェックリストにインクリメンタルに追加されることを期待する限り、各チェックリストをRunning状態に保持しなければならない[MUST]。その後、チェックリストの状態は[rfc5245bis]の手順に従って設定される。

タイマーTaが切れて空のチェックリストが選ばれると、そのリストには何もアクションが実行されません。 タイマーTaが再び満了するのを待たずに、エージェントは[rfc5245bis]のセクション6.1.4.2に従って、実行中状態で次のチェックリストを選択します。

[rfc5245bis]のセクション7.2.5.3.3は、エージェントが接続性チェック・トランザクションの完了時にチェックリストとタイマー状態を更新することを要求している。 このような更新の間、通常のICEエージェントは、以下の2つの条件の両方が満たされた場合、チェックリストの状態を「Failed」に設定する。

- チェックリスト内のすべてのペアが「失敗」または「成功」の状態にある。
- データストリームの各コンポーネントの有効リストにペアが存在しない。

Trickle ICEでは、将来のチェックが成功する可能性が十分にあるにもかかわらず、候補の収集とTricklingがまだ進行中である場合、上記の状況がしばしば発生します。 このため、Trickle ICEエージェントは、上記のリストに以下の条件を追加します。

- すべての候補者収集が完了し、エージェントは新しいローカル候補者を発見することを期待していない。
- リモートエージェントが、セクション13に記載されているように、そのチェックリストの候補終了の指示を伝えた。

## 9.  新たに集まったLocal Candidatesの収集と発信

Trickle ICEエージェントが初期ICE記述と初期ICE応答を伝えた後、STUN、TURN、および他の非ホスト候補収集メカニズムが結果を出し始めると、彼らは新しいローカル候補を収集し続ける可能性が高い。 エージェントがそのような新しい候補を発見するたびに、エージェントは、通常のICE手順に従って、その優先度、タイプ、基盤、およびコンポーネントIDを計算する。

次に、新しい候補は、既存のローカル候補のリストに対して冗長性があるかどうかチェックされる。 そのトランスポートアドレスとベースが既存の候補と一致する場合、それは冗長であるとみなされ、無視される。 これは、取得元のホストアドレスと一致するサーバ再帰的な候補でよく起こる(例: 後者がパブリックIPv4アドレスの場合)。 通常のICEとは逆に、Trickle ICEエージェントは、その優先度に関係なく、新しい候補を冗長化したものとみなします。

次にエージェントは、新たに発見された候補をリモートエージェントに「Trickle」する。 新しい候補の実際の配送は、SIPやXMPPのようなプロトコルを使用して処理される。 Trickle ICEは、この方法に制限を課さない(例えば、いくつかの使用プロトコルでは、 server reflexive candidatesの更新をTrickleしないことを選択し、代わりにpeer reflexive candidatesの発見に依存するかもしれない)。

候補がTrickleされるとき、使用プロトコルは、各候補(およびセクション 13で述べられているようなEnd-of-Candidates表示)を、受信側のTrickle ICE実装に正確に一度だけ、それが伝えられたのと同じ順番で配送しなければならない[MUST]。 使用プロトコルがいかなる候補の再送も提供する場合、それはICE実装から隠蔽される必要がある。

また、候補のTricklingは、ICEの再起動がある場合、前のセッションに対する遅延した更新がそのように認識され、受信側によって無視されることができるように、特定のICEセッションと相関関係を持つ必要がある。 例えば、SDP経由で候補者にシグナルを送るプロトコルを使用する場合、対応する a=candidate行にUsername Fragment値を含めることができる。

    a=candidate:1 1 1 UDP 2130706431 2001:db8::1 5000 typ host ufrag 8hhY

あるいは、別の例として、WebRTC の実装では、候補を表す JavaScript オブジェクトにユーザー名フラグメントを含めることができます。

注意: 使用プロトコルは、(Username Fragment と Password の組み合わせで識別される)有効な ICE セッションを示し、同意するためのメカニズムを提供する必要がある。これはICE再起動の場合に特に重要である(セクション15参照)。

注意: 使用するプロトコルは、公開されていることが知られていて、直接STUNバインディングリクエストを送る方が、シグナリングパスを通過するTrickle更新よりも速く宛先に到達する可能性が高いエンティティに対して、server reflexive candidatesをTrickleしないことを好むかもしれない。

## 10.  新たに集まったLocal Candidatesとのペアリング

Trickle ICEエージェントがローカルの候補を収集する際には、候補のペアを形成する必要がある。

1. Trickle ICEエージェントは、リモートパーティにTrickleされるまでローカル候補をペアにしてはならない[MUST NOT]。
2. いったんエージェントがローカル候補をリモートパーティに伝えると、エージェントは、この同じストリームとコンポーネントに対して、現在知られているリモート候補があるかどうかをチェックする。 もしそうでなければ、エージェントは単にその新しい候補を(ペアリングせずに)ローカル候補のリストに追加するだけである。
3. そうでなければ、エージェントがこのストリームとコンポーネントに対して1つ以上のリモー ト候補をすでに知っている場合、ICE仕様[rfc5245bis]に記述されているように、新しいローカル候補のペアリングを 試みます。
4. 新しく形成されたペアが、タイプが server reflexiveなローカル候補を持つ場合、 エージェントは、関連する冗長性テストを完了する前に、そのローカル候補をそのベースで置き換えなければならない[MUST]。
5. エージェントは[rfc5245bis]のセクション6.1.2.4のルールに従って冗長ペアを削除しますが、既存のペアが「Waiting」または「Frozen」の状態にある場合にのみチェックします。これにより、接続性チェックが進行中のペア(In-Progressの状態)や、接続性チェックですでに確定的な結果が得られているペア(Succeeded or Failedの状態)の削除を避けることができます。
6. 関連する冗長性テストの後で、ペアが追加されるチェックリストが既に最大数の候補ペアを含んでいる場合([rfc5245bis]のデフォルトでは100)、エージェントは新しいペアのための余地を作るために「失敗」状態のペアをすべて破棄すべきである[SHOULD]。 そのようなペアがない場合、エージェントは、ペアの数がペアの最大数に等しくなるまで、新しいペアのための余地を作るために、新しいペアよりも低い優先度のペアを破棄するべきである[SHOULD]。 この処理は[rfc5245bis]のセクション6.1.2.5と一致している。

## 11.  Trickled Candidateの受信

ICEセッション中はいつでも、Trickle ICEエージェントはリモートエージェントから新しい候補を受け取るかもしれない。

1. エージェントは、この同じストリームとコンポーネントに対して、ローカルの候補が現在知られているかどうかをチェックする。 もしそうでなければ、エージェントは単に新しい候補をリモート候補のリストに追加するだけである(ペアリングは行わない)。
2. そうでなければ、エージェントがこのストリームとコンポーネントに対して1つ以上のローカル候補をすでに集めていた場合、ICE仕様[rfc5245bis]に記述されているように、新しいリモート候補をペアリングしようとする。
3. 新しく形成されたペアに、タイプが server reflexiveなローカル候補がある場合、 エージェントは、次のステップで冗長性チェックを完了する前に、 そのローカル候補をそのベースに置き換えなければならない[MUST]。
4. これにより、接続性チェックが飛行中(In-Progressの状態)であるペアや、接続性チェックですでに確定的な結果が得られているペア(SucceededまたはFailedの状態)の削除を避けることができる。

    A.  エージェントが2つのペア間に冗長性を見つけ、それらのペアのうちの1つが、タイプがpeer reflexiveである新たに受信したリモート候補を含む場合、エージェントは、その候補を含むペアを破棄し、既存のペアの優先度を破棄したペアの優先度に設定し、チェックリストを再ソートするべきである[SHOULD]。 (このポリシーは、候補のシグナリングが受信エージェントにTrickleされる前に STUNバインディングリクエストが受信されるリモートpeer reflexive型候補に関する問題を排除するのに役立つ。例えば、同じ候補が一方のエージェントによって peer reflexive型として認識され、他方のエージェントによってserver reflexive型として認識される可能性があるため、 ローカルエージェントとリモートエージェントの間でペアの優先度が異なる見解になるなど)。

    B.  次にエージェントは、[rfc5245bis]のセクション6.1.2.4で定義されているルールを適用する。

5. 関連する冗長性テストの後で、ペアが追加されるチェックリストがすでに最大数の候補ペアを含んでいる場合([rfc5245bis]のデフォルトでは100)、エージェントは新しいペアのための余地を作るために失敗状態にあるペアをすべて破棄するべきである[SHOULD]。 そのようなペアがない場合、エージェントは、ペアの数がペアの最大数に等しくなるまで、新しいペアのための余地を作るために、新しいペアよりも低い優先度のペアを破棄するべきである[SHOULD]。 この処理は[rfc5245bis]のセクション6.1.2.5と一致している。

## 12.  チェックリストへのTrickled Candidateペアの挿入

ローカルエージェントが候補をTrickleしてそのローカル候補から候補ペアを形成した後(セクション9)、またはリモートエージェントがTrickleした候補を受信してそのリモート候補から候補ペアを形成した後(セクション11)、Trickle ICEエージェントは、このセクションで定義されているように、新しい候補ペアをチェックリストに追加する。

本セクションで定義されている手順を理解するための補助として、以下の表形式でのすべてのチェックリストの表現を考えてみてください。

エージェント(最初は基礎の1つ、すなわちf5については、候補ペアが存在しないことに注意してください)。

```
   +-----------------+------+------+------+------+------+
   |                 |  f1  |  f2  |  f3  |  f4  |  f5  |
   +-----------------+------+------+------+------+------+
   | s1 (Audio.RTP)  |  F   |  F   |  F   |      |      |
   +-----------------+------+------+------+------+------+
   | s2 (Audio.RTCP) |  F   |  F   |  F   |  F   |      |
   +-----------------+------+------+------+------+------+
   | s3 (Video.RTP)  |  F   |      |      |      |      |
   +-----------------+------+------+------+------+------+
   | s4 (Video.RTCP) |  F   |      |      |      |      |
   +-----------------+------+------+------+------+------+
```

図2：チェックリスト状態の例

表の各行は、与えられたデータストリームのコンポーネントを表します（例えば、s1とs2はオーディオ用のRTPとRTCPのコンポーネントかもしれません）。 各列は、1つのファンデーションを表す。 各セルは、1つの候補ペアを表す。本節の表では、「Ｆ」は「Frozen」、「Ｗ」は「Waiting」、「Ｓ」は「Succeeded」を表し、さらに、新たに追加された候補ペアを示すために「^^」を用いる。

エージェントがICE処理を開始するとき、[rfc5245bis]のセクション6.1.2.6に従って、各ファンデーションについて、最も低いコンポーネントIDを持つペアをunfreezeし、コンポーネントIDが等しい場合には、最も高い優先度を持つペアをunfreezeする(これは、すべての列の最上位の候補ペアである)。 この初期状態を以下の表に示します。

```
   +-----------------+------+------+------+------+------+
   |                 |  f1  |  f2  |  f3  |  f4  |  f5  |
   +-----------------+------+------+------+------+------+
   | s1 (Audio.RTP)  |  W   |  W   |  W   |      |      |
   +-----------------+------+------+------+------+------+
   | s2 (Audio.RTCP) |  F   |  F   |  F   |  W   |      |
   +-----------------+------+------+------+------+------+
   | s3 (Video.RTP)  |  F   |      |      |      |      |
   +-----------------+------+------+------+------+------+
   | s4 (Video.RTCP) |  F   |      |      |      |      |
   +-----------------+------+------+------+------+------+
```

図3：初期チェックリストの状態

次に、チェックが進むにつれて([rfc5245bis]のセクション7.2.5.4を参照)、Succeeded状態(ここでは "S "と表記)に入る各ペアについて、エージェントは、同じ基盤を持つ全てのデータストリームについて全てのペアをアンフリーズする(例えば、列1、行1のペアが成功した場合、エージェントは、列1、行2、行3、および4のペアをアンフリーズするだろう)。

```
   +-----------------+------+------+------+------+------+
   |                 |  f1  |  f2  |  f3  |  f4  |  f5  |
   +-----------------+------+------+------+------+------+
   | s1 (Audio.RTP)  |  S   |  W   |  W   |      |      |
   +-----------------+------+------+------+------+------+
   | s2 (Audio.RTCP) |  W   |  F   |  F   |  W   |      |
   +-----------------+------+------+------+------+------+
   | s3 (Video.RTP)  |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+
   | s4 (Video.RTCP) |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+
```

図4: 成功した候補ペアを持つチェックリストの状態

Trickle ICEは、「静的な」チェックリストセットに適用されるこれらのルールをすべて保持する。 これは、Trickle ICEエージェントがすべてのペアがすでに存在する状態で接続性チェックを開始した場合、ペアの状態が変化する方法は通常のICEエージェントのそれと区別がつかないことを意味します。

もちろん、Trickle ICEとの大きな違いは、接続性チェックが開始された後に候補者が到着する可能性があるため、チェックリストセットが動的に更新される可能性があることです。 このような場合、エージェントは、以下に説明するように、新たに形成されたペアの状態を設定する。

ルール1：新たに形成されたペアが、最も低いコンポーネントIDを有し、コンポーネントIDが等しい場合、このファンデーションのための任意の候補ペアの中で最も高い優先度を有する場合（すなわち、列の最上位のペアである場合）、状態をWaitingに設定する。 例えば、新たに形成されたペアが5列目の1列目に配置された場合はこのようになります。 このルールは、[rfc5245bis]のセクション6.1.2.6と一致しています。

```
   +-----------------+------+------+------+------+------+
   |                 |  f1  |  f2  |  f3  |  f4  |  f5  |
   +-----------------+------+------+------+------+------+
   | s1 (Audio.RTP)  |  S   |  W   |  W   |      | ^W^  |
   +-----------------+------+------+------+------+------+
   | s2 (Audio.RTCP) |  W   |  F   |  F   |  W   |      |
   +-----------------+------+------+------+------+------+
   | s3 (Video.RTP)  |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+
   | s4 (Video.RTCP) |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+
```

図5: ルール1のペアが新規結成されたチェックリスト状態

ルール2：このファンデーションのSucceeded状態のペアが少なくとも１つある場合、その状態をWaiting状態にする。 例えば、5列目1行目のペアが成功し、新たに形成されたペアが5列目2行目に配置された場合がこれに該当する。 このルールは、[rfc5245bis]のセクション7.2.5.3.3と一致している。

```
   +-----------------+------+------+------+------+------+
   |                 |  f1  |  f2  |  f3  |  f4  |  f5  |
   +-----------------+------+------+------+------+------+
   | s1 (Audio.RTP)  |  S   |  W   |  W   |      |  S   |
   +-----------------+------+------+------+------+------+
   | s2 (Audio.RTCP) |  W   |  F   |  F   |  W   | ^W^  |
   +-----------------+------+------+------+------+------+
   | s3 (Video.RTP)  |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+
   | s4 (Video.RTCP) |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+
```

図6：ルール2の新たに形成されたペアが配置されたチェックリスト状態

ルール3：それ以外の場合は、状態をFrozenに設定する。 例えば、新たに形成されたペアが3列目の3行目に配置されている場合です。

```
   +-----------------+------+------+------+------+------+
   |                 |  f1  |  f2  |  f3  |  f4  |  f5  |
   +-----------------+------+------+------+------+------+
   | s1 (Audio.RTP)  |  S   |  W   |  W   |      |  S   |
   +-----------------+------+------+------+------+------+
   | s2 (Audio.RTCP) |  W   |  F   |  F   |  W   |  W   |
   +-----------------+------+------+------+------+------+
   | s3 (Video.RTP)  |  W   |      | ^F^  |      |      |
   +-----------------+------+------+------+------+------+
   | s4 (Video.RTCP) |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+
```

図7: 新規に形成されたペアがあるチェックリストの状態、ルール3

## 13.  End-of-Candidatesの表示の生成

特定のデータストリームに関連付けられたICEセッションのためにすべての候補の収集が完了するか、または期限切れになると、エージェントはそのセッションのための 「end-of-candidates」指示を生成し、シグナリングチャネルを介してリモートエージェントにそれを伝える。 指示の正確な形式は使用プロトコルに依存するが、指示は、エージェントが特定のICEセッションとend-of-cidates指示を関連付けることができるように、生成(Username FragmentとPasswordの組み合わせ)を指定しなければならない[MUST]。 指示は以下の方法で伝えることができる。

- 開始リクエストの一部として(これは通常、Half Trickleの最初のICE記述の場合である)
- エージェントがストリームのために送ることができる最後の候補と一緒に
- スタンドアロンの通知として(例えば、STUN BindingリクエストやTURN Allocateリクエストがサーバータイムアウトしてエージェントが積極的に候補を収集しなくなった後)

曖昧さを回避し、ICE処理の終了を迅速にするためには、タイムリーな方法で候補者の終了表示を伝えることが重要である。 特に

- 制御されたTrickle ICEエージェントは、エージェントがギャザリングを完了するチャンスがある前にICE処理が終了しない限り、データストリームのギャザリングを完了 した後に、End-of-Candidatesの指示を伝えるべきである[SHOULD]。
- 制御するエージェントは、すべてのストリームについてEnd-of-Candidatesの指示を伝える前に、 ICE処理を終了してもよい[MAY]。 しかしながら、一貫性のために、そしてICE処理の状態についてミドルボックスと制御されたエージェントを最新の状態に保つために、可能なときはいつでもEnd-of-Candidatesの指示を伝えることが制御するエージェントに推奨される[RECOMMENDED]。

Trickling中に(最初のICE記述またはそれに対する応答の一部としてではなく) end-of-candidates指示を伝えるとき、その指示を1つ以上の特定のデータストリームに関連付けるための方法を定義することは、使用プロトコルの責任である。

エージェントはまた、エージェントがギャザリングが許容される期間を超えて継続しているとエージェントが判断した場合、候補の収集が実際に完了する前に候補の終了指示を生成することを選択してもよい[MAY]。 しかしながら、エージェントは、End-of-Candidates表示を伝えた後、それ以上の候補を伝えてはならない[MUST NOT]。

Half Trickleを行うとき、エージェントは、潜在的に追加の候補をTrickleすることを 計画していない限り(例えば、リモートパーティがTrickle ICEをサポートしていることが判明した場合)、最初のICE記述と一緒に候補の終了指示を伝えるべきである[SHOULD]。

エージェントがend-of-cididates指示を伝えた後、それはセクション8で説明されているように、 対応するチェックリストのステートを更新する。 それ以降、エージェントはこのICEセッション内で新規候補をTrickleしてはならない[MUST NOT]。 したがって、ネゴシエーションに新しい候補を追加することは、ICE再起動(セクション15参照)を介してのみ可能である。

この仕様は、ICE処理を終了するための通常のICEセマンティクスを上書きしない。 したがって、End-of-Candidatesの指示が伝達されたとしても、エージェントはペア指名を経る必要がある。 また、ペアがコンポーネントとデータストリームに対して指名されている場合、たとえすべてのストリームに対してend-of-cidatesの指示が受信されていなくても、ICE処理は終了してもよい[MAY]。 すべての場合において、[rfc5245bis]のセクション8.1.1.1で述べられているように、エージェントは候補ペアの指名後にICEセッション内で新しい候補をTrickleしてはならない[MUST NOT]。

## 14.  End-of-Candatesインジケーションの受信

End-of-Candidatesの指示を受信することで、エージェントはチェックリストの状態を更新し、すべてのデータストリームのすべてのコンポーネントに対して有効なペアが存在しない場合、ICE処理が失敗したと判断することを可能にする。 また、候補ペアが検証されたが、TURNのようなより低い優先度のトランスポートの使用を伴う場合、エージェントはICE処理の終了を迅速化することができる。 このような状況では、実装は、より高い優先度の候補が受信されるかどうかを待って確認することを選択してもよい[MAY]。

表示は、そのような候補が来ないという通知を提供する。

エージェントが特定のデータストリームに対するEnd-of-Candidates指示を受け取ると、エージェントは、セクション８に従って、関連するチェックリストの状態を更新する（これは、いくつかのチェックリストが失敗とマークされることにつながるかもしれない）。 更新後にチェックリストがまだ実行中の状態にある場合、エージェントは、End-of-Candidates指示を受信したという事実を永続化し、チェックリストへの将来の更新でそれを考慮に入れる。

エージェントがEnd-of-Candidates指示を受け取った後、そのデータストリームまたはデータセッションに対して新たに受け取ったいかなる候補も無視しなければならない[MUST]。

## 15.  後続の交換とICEの再起動

end-of-candidatesの指示を伝える前に、どちらかのエージェントは、 使用プロトコルによって許可されたいつでも後続の候補情報を伝えてもよい[MAY]。 これが起こるとき、エージェントは[rfc5245bis]セマンティクス(例えば、ユーザー名フラグメントとパスワードの組み合わせのチェック)を使用して、新しい候補情報がICEの再起動を必要とするかどうかを判断する。

ICE再起動が発生した場合、エージェントは、サポートが以前に決定されていた場合、Trickle ICEがまだサポートされていると仮定することができ、したがって、能力発見方法によってサポートが決定されたICE記述の最初の交換の場合と同様に、Trickle ICEの動作に従事することができる。

## 16.  Half Trickle


Half Trickleでは、イニシエータは最初の ICE 記述を、使用可能な候補を含むが、必ずしも完全な生成を行う必要はない。これは、通常の ICE レスポンダが ICE 記述を処理できることを保証するものであり、初期 ICE 記述を伝える前に Trickle ICE のサポートが確認できない場合に使用することを主眼としている。 最初の ICE 記述は Trickle ICE をサポートしていることを示し、レスポンダは候補の完全な世代未満のもので応答し、残りを Trickle することができます。 Half Trickleの最初の ICE 記述には、候補の終了表示を含めることができるが、Trickleのサポートが確認された場合、イニシエータは候補の終了表示を伝える前に追加の候補をTrickleすることを選択できるため、これは必須ではない。

Half Trickleメカニズムは、エージェントがリモートパーティがTrickle ICEをサポートしているかどうかを事前に確認する方法がない場合に使用できる。 最初のICE記述は候補の完全な生成を含むので、通常のICEエージェントによって処理され、一方でTrickle ICEエージェントが本仕様で定義された最適化を使用することを可能にする。 これにより、前者ではネゴシエーションの失敗を防ぎ、後者ではTrickle ICEの利点の約半分を与えることができます。

Half Trickleの使用は、ICE記述の最初の交換の間にのみ必要である。 両当事者がピアからICE記述を受け取った後、それぞれがTrickle ICEサポートを確実に判断し、それ以降のすべての交換に使用することができる(セクション15参照)。

いくつかの例では、Half Trickleを使用することは、ユーザエクスペリエンスの点で半分以上の改善をもたらすかもしれない。 これは、エージェントが、キーパッド上での活動や電話がフックから外れるなど、ユーザーがすぐにインタラクションを開始するというユーザーインターフェースの合図に基づいて候補の収集を開始したときに起こりうる。 これは、エージェントが実際に候補者情報を伝える必要がある前に、候補者収集の一部またはすべてが完了する可能性があることを意味する。 レスポンダは候補者をTrickleすることができるので、両方のエージェントは、通常のICEよりも早く接続性チェックを開始し、ICE処理を完了することができ、潜在的にはFull Trickleよりも早く完了する可能性がある。

しかし、そのような予測は常に可能とは限らない。 例えば、通信が非中央機能（例えば、主要機能に問題がある場合にサポートラインに電話をかけるなど）である多目的ユーザエージェントやWebRTC Webページは、必ずしも呼出意図と他のユーザ活動を区別する方法を持っているとは限らない。 そのような場合には、Full Trickleを使用することが理想的なユーザー・エクスペリエンスをもたらす可能性が高い。 そうであっても、Half Trickleを使用することで、通常のICEよりも応答者の体験を向上させることができるため、通常のICEよりも改善されると考えられる。

## 17.  Trickle中に候補者の順番を保持する

通常のICEの重要な側面の一つは、特定の基盤とコンポーネントの接続性チェックが両方のエージェントによって同時に試みられることである。 これは、適切なタイミングで候補のFrozenを解除するためにも重要です。 重要ではありませんが、Trickle ICEでこの動作を維持することはICEのパフォーマンスを向上させる可能性があります。

これを達成するために、候補をTrickleするときに、エージェントはコンポーネントIDによって反映されるコンポーネントの順番を尊重すべきである[SHOULD]。

同じファンデーションの中で、より低いID番号のコンポーネントの候補者をペアリングすべきである[SHOULD]。 さらに、セクション12の手順に従って、伝達された順番と同じ順番で候補をペアにするべきである[SHOULD]。

たとえば、以下のSDP記述には、2つのコンポーネント(RTPとRTCP)と2つのファウンデーション (hostとserver reflexive)が含まれている。

```
     v=0
     o=jdoe 2890844526 2890842807 IN IP4 10.0.1.1
     s=
     c=IN IP4 10.0.1.1
     t=0 0
     a=ice-pwd:asd88fgpdd777uzjYhagZg
     a=ice-ufrag:8hhY
     m=audio 5000 RTP/AVP 0
     a=rtpmap:0 PCMU/8000
     a=candidate:1 1 UDP 2130706431 10.0.1.1 5000 typ host
     a=candidate:1 2 UDP 2130706431 10.0.1.1 5001 typ host
     a=candidate:2 1 UDP 1694498815 192.0.2.3 5000 typ srflx
         raddr 10.0.1.1 rport 8998
     a=candidate:2 2 UDP 1694498815 192.0.2.3 5001 typ srflx
         raddr 10.0.1.1 rport 8998
```

この候補情報では、RTCPホスト候補はRTPホスト候補の前には伝えられない。 同様に、RTP server reflexive candidatesは、RTCP server reflexive candidatesと一緒に、またはRTCP server reflexive candidatesよりも前に伝達される。

## 18.  プロトコルを使用するための要件

Trickle ICEの使用を完全に可能にするために、本仕様では、プロトコルを使用するための以下の要件を定義する。

- 使用プロトコルは、ICEセッションが始まる前に、パーティがTrickle ICEのサポートをアドバタイズして発見するための方法を提供するべきである[SHOULD] (セクション3参照)。
- 使用プロトコルは、最初のICE記述を伝えた後に追加の候補をインクリメンタルに伝える(すなわち「Trickle」)ための方法を提供しなければならない[MUST] (セクション9参照)。
- 使用プロトコルは、各Trickled Candidateまたはend-of-candidate指示を、正確に一度だけ、それが伝えられたのと同じ順番で伝えなければならない[MUST] (セクション9参照)。
- 使用プロトコルは、双方が発効中のICEセッションを示し、同意するためのメカニズムを提供しなければならない[MUST] (セクション9参照)。
- 使用プロトコルは、当事者がEnd-of-Candidatesの指示を伝える方法を提供しなければならず[MUST]、その指示が適用される特定のICEセッションを特定しなければならない[MUST] (セクション13参照)。

## 19.  IANAの考察

IANAでは、[RFC6336]で定義されている手順に従って、"Interactive Connectivity Establishment (ICE) registry" の "ICE Options" サブレジストリに以下のICEオプションを登録することを要求しています。

     ICE Option:  trickle
     Contact:  IESG, iesg@ietf.org
     Change control:  IESG
     Description:  An ICE option of "trickle" indicates support for incremental communication of ICE candidates.
     Reference:  RFC XXXX

## 20.  セキュリティに関する考察

この仕様はそのセマンティクスの大部分を[rfc5245bis]から継承しており、結果としてそこで述べられているセキュリティ上の考慮事項はすべてTrickle ICEに適用される。

エンドポイントデバイス上のホストアドレスを明かすことによるプライバシーへの影響が懸念される場合(例えば、[I-D.ietf-rtcweb-ip-handling]や[rfc5245bis]のセクション19の議論を参照)、エージェントは候補を含まないICE記述を生成し、ホストアドレスを明かさない候補(例えば、中継された候補)のみをTrickleすることができる。

## 21.  謝辞

この文書を改善するためのレビューや提案をしてくれた Bernard Aboba, Flemming Andreasen, Rajmohan Banavi, Taylor Brandstetter, Philipp Hancke, Christer Holmberg, Ari Keranen, Paul Kyzivat, Jonathan Lennox, Enrico Marocco, Pal Martinsen, Nils Ohlmeier, Thomas Stach, Peter Thatcher, Martin Thomson, Brandon Williams, Dale Worley に感謝の意を表したいと思う。 Sarah Banks、Roni Even、David Mandelberg はそれぞれ opsdir、genart、セキュリティのレビューを完了しました。 また、議長としての役割を果たした Ari Keranen と Peter Thatcher、責任あるエリアディレクターとしての役割を果たした Ben Campbell にも感謝する。

## 22.  参考文献

### 22.1.  Normative References

[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate            Requirement Levels", BCP 14, RFC 2119,            DOI 10.17487/RFC2119, March 1997, <https://www.rfc-            editor.org/info/rfc2119>.

[rfc5245bis]            Keranen, A., Holmberg, C., and J. Rosenberg, "Interactive            Connectivity Establishment (ICE): A Protocol for Network            Address Translator (NAT) Traversal", draft-ietf-ice-            rfc5245bis-20 (work in progress), March 2018.

### 22.2.  Informative References


[I-D.ietf-mmusic-trickle-ice-sip]            Ivov, E., Stach, T., Marocco, E., and C. Holmberg, "A            Session Initiation Protocol (SIP) usage for Trickle ICE",            draft-ietf-mmusic-trickle-ice-sip-14 (work in progress),            February 2018.

[I-D.ietf-rtcweb-ip-handling]            Uberti, J. and G. Shieh, "WebRTC IP Address Handling            Requirements", draft-ietf-rtcweb-ip-handling-06 (work in            progress), March 2018.

[RFC1918]  Rekhter, Y., Moskowitz, B., Karrenberg, D., de Groot, G.,            and E. Lear, "Address Allocation for Private Internets",            BCP 5, RFC 1918, DOI 10.17487/RFC1918, February 1996,            <https://www.rfc-editor.org/info/rfc1918>.

[RFC3261]  Rosenberg, J., Schulzrinne, H., Camarillo, G., Johnston,            A., Peterson, J., Sparks, R., Handley, M., and E.            Schooler, "SIP: Session Initiation Protocol", RFC 3261,            DOI 10.17487/RFC3261, June 2002, <https://www.rfc-            editor.org/info/rfc3261>.

[RFC3264]  Rosenberg, J. and H. Schulzrinne, "An Offer/Answer Model            with Session Description Protocol (SDP)", RFC 3264,            DOI 10.17487/RFC3264, June 2002, <https://www.rfc-            editor.org/info/rfc3264>.

[RFC4566]  Handley, M., Jacobson, V., and C. Perkins, "SDP: Session            Description Protocol", RFC 4566, DOI 10.17487/RFC4566,            July 2006, <https://www.rfc-editor.org/info/rfc4566>.

[RFC4787]  Audet, F., Ed. and C. Jennings, "Network Address            Translation (NAT) Behavioral Requirements for Unicast            UDP", BCP 127, RFC 4787, DOI 10.17487/RFC4787, January            2007, <https://www.rfc-editor.org/info/rfc4787>.

[RFC5389]  Rosenberg, J., Mahy, R., Matthews, P., and D. Wing,            "Session Traversal Utilities for NAT (STUN)", RFC 5389,            DOI 10.17487/RFC5389, October 2008, <https://www.rfc-            editor.org/info/rfc5389>.

[RFC5766]  Mahy, R., Matthews, P., and J. Rosenberg, "Traversal Using            Relays around NAT (TURN): Relay Extensions to Session            Traversal Utilities for NAT (STUN)", RFC 5766,            DOI 10.17487/RFC5766, April 2010, <https://www.rfc-            editor.org/info/rfc5766>.

[RFC6120]  Saint-Andre, P., "Extensible Messaging and Presence            Protocol (XMPP): Core", RFC 6120, DOI 10.17487/RFC6120,            March 2011, <https://www.rfc-editor.org/info/rfc6120>.

[RFC6336]  Westerlund, M. and C. Perkins, "IANA Registry for            Interactive Connectivity Establishment (ICE) Options",            RFC 6336, DOI 10.17487/RFC6336, July 2011,            <https://www.rfc-editor.org/info/rfc6336>.

[XEP-0030]            Hildebrand, J., Millard, P., Eatmon, R., and P. Saint-            Andre, "XEP-0030: Service Discovery", XEP XEP-0030, June            2008.

[XEP-0176]            Beda, J., Ludwig, S., Saint-Andre, P., Hildebrand, J.,            Egan, S., and R. McQueen, "XEP-0176: Jingle ICE-UDP            Transport Method", XEP XEP-0176, June 2009.

## 付録A.  通常のICEとのやりとり

ICE プロトコルは、可能な限り多くのネットワーク環境で動作し、適応できるように柔軟に設計されています。 その柔軟性にもかかわらず、[rfc5245bis]で規定されているICEはそれ自体がTrickle ICEをサポートしていません。 このセクションでは、候補のTricklingがICEとどのように相互作用するかを説明します。

[rfc5245bis]は、ICEエージェントがRunning状態にある間、チェックリストとタイマー状態を更新するために必要な条件を記述しています。 これらの条件はトランザクション完了時に検証され、そのうちの1つは次のように規定している。

    データストリームの各コンポーネントに対して有効なリストにペアがない場合、チェックリストの状態は「失敗」に設定される。

これは問題であり、多くのシナリオでICE処理が早期に失敗する原因となります。 以下のケースを考えてみましょう。

1. アリスとボブは、ネットワークアドレス変換(NAT)のある異なるネットワークに存在します。 アリスとボブ自身は異なるアドレスを持っているが、どちらのネットワークも同じプライベートインターネットブロック(例えば、[RFC1918]で指定されている「20ビットブロック」172.16/12)を使用している。
2. AliceはBobに172.16.0.1の候補を伝えるが、それはBobのネットワーク上の既存のホストに偶然にも対応している。
3. Bobは172.16.0.1だけで構成されるチェックリストを作成し、チェックを開始する。
4. これらのチェックはBobのネットワーク上の172.16.0.1のホストに到達し、ICMP "port unreachable "エラーで応答します。

この時点でチェックリストには失敗の候補だけが含まれており、有効なリストは空です。 これにより、データストリームとすべてのICE処理が失敗する可能性がありますが、Trickle ICEエージェントがその後に候補を伝えることができた場合、以前は空だったチェックリストが空にならないようになります。

同様の競合状態は、Aliceからの最初のICEの記述に、Bobが収集した候補の中から到達不可能と判断できる候補のみが含まれている場合に発生するだろう(例えば、Bobの候補がIPv4アドレスしか含まれておらず、Aliceから受け取った最初の候補がIPv6のものであった場合などがこれに該当する)。

もう一つの潜在的な問題は、非Trickle ICE実装がTrickle ICE実装との相互作用を開始した場合に発生する可能性があります。 以下のケースを考えてみましょう。

1. AliceのクライアントがTrickle以外のICE実装を持っている。
2. Bob のクライアントが Trickle ICE をサポートしている。
3.  AliceとBobはアドレス依存フィルタリング[RFC4787]を持つNATの後ろにいる。
4. Bobは2つのSTUNサーバを持っているが、そのうちの1つは現在到達不可能である。

BobのエージェントがAliceの最初のICE記述を受け取ると、すぐに接続性チェックを開始する。 また、候補者の収集を開始するが、STUNサーバに到達できないため、長い時間がかかるだろう。 Bobの回答が準備されてAliceに伝わる頃には、Bobの接続性チェックは失敗しているかもしれません: AliceがBobの回答を受け取るまで、Aliceは接続性チェックを開始することができず、NATに穴を開けることができません。 そのため、NATはBobの接続性チェックを未知のエンドポイントからのものとしてフィルタリングしていることになります。

## 付録B.  ICE Liteとの連携

Trickle ICEが可能なICE liteエージェントの動作は、この仕様および[rfc5245bis]ですでに定義されているもの以外の特定のルールを必要としない。 したがって、このセクションは情報提供のみを目的として提供される。

ICE liteエージェントは[rfc5245bis]に従って候補情報を生成し、Trickle ICEのサポートを示す。 候補情報が候補のフルジェネレーションを含むことを考えると、それはまた、候補の終わりの表示を伴うだろう。

Full Trickleを実行する場合、完全なICE実装は、最初のICE記述またはそれに対する応答を候補なしで伝えることができる。リモートエージェントをICE lite実装として識別する応答を受け取った後、開始者は、追加の候補をTrickleしないことを選択することができる。 ICE liteエージェントがインタラクションを開始し、完全なICEエージェントがレスポンダである場合も同様である。 これらの場合、ICE lite実装は、接続性チェックを行うだけで、peer reflexiveにすべての潜在的に有用な候補を発見することができる。 以下の例は、SDP構文を使用したICEセッションの例である。

```
           ICE Lite                                          Bob
            Agent
              |   Offer (a=ice-lite a=ice-options:trickle)    |
              |---------------------------------------------->|
              |                                               |no cand
              |         Answer (a=ice-options:trickle)        |trickling
              |<----------------------------------------------|
              |              Connectivity Checks              |
              |<--------------------------------------------->|
     peer rflx|                                               |
    cand disco|                                               |
              |<========== CONNECTION ESTABLISHED ===========>|
```

図8：例

シグナリング・トラフィックの削減に加えて、このアプローチはSTUNバインディングの検出やTURN割り当ての必要性を排除し、ICE処理を大幅に軽量化することができます。

## 付録C.  以前のバージョンからの変更点

RFC編集者への注意: RFCとして公開される前に、このセクションを削除してください。

### C.1.  draft-ietf-ice-trickle-20 からの変更点

- peer reflexive candidatesのハンドリングをわずかに修正しました。
- いくつかのセクションでの言葉遣いの修正。

### C.2.  draft-ietf-ice-trickle-19 からの変更点

- リモートpeer reflexive候補者の取り扱いをさらに明確にした。
- 読みやすさを向上させるために、いくつかのセクションとサブセクションの名称を変更し、再構成し、いくつかの文言を修正した。

### C.3.  draft-ietf-ice-trickle-18 からの変更点

- IESGのフィードバックとWGの議論に基づき、新たに発見された候補者のためのペアリングと冗長性チェックのルールをクリーンアップしました。
- half trickleセクションの表現を改善した。
- "not more than once "を "accordly once "に変更。
- NATの例をIPv4に戻した。

### C.4.  draft-ietf-ice-trickle-17 からの変更点

- チェックリストに新しいペアを挿入する際のルールを簡略化しました。
- 既に指名されているペアの後に候補のペアを指名することはできないことを明確にしました。
- 古いバージョンのrfc5245bisを参照していたいくつかの文章を削除しました。
- rfc5245bisで指定されている概念や手順と重複している文章を削除しました。
- ストリームオーダーの不明確な概念を削除。
- 導入部を短くした。

### C.5.  draft-ietf-ice-trickle-16 からの変更点

- 「ufrag」の用語を5245bisと一致させた。
- End-of-Candidatesの表示に注文時の配送ルールを適用した。

### C.6.  draft-ietf-ice-trickle-15からの変更点

- ADレビューのフィードバックに対応するための調整。

### C.7.  draft-ietf-ice-trickle-14 からの変更点

- ICEコアへの変更を追跡するためのマイナーな修正。

### C.8.  draft-ietf-ice-trickle-13 からの変更点

- ICEコアで定義されているRunning状態にチェックリストを配置することで処理されるため、フリーズまたはアクティブのチェックリスト「状態」の独立した監視を削除した。

### C.9.  draft-ietf-ice-trickle-12 からの変更点

- end-of-candidates表示は、特定のICEセッションとの関連付けを可能にするための生成(ufrag/pwd)を含めなければならないことを指定した。
- WGLCのフィードバックに対応するためのさらなる編集修正。
 
### C.10.  draft-ietf-ice-trickle-11からの変更点

- WGLCからのフィードバックに対応するための編集と用語の修正。

### C.11.  draft-ietf-ice-trickle-10 からの変更点

- マイナーな編集上の修正。

### C.12.  draft-ietf-ice-trickle-09 からの変更点

- Fail時の即時unfreezeを削除しました。
- ice-optionsに関するMUST NOTの指定。
- 実装者の混乱を避けるために、初期ICEパラメータに関する用語を変更した。

### C.13.  draft-ietf-ice-trickle-08 からの変更点

- シグナリングプロトコルの要件として、メッセージのインオーダー処理に関するテキストを再掲。
- ICEオプションのためのIANA登録テンプレートを追加した。
- 通常のICEルールとの整合性を確保するために、セクション8.1.1のCase 3ルールを修正。
- 新しいペアルールを説明するために、セクション8.1.1に表形式の表現を追加した。

### C.14.  draft-ietf-ice-trickle-07 からの変更点

- 5245bis との整合性のため、"ICE の記述 "を "候補者情報 "に変更した。

### C.15.  draft-ietf-ice-trickle-06 からの変更点

- 委員長のレビューからの編集上のフィードバックに対応した。
- 世代に関する用語を明確にした。

### C.16.  draft-ietf-ice-trickle-05 からの変更点

- チェックリストに新しいペアを挿入する際のテキストを書き直しました。
 
### C.17.  draft-ietf-ice-trickle-04 からの変更点

- SDPおよびオファー/アンサーモデルへの依存関係を削除した。
- 5245bisで非推奨とされているため、攻撃的な指名に関する言及を削除した。
- シグナリングプロトコルの要件に関するセクションを追加した。
- 用語を明確にした。
- 様々なWGのフィードバックに対応した。

### C.18.  draft-ietf-ice-trickle-03 からの変更点

- unfreezeの動作の詳細な説明を提供した。具体的には、既存のpeer reflexive candidatesをトリクリング経由で受信したより優先度の高い候補に置き換える方法を示した。

### C.19.  draft-ietf-ice-trickle-02 からの変更点

- 基礎がバラバラな場合のunfreeze動作を調整しました。

### C.20.  draft-ietf-ice-trickle-01 からの変更点

- IPv6を使用するように例を変更しました。

### C.21.  draft-ietf-ice-trickle-00 からの変更点

- SDPへの依存関係を削除した(これは別の仕様で提供される)。
- 候補がまだ送受信されていない場合、チェックリストが空になる可能性があるという事実に関する文言を明確化した。
- チェックリストの状態に関する文言を明確化し、「Active」と「Frozen」の状態はICEコアではチェックリスト(候補者ペアのみ)に定義されていないため、新たな状態を定義しないようにした。
- 公開課題リストが古くなっていたので削除しました。
- 徹底したコピー編集を完了した。

### C.22.  draft-mmusic-trickle-ice-02 からの変更点

- Rajmohan BanaviおよびBrandon Williamsからのフィードバックに対応した。
- サポートの判断、および留守番電話がTrickle ICEをサポートしていないと判断された場合の対応方法についての文言を明確にしました。
- チェックリストとタイマーの更新に関するテキストを明確化しました。
- Half Trickleを使用することが適切な場合や、オファーやアンサーで候補者を送信しないことが適切な場合を明確にしました。
- 未解決問題のリストを更新しました。

### C.23.  draft-IVOV-01およびdraft-MMUSIC-00からの変更点

- unfreezeアルゴリズムのデッドロックを避けるために、コンポーネントの順番で候補をTrickleさせる要件を追加した。
- peer reflexive candidatesについて、意味的には何も変わらないが、Trickle ICEでは発生する可能性が高くなることを説明する有益なメモを追加した。
- 5245に準拠するためにペア数を100に制限した。
- 新たに発見された候補がどのようにしてリモートパーティにTrickle/送信されるか、あるいはそれが全く行われているかどうかは重要ではないことを明確にした。
- Dale Worleyの勧告に基づき、Trickled Candidateの輸送についての期待値を追加した。

### C.24.  draft-IVOV-00からの変更点

- end-of-candidatesはメディアレベルの属性であることを指定しました。 また、管理されたエージェントのための積極的な指名のようなケースでは、end-of-candidatesをオプションにした。
- ICE liteとTrickle ICEの例を追加し、ICE liteエージェントと話しているときに、どのようにして候補を送信したり、候補を発見したりする必要がないかを説明しました。
- ICE liteとTrickle ICEの例を追加し、ICE liteエージェントに話しかける際に、どのようにして候補を送信する必要がないのか、あるいはどのようにして候補を発見する必要がないのかを説明するために追加した。
- ICE liteエージェントはシグナリングを超えて候補を受信しないように準備しなければならず、これが起こってもパニックになってはいけないと明示的に述べている文言を追加した。 (対応するオープンイシューを閉じた)。
- 候補をTrickleするときにMIDを使用することが必須となり、m-lineインデックスを使用することは許可されなくなった。
- 0.0.0.0.0を保留中の操作と解釈するRFC2543 SDPライブラリとの潜在的な問題を避けるため、0.0.0.0.0の使用をIP6 ::に置き換えた。 また、ポート番号を1から9に変更した。これは、すでにより適切な意味を持つためである。 (Jonathan Lennoxが提案したポートの変更)。
- エンド・オブ・キャンドの後に受け取ったキャンドをどうするかについての使用に関するオープン・イシューを閉じました。 解決策: 無視して、何かを追加したい場合はICEの再起動をしてください。
- Trickle、Trickled Candidate、Half Trickle、Full Trickleを含む、より多くの用語を追加しました。
- ボストン中間会議で要求されたTrickle ICEのSIP使用法を追加した。

### C.25.  draft-rescorla-01からの変更点

- オファー/アンサーの明示的な使用を復活させた。 O/Aに依存しない方法でこれを行おうとする試みはもうありません。 また、ICE Descriptionsの使用も削除した。
- Trickled Candidate、Trickleオプション、m-lineの0.0.0.0.0アドレス、およびエンドオブキャンディデートの SDP仕様を追加した。
- サポートとディスカバリー。 このセクションを抽象度の低いものに変更した。   IETF85で議論されたように、この草案では、実装と使用方法は、事前にサポートを決定して直接 trickle を使用するか、あるいはHalf trickle を使用する必要があるとされています。 SIPでのディスカバリの使用についての提案や、実装プロトコルに好きなことをさせることについての提案を削除した。
- Half Trickleを定義した。 それがどのように動作するかというセクションを追加した。   それは最初のO/Aでのみ起こる必要があることに言及し(アップデートでは必要ない)、ユーザーが実際に呼び出しボタンを押す前に候補の一部またはすべてを事前に集めることができれば、どのようにして、いくつかのケースでは半分以上の改善を提供することができるかについてのJonathanのコメントを追加した。
- その後のオファー/アンサーの交換についての短いセクションを追加しました。
- ICE Lite実装との相互作用についての短いセクションを追加しました。
- 公開されている問題のセクションに2つの新しいエントリを追加しました。

### C.26.  draft-rescorla-00からの変更点

- MMUSICに関する議論を受けて、サポートの検証に関する要件を緩和した。
- 3264言語の曖昧な使用と、オファーとアンサーへの不適切な言及を削除するためにICEの記述を導入した。
- Martin Thomson氏が指摘したRTCWEBによる採用を前提とした不適切な記述を削除した。

## 著者のアドレス

Emil Ivov Atlassian 303 Colorado Street, #1600 Austin, TX  78701 USA
Phone: +1-512-640-3000 Email: eivov@atlassian.com

Eric Rescorla RTFM, Inc. 2064 Edgewood Drive Palo Alto, CA  94303 USA
Phone: +1 650 678 2350 Email: ekr@rtfm.com

Justin Uberti Google 747 6th St S Kirkland, WA  98033 USA
Phone: +1 857 288 8888 Email: justin@uberti.name

Peter Saint-Andre Mozilla P.O. Box 787 Parker, CO  80134 USA
Phone: +1 720 256 6756 Email: stpeter@mozilla.com URI:   https://www.mozilla.com/
