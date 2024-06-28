[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/lychee1223/NovelAnalysisByNER/blob/main/ner.ipynb)
# BERTによる固有表現抽出を用いた小説の概要分析
### Set up
本リポジトリではpoetryでライブラリの管理を行っています．以下のコマンドを実行し，適宜インストールをお願いします．
```
curl -sSL https://install.python-poetry.org | python -
poetry install
```
Google Colabで実行する場合は以下のセルのコメントアウトを外し，適宜インストールをお願いします．
``` python
# Google Colab で実行する場合は以下のライブラリをインストール
# !pip install fugashi ipadic
```

# 1．目的
　小説において，登場人物や舞台はその内容を左右する重要な要素である．そして，一般的にこれらの要素は人名や地名などの固有表現であることが多い．
そこで，本実験ではBERTモデルのファインチューニングを行い固有表現抽出に適応させ，モデルの性能評価を行う．次に，ファインチューニングしたBERTモデルを用いて小説の固有表現抽出を行う．そして，固有表現抽出を用いた小説の概要分析の有用性について考察する．

# 2．分析に用いるデータセットおよびモデルの特徴
## 2.1．BERT
### 2.1.1．事前学習済みモデル
　本実験では東北大学の自然言語処理グループによって提供されているBERTモデル「cl-tohoku/bert-base-japanese-whole-word-masking」を用いる．このモデルはマスク言語モデリングを目標として，2019年9月1日時点のWikipediaの日本語記事のデータに対して全単語マスキングで学習されたものである．モデルのアーキテクチャはBERT BASEモデルと同様であり，12レイヤの総数は12，隠れ層は768次元，Multi-Head Attentionのヘッドは12個である．
このモデルで提供されているトークナイザはトークナイザクラスBertJapaneseTokenizerを用いて読み込むことが可能である．このトークナイザではIPA辞書を用いたMeCab形態素パーサによるトークン化，WordPieceアルゴリズムによるサブワード分割の2ステップで文章のトークン化を行う．語彙のサイズは32000である．また，トークン列の先頭に‘[CLS]’，トークン列の末尾に‘[SEP]’，未知語に対して‘[UNK]’，系列長を揃えるためのパディングとして‘[PAD]’といった特殊トークンを付加する．そして，このトークナイザを用いて符号化することで，BERTモデルに入力可能なPyTorchが提供するtorch.Tensor型としてトークン化が可能である．符号化したデータはキー“input_ids”，“token_type_ids”，“attention_mask”からなる辞書型として与えられる．“input_ids”は各トークンに対するIDのテンソル，“token_type_ids”は2つの文章を入力した際に各文章を区別するためのテンソル，“attention_mask”はどのトークンにアテンションをかけるかを示すテンソルを値として持つ．

### 2.1.2．トークンごとの分類を行うクラス
　本実験ではBERTモデル「cl-tohoku/bert-base-japanese-whole-word-masking」を分類問題に対して適合させるため，BertForTokenClassificationクラスを用いる．このクラスはトークンごとのラベルの分類スコアを出力とする．損失関数としてクロスエントロピー誤差が用いられている．

## 2.2．モデルのファインチューニングに用いるデータセット
　本実験ではモデルのファインチューニングに用いる学習データ，検証データ，テストデータとして「ner-wikipedia-dataset」を用いる．このデータセットはWikipediaから抽出された文章に含まれる固有表現にラベル付けされたものであり，ストックマーク株式会社によって作成された．データ数は5343であり，人名，法人名，政治的組織名，その他の組織名，地名，施設名，製品名，イベント名の8種類のタイプの固有表現がラベル付けされている．ただし，固有表現を含まない負例の文章のデータが約10%程度含まれている．表1に各タイプの固有表現数を示す．

<div align="center">

表1　各タイプの固有表現数
| タイプ | 固有表現数 |
| --- | ---: |
| 人名 | 2980 |
| 法人名 | 2485 |
| 政治的組織名 | 1180 |
| その他の組織名 | 1051 |
| 地名 | 2157 |
| 施設名 | 1108 |
| 製品名 | 1215 |
| イベント名 | 1009 |

</div>

　また，このデータセットはjson形式で提供され，各データは以下のような辞書形式で示される．
```
	{
		"curid": "473536",
		"text": "イギリスはリーマンショック直後の2008年10月にイングランド銀行のバランスシートを一気に3倍近く増やした後、2008年11月から2009年3月にかけて段階的に縮小させていった。",
		"entities": [
			{
				"name": "イギリス",
				"span": [0,4],
				"type": "地名"
			},
			{
				"name": "リーマンショック",
				"span": [5,13],
				"type": "イベント名"
			},
			{
				"name": "イングランド銀行",
				"span": [25,33],
				"type": "政治的組織名"
			}
		]
	}
```
　キー“curid”はデータ元のWikipediaのページID，キー“text”はWikipediaから抽出した対象の文章，キー“entities”は文章に含まれる固有表現のリストを値として持つ．“entities”の各要素のキー“name”は固有表現名，キー“span”は文章中の位置，キー“type”は固有表現のタイプを示す．

## 2.3．分析対象の小説のデータセット
　本実験では固有表現抽出を行う分析対象の小説として青空文庫で提供されている宮沢賢治の『銀河鉄道の夜』のルビありテキストファイルを用いる．青空文庫は著作権の保護期間を過ぎた作品の電子化，書き手自身が対価を求めないと決めた作品を慕うというモチベーションの基，ボランティアにより言語による著作物のデジタル化，共有財産化を行う活動である．本実験で用いたデータセットは‘|’と‘《》’を用いてルビを表記し，‘[]’を用いて入力者注を記述するといった一定の形式に従っている．

# 3．分析手順
## 3.1．前処理および準備
1. 学習用データセット「ner-wikipedia-dataset」を読み込んだ．
1. データセットの文章を正規化した．
1. データセットの各固有表現のタイプを値として持つキー“type”を削除し，代替として各タイプに対応するIDを値として持つキー“type_id”を追加した．各タイプに対応するIDは人名を1，法人名を2，政治的組織名を3，その他の組織名を4，地名を5，施設名を6，製品名を7，イベント名を8と定めた．
1. データセットを6:2:2の比率で学習データ，検証データ，テストデータに分割した．
1. 事前学習済みモデル「cl-tohoku/bert-base-japanese-whole-word-masking」の学習済みトークナイザを読み込んだ．
1. トークナイザクラスBertJapaneseTokenizerを拡張し，固有表現抽出に適応したトークナイザを作成した．このトークナイザを用いた符号化では，キー“input_ids”や“attension_mask”に加え，各トークンのラベルを格納したテンソルを値として持つキー“labels”を付加する．各トークンのラベルとして固有表現中のトークンに対してID，その他のトークンに対しては0を付与する．表2にラベル付けの例を示す．

<div align="center">

表2　トークンに対するラベル付与の例
| トークン | ‘[CLS] ’ | ‘2019’ | ‘年’ | ‘、’ | ‘宗像’ | ‘サ’ | ‘ニックス’ | ‘ブルース’ | ‘に’ | ‘加入’ | ‘。’ | ‘[SEP] ’ |
| --- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| ラベル |0 | 0 | 0 | 0 | 4 | 4 | 4 | 4 | 0 | 0 | 0 | 0 |

</div>

7. 学習データおよび検証データについて，(6)で作成したトークナイザを持つデータローダおよびデータセットを作成した．系列長は128，学習データのバッチサイズは16，検証データのバッチサイズは256とした．
 
## 3.2．事前学習済みモデルによる固有表現抽出
1. BertForTokenClassificationクラスを用い，事前学習済みモデル「cl-tohoku/bert-base-japanese-whole-word-masking」を読み込んだ．
1. 事前学習済みモデルを用い，テストデータの固有表現抽出を行った．ここで，各トークンの予測値として分類スコアが最も高いラベルを採用した．そして，予測値が同一の連続するトークンを1つにまとめ，予測値が0以外であればその表現を固有表現とした．予測された固有表現はデータセット「ner-wikipedia-dataset」のキー“entities”の形式で文章中の位置，固有表現のタイプと共にリストに格納した．
1. ランダムに3つのデータを選出し，推論結果を確認した．
1. 推論全体および各固有表現のタイプについて，評価指標として再現率，適合率，F値を導出した．ここで，抽出した固有表現はキー“name”， “span”，“type_id”が全て一致した場合のみ正解とした．

## 3.3．ファインチューニングしたモデルによる固有表現抽出
1. 事前学習済みモデルのファインチューニングを行った．最適化器はAdam，エポック数は5，学習率は2e^(-5)とした．また，学習はバックプロパゲーションで行った．
1. ファインチューニングしたモデルを用い，テストデータの固有表現抽出を行った．
1. ランダムに3つのデータを選出し，推論結果を確認した．
1. 推論全体および各固有表現のタイプについて，評価指標として再現率，適合率，F値を導出した．

## 3.4．『銀河鉄道の夜』の固有表現抽出
1. 『銀河鉄道の夜』のルビありテキストファイルを読み込んだ．
1. データセットの文章からヘッダ，フッタ，ルビ，入力者注を削除し，正規化した．
1. ファインチューニングしたモデルを用いて固有表現抽出を行った．
1. 各固有表現のタイプごとに推論結果を確認した．
 
# 4．結果
## 4.1．ファインチューニングの結果
　図1にファインチューニングを行った際の学習の進捗に伴う学習データと検証データの損失の推移を示す．

<div align="center">
  
図1　損失に関する学習曲線
![learning curve](https://github.com/lychee1223/NovelAnalysisByNER/assets/110546438/3ef51fc9-7954-444d-81ee-c0ada8ccba13)

</div>

　1エポック目から5エポックにかけて，学習データの損失は0.2147から0.0425まで0.1722低下した．一方，検証データの損失は0.1705から0.0009まで0.1696低下した．
 
## 4.2．ファインチューニング前後のモデルによる予測結果
### 4.2.1．テストデータの固有表現抽出結果
　事前学習済みモデルを用いた固有表現抽出を行った際，推論結果を確認したデータの正解の固有表現と予測された固有表現を表3に示す．また，ファインチューニングしたモデルを用いた固有表現抽出を行った際，推論結果を確認したデータの正解の固有表現と予測された固有表現を表4に示す．

<div align="center">
  
表3　事前学習済みモデルによる固有表現抽出結果
<table>
<tr>
  <th rowspan=2>index</th>
  <th colspan=3>正解の固有表現</th>
  <th colspan=3>予測された固有表現</th>
</tr>
<tr>
  <th>固有表現</td>
  <th>スパン</td>
  <th>タイプ</td>
  <th>固有表現</td>
  <th>スパン</td>
  <th>タイプ</td>
</tr>
<tr>
  <td>499</td>
  <td>‘FC琉球’</td>
  <td>[ 2, 6]</td>
  <td>その他の組織名</td>
  <td>‘で’</td>
  <td>[17,18]</td>
  <td>地名</td>
</tr>
<tr>
  <td rowspan=2>882</td>
  <td>‘土家由岐雄’</td>
  <td>[ 0, 5]</td>
  <td>人名</td>
  <td>‘、’</td>
  <td>[ 6, 7]</td>
  <td>地名</td>
</tr>
<tr>
  <td>‘日本’</td>
  <td>[ 7, 9]</td>
  <td>地名</td>
  <td>‘児童文学作家’</td>
  <td>[10,17]</td>
  <td>地名</td>
</tr>
<tr>
  <td rowspan=2>493</td>
  <td>‘鴇田正憲’</td>
  <td>[ 0, 4]</td>
  <td>人名</td>
  <td>‘兵庫’</td>
  <td>[ 6, 8]</td>
  <td>地名</td>
</tr>
<tr>
  <td>‘兵庫県神戸市’</td>
  <td>[ 6,12]</td>
  <td>地名</td>
  <td>‘サッカー選手、経営者’</td>
  <td>[15,25]</td>
  <td>地名</td>
</tr>
</table>

表4　ファインチューニングしたモデルによる固有表現抽出結果
<table>
<tr>
  <th rowspan=2>index</th>
  <th colspan=3>正解の固有表現</th>
  <th colspan=3>予測された固有表現</th>
</tr>
<tr>
  <th>固有表現</td>
  <th>スパン</td>
  <th>タイプ</td>
  <th>固有表現</td>
  <th>スパン</td>
  <th>タイプ</td>
</tr>
<tr>
  <td rowspan=2>938</td>
  <td>‘テイチク’</td>
  <td>[ 6, 10]</td>
  <td>企業名</td>
  <td>‘テイチク’</td>
  <td>[ 6, 10]</td>
  <td>企業名</td>
</tr>
<tr>
  <td>‘役者’</td>
  <td>[20, 22]</td>
  <td>製品名</td>
  <td>‘役者’</td>
  <td>[20, 22]</td>
  <td>製品名</td>
</tr>
<tr>
  <td rowspan=2>886</td>
  <td>‘京成電鉄’</td>
  <td>[ 3, 7]</td>
  <td>企業名</td>
  <td>‘京成電鉄’</td>
  <td>[ 3, 7]</td>
  <td>企業名</td>
</tr>
<tr>
  <td>‘茨城’</td>
  <td>[34,36]</td>
  <td>地名</td>
  <td>‘茨城県内’</td>
  <td>[34,38]</td>
  <td>地名</td>
</tr>
<tr>
  <td>953</td>
  <td>‘ラフーン’</td>
  <td>[ 0, 4]</td>
  <td>人名</td>
  <td>‘ラフーン’</td>
  <td>[ 0, 4]</td>
  <td>人名</td>
</tr>
</table>

</div>

　事前学習済みモデルによる固有表現抽出において，ほとんど正解の固有表現を抽出することはできず，でたらめな表現が抽出された．<br/>
　一方，ファインチューニングしたモデルによる固有表現抽出において，ほとんどの固有表現を正しく抽出できた．地名である‘茨城’は正しく抽出されず，‘茨城県内’が地名として抽出された．
 
### 4.2.2．テストデータの固有表現抽出における性能評価結果
　事前学習のモデルを用いた固有表現抽出における，各固有表現のタイプの性能評価指標を表5に示す．また，ファインチューニングしたモデルを用いた固有表現抽出における，各固有表現のタイプの性能評価指標を表6に示す．ここで，正解した予測の数が0の場合，F値は計算不可能であるため‘-’として示す．

<div align="center">

表5　ファインチューニング前のモデルによる固有表現抽出の性能評価結果
| 人名 | 法人名 | 政治的組織名 | その他の組織名 | 地名 | 施設名 | 製品名 | イベント名 | ALL |
| --- | ---: |  ---: |  ---: |  ---: |  ---: |  ---: |  ---: |  ---: | 
| 正解固有表現の数 | 604 | 504 | 249 | 222 | 452 | 222 | 231 | 203 | 2687 |
| 予測固有表現の数 | 60 | 244 | 2005 | 1420 | 3634 | 244 | 177 | 1245 | 9029 |
| 正解した予測の数 | 0 | 0 | 2 | 0 | 19 | 0 | 0 | 1 | 22 |
| 適合率 | 0.0000 | 0.0000 | 0.0010 | 0.0000 | 0.0052 | 0.0000 | 0.0000 | 0.0008 | 0.0024 |
| 再現率 | 0.0000 | 0.0000 | 0.0080 | 0.0000 | 0.0420 | 0.0000 | 0.0000 | 0.0049 | 0.0082 |
| F値 | - | - | 0.0018 | - | 0.0093 | - | - | 0.0014 | 0.0038 |

表6　ファインチューニング後のモデルによる固有表現抽出の性能評価結果
| 人名 | 法人名 | 政治的組織名 | その他の組織名 | 地名 | 施設名 | 製品名 | イベント名 | ALL |
| --- | ---: |  ---: |  ---: |  ---: |  ---: |  ---: |  ---: |  ---: | 
| 正解固有表現の数 | 604 | 504 | 249 | 222 | 452 | 222 | 231 | 203 | 2687 |
| 予測固有表現の数 | 608 | 472 | 239 | 228 | 485 | 259 | 226 | 210 | 2727 |
| 正解した予測の数 | 553 | 412 | 164 | 177 | 391 | 170 | 156 | 170 | 2193 |
| 適合率 | 0.9095 | 0.8729 | 0.6862 | 0.7763 | 0.8062 | 0.6564 | 0.6903 | 0.8095 | 0.8042 |
| 再現率 | 0.9156 | 0.817 | 0.6586 | 0.7973 | 0.8650 | 0.7658 | 0.6753 | 0.8374 | 0.8162 |
| F値 | 0.9125 | 0.8443 | 0.6721 | 0.7867 | 0.8346 | 0.7069 | 0.6827 | 0.8232 | 0.8101 |

</div>

　事前学習済みモデルによる固有表現抽出において，推論全体の適合率は0.0024，再現率は0.0082，F値は0.0038であった．各固有表現のタイプの結果に注目すると，人名，法人名，その他の組織名，施設名，製品名は予測が1つも正解しなかった．<br/>
　一方，ファインチューニングしたモデルによる固有表現抽出において，推論全体の適合率は0.8042，再現率は0.8162，F値は0.8101であった．各固有表現のタイプの結果に注目すると，人名，法人名，地名のF値が他のタイプと比較して大きかった．

## 4.3．『銀河鉄道の夜』の固有表現抽出結果
　表7に『銀河鉄道の夜』の文章をファインチューニングしたモデルに入力し，各タイプの固有表現抽出を行った結果を示す．

<div align="center">
 
表7　『銀河鉄道の夜』から抽出した固有表現
| タイプ | 抽出された固有表現 |
| --- | ---|
| 人名 | ‘ジョバンニ’ ，‘ジョバン’ ，‘カムパネルラ’ ，‘カムパネル’ ，‘きくよ’ ，‘かおる’ ，‘かおる子’ ，‘かお’ ，‘ザウエル’ ，‘ザネリ’ ，‘マルソ’ ，‘カトウ’ ，‘鳥捕り’ ，‘インデアン’ ，‘大学士’ ，‘燈’ ，‘虫めがね’ ，‘コムパス’ ，‘わっし’ ，‘鷺’ ，‘孔雀’ ，‘鶴’ ，‘鶴や雁’ ，‘さぎも白鳥’ ，‘なんべん’ ，‘向う’ ，‘ひる’ ，‘あすこ’ ，‘サウ’ ，‘のり’ ，‘お宮’ |
| 法人名 | ‘銀河鉄道’ ，‘牛乳’ |
| 政治的組織名 | ‘空’ ，‘工兵大隊’ |
| その他の組織名 | ‘新世界交響楽’ |
| 地名 | ‘白鳥区’ ，‘白鳥の島’ ，‘天の川’ ，‘南十字’ ，‘サウザンクロス’ ，‘ザンクロス’ ，‘アルビレオ’ ，‘コロラド’ ，‘ランカシャイヤ’ ，‘コンネクテカット州’ ，‘バルドラ’ ，‘森’ ，‘ケンタウル’ ，‘本国’ ，‘北’，‘方’ ，‘あすこ’ ，‘うし’ ，‘カムパネル’ |
| 施設名 | ‘白鳥停車場’ ，‘白鳥’ ，‘銀河ステーション’ ，‘六、銀河ステーション’ ，‘北十字’ ，‘プリオシン海岸’ ，‘青い橄欖の’ ，‘ひる学校’ |
| 製品名 | ‘露’ ，‘大熊’ ，‘あなたの神さまうその神さまよ。’ ，‘いいえ。’ ，‘ごとごとごとごと汽車’ ，‘いま海へ行ってらあ。’ ，‘ハルレヤハルレヤ。’ ，‘さよなら’ ，‘の停車場’ ，‘いいえ’ ，‘烏’ ，‘どら、’ ，‘ええ河’，‘鷺’ ，‘ケンタウル露’ ，‘神さまうその神さまだい。’ ，‘ケンタウルス’ ，‘ねえ’ ，‘ああ。’ ，‘ハルレヤ、ハルレヤ’ ，‘ぼくたべよう。’ ，‘せ’ ，‘僕わからない’ ，‘そうそう’ ，‘降りよう。’ ，‘雀’ ，‘ザネリ’ ，‘ま’，‘はなしてごらん’ ，‘わっし’ ，‘さよなら。’ ，‘だい’ ，‘鳥です’ ，‘あ孔雀’ ，‘カムパネルラ’ |
| イベント名 | ‘星祭’ ，‘ケンタウル祭’ ，‘こんや’ ，‘銀河の’ |

</div>

　人名の固有表現について，『銀河鉄道の夜』の登場人物である‘ジョバンニ’，‘カムパネルラ’，‘きくよ’，‘かおる’，‘ザウエル’ ，‘ザネリ’ ，‘マルソ’ ，‘カトウ’ ，‘鳥捕り’ ，‘インデアン’ ，‘大学士’ が正しく抽出された．また，‘ジョバン’など，登場人物の一部の表現も抽出された．その他，あだ名や代名詞，鳥の名前などが抽出された．<br/>
　法人名，政治的組織名，その他の組織名の固有表現について，『銀河鉄道の夜』に登場する組織である‘銀河鉄道’が正しく抽出された．また，‘工兵大隊’など，登場する組織の一部の表現も抽出された．<br/>
　地名，施設名，イベント名の固有表現について，『銀河鉄道の夜』に登場する場所や舞台となる‘白鳥区’，‘天の川’，‘南十字’，‘アルビレオ’ ，‘コロラド’ ，‘ランカシャイヤ’ ，‘コンネクテカット州’ ，‘バルドラ’，‘白鳥停車場’，‘銀河ステーション’，‘北十字’，‘プリシオン海岸’，‘星祭’，‘ケンタウル祭’が正しく抽出された．<br/>
　製品名の固有表現について，製品名でない文章や人名などの表現が多く抽出された．<br/>

# 5．考察
## 5.1．モデルの評価
### 5.1.1．固有表現抽出に用いるモデルとしての妥当性
　事前学習済みモデルによる推論全体のF値は0.0038と極めて小さく，抽出された表現はでたらめであった．また，一部のタイプの固有表現では正解した予測が0であった．これは事前学習済みモデルが固有表現抽出を目的に学習されたモデルでないためと考えられる．<br/>
　一方，ファインチューニングしたモデルによる推論全体のF値は0.8101であり，事前学習済みモデルの結果より0.8063上昇した．したがって，ファインチューニングによりモデルが固有表現抽出に適合したと考えられる．また，図1の学習曲線より，学習データの損失の低下に伴い検証データの損失も低下した．よって，過学習は生じていないと考えられる．以上より，本実験でファインチューニングしたモデルは固有表現抽出に用いるモデルとして妥当であると考察できる．

### 5.1.2．ファインチューニングしたモデルの性能向上における課題
　ファインチューニングしたモデルによる人名，法人名，地名の固有表現の推論におけるF値が他のタイプと比較して大きかった．表1より，これらのタイプはデータセットに含まれる固有表現数が多く，学習データとなる固有表現も同様に多いと考えられる．したがって，学習データ数の増加に伴いモデルの精度が向上すると考えられる．ゆえに，よりモデルの精度を向上させるためには学習データ数を多くする必要があると考察できる．

### 5.1.3．モデルの性能評価の妥当性
　本実験では抽出した固有表現のキー“name”， “span”，“type_id”が全て一致した場合のみ正解としている．よって，実際には適切な固有表現を抽出できていた場合でも不正解になりうる．具体的な例として，連続する2つの固有表現，‘東京都’，‘小金井市’が1つの固有表現‘東京都小金井市’と抽出された場合や，‘東京都’が‘東京’と抽出された場合は不正解となる．したがって，より正確にモデルを評価するためには正解の固有表現と予測された固有表現間の類似性を評価するべきであると考察できる．性能評価の改善を今後の課題としたい．

## 5.2．固有表現抽出による小説の要約の妥当性
　人名の固有表現について，‘ジョバンニ’や‘カムパネルラ’をはじめ，『銀河鉄道の夜』の登場人物を概ね正しく抽出できた．一方，一般名詞である鳥の名前が誤って人名として多く抽出された．人名と鳥の名前は共に生物を示す名詞として働く．ゆえに，推論を誤りやすかったのではないかと考えられる．また，全く人名と関係のない‘なんべん’，‘向う’，‘ひる’などの表現も誤って人名として抽出された．これらの表現はそれぞれ‘なんべんもお母さんから聴いたわ’，‘向うの坊ちゃん’，‘ひる先生の云ったような’という文章中の表現である．これらの表現は漢字に変換されていないことや現代と送り仮名が異なるといった理由で人名として勘違いしうる表現である．ゆえに，推論を誤りやすかったのではないかと考えられる． 　しかし，人名の固有表現に関しては大きな誤りはなく，概ね小説に対しても適切に抽出されたと言える．よって，小説の登場人物の抽出として固有表現抽出を利用することは妥当であると考察できる．<br/>
　また，法人名，政治的組織名，その他の組織名の固有表現について，‘牛乳屋’が‘牛乳’，‘空の工兵大隊’が‘空’と‘工兵大隊’，‘新世界交響楽団’が‘新世界交響楽’として抽出された．また，法人名として抽出された‘銀河鉄道’が法人団体であるかは不明である．しかし，『銀河鉄道の夜』に登場する組織としては概ね正しく抽出できたと言える．よって，小説に登場する組織の抽出として固有表現抽出を利用することは妥当であると考察できる．<br/>
　そして，地名，施設名，イベント名の固有表現について，‘天の川’や‘銀河ステーション’，‘ケンタウル祭’をはじめ，『銀河鉄道の夜』に登場する場所や舞台が概ね正しく抽出できた．よって，小説の舞台の抽出として固有表現抽出を利用することは妥当であると考察できる．<br/>
　しかし，製品名の固有表現については全く正しく抽出できなかった．一般的に製品名となる表現は文章や一般名詞，一般的でない創作の単語など多種多様である．したがって，学習データに含まれる固有表現のうち，製品名とラベル付けされた表現は多種多様でその共通する特徴を見つけるのは難しいと考えられる．ゆえに，推論を誤りやすかったのではないかと考えられる．よって，小説に登場する製品名の抽出として固有表現抽出を利用することは妥当でないと考察できる．<br/>
　以上より，小説の固有表現を抽出することで登場人物や舞台を抽出することができると考えられる．したがって，固有表現抽出を小説の様々な分析や概要の推察に用いることが可能であると考察できる．<br/>

# 6．参考文献
1. 近江崇宏，金田健太郎，森長誠，江間見亜利．BERTによる自然言語処理入門 -Transformersを使った実践プログラミング-，オーム社，pp.105-142，2021．
1. 近江崇宏．Wikipediaを用いた日本語の固有表現抽出のデータセットの構築，言語処理学会第27回年次大会発表論文集，pp.350-352，2021．https://anlp.jp/proceedings/annual_meeting/2021/pdf_dir/P2-7.pdf
1. 東北大学，自然言語処理グループ．cl-tohoku/bert-base-japanese-whole-word-masking．https://huggingface.co/cl-tohoku/bert-base-japanese-whole-word-masking
1. ストックマーク株式会社．stockmarkteam/ner-wikipedia-dataset．https://github.com/stockmarkteam/ner-wikipedia-dataset
1. 宮沢賢治．『銀河鉄道の夜』，新潮社，1989．青空文庫．https://www.aozora.gr.jp/cards/000081/card456.html

