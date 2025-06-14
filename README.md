# embedding_sample
embeddingについて調べてlocalで動かしてみる

# そもそもembeddingとは？
言葉やデータをベクトルで表現する方法
猫とか犬といった単語をその意味を反映した数値ベクトルに変換する。これで機会が単語同士の意味的な距離や関係を数値的に扱える.
単語を点として捉えず、機械でも単語を線として捉えることができるのか

チャットbotとか、生成aiの入力のプロンプトが人間のように解析して読み取れるのか。

# embeddingの例
| 技術名                                  | 特徴                          |
| ------------------------------------ | --------------------------- |
| **Word2Vec**                         | 単語の意味を周囲の単語から学ぶ（Google）     |
| **GloVe**                            | 全体の共起行列から意味を学ぶ（Stanford）    |
| **FastText**                         | サブワード情報も使って未知語に強い（Facebook） |
| **BERT Embedding**                   | 文脈を考慮した動的なベクトル表現（Google）    |
| **Sentence-BERT / OpenAI Embedding** | 文単位の類似度計算に強い、RAGや検索などにも使われる |

# 割と軽量で試せるsentence-transformersで試してみる

```
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np

# 文リスト
corpus = [
    "パスワードを忘れました",
    "ログインできません",
    "アカウントを削除したいです",
    "新しい機能の使い方を知りたい",
    "支払い方法を変更したい"
]

# モデルロード（軽量）
model = SentenceTransformer('all-MiniLM-L6-v2')

# ベクトル化
corpus_embeddings = model.encode(corpus)

# FAISSでインデックス作成
index = faiss.IndexFlatL2(corpus_embeddings[0].shape[0])
index.add(np.array(corpus_embeddings))

# クエリ例
query = "サブスクリプションをキャンセルしたい"
query_vec = model.encode([query])

# 検索
top_k = 2
D, I = index.search(np.array(query_vec), top_k)
for i in range(top_k):
    print(f"{i+1}. {corpus[I[0][i]]}（距離: {D[0][i]:.2f}）")

```

## 自分理解用にメモ


### やりたいこと
ユーザー入力されものからあらかじめ用意した中から意味的に近いものを見つけ出す

### ベクトルのインデックス化
```
faiss.IndexFlatL2(corpus_embeddings[0].shape[0])
```
FAISSライブラリにベクトルの「次元数」を渡すためにcorpus_embeddings[0]を使っている
corpus_embeddingsは複数の文をエンコードした2次元のnumpy配列になっているので、次元数をFAISSに渡してあげている
arr.shape → (2, 3)（タプル）
arr.shape[0] → 2（タプルの最初の値＝行数）
arr.shape[1] → 3（タプルの2つ目の値＝列数）

### IndexFlatL2とは？
ベクトルの類似検索用のインデックスのこと

つまり、分や単語をベクトル化してこの文に一番似ている文はどれか？って検索したい時に使うらしい

Index=ベクトル検索のためのインデックスを作るクラス
Flat=フラットな構造（すべてのベクトルをそのまま並べて比較する）
L2=類似度の計算方法に「ユークリッド距離」を使う

他にもこんなのがあるらしい
| インデックス          | 構造 | 精度 | 検索速度 | 特徴                       |
| --------------- | -- | -- | ---- | ------------------------ |
| `IndexFlatL2`   | なし | ◎  | △    | 最もシンプル。全件比較で正確だが遅め。      |
| `IndexIVFFlat`  | あり | ◯  | ◯～◎  | クラスタ構造あり。高速だけどトレーニングが必要。 |
| `IndexHNSWFlat` | あり | ◎  | ◎    | 高速で精度も高いが構築が複雑。          |
| `IndexPQ`       | 圧縮 | △  | ◎    | メモリ効率◎だが精度は落ちる。          |

ポータルとかのFAQとかでこんな類似のものを表示させてあげているのか！
・ユーザーが入力する
・入力値からベクトル文章の類似検索
・予め用意した回答or類似の回答を生成？してユーザーにレスポンスする

## 実際にembeddingしているのはmode.encode()!!
簡単にいうとmodelにて文章を数字に変換している

### ステップ1　言葉を分ける（トークナイズ）
形態素解析みたいなことをしている
```
「私」「は」「りんご」「を」「食べた」
```

### ステップ2　数字に変える（IDに変換
```
「私」→ 101  
「りんご」→ 202  
「食べた」→ 303 
```

### ステップ3　トランスフォーマーによるエンコーディング
ID列をトランスフォーマが受け取り、各トークンに対してベクトル表現を計算する。つまり1つの分がトークン数*次元数の2次元配列として表現される

```
[0.2, -0.1, 0.6, ..., 0.3]  ← 長さ384のベクトル（＝384個の数字）
```

### ステップ4　プーリング(分全体を1つのベクトルにする)
分の各トークンベクトルを集約して、文全体の特徴をもつ1つのベクトルに変換する。これがsentenceEmbedding

### ステップ5　384次元のベクトル（numpy配列）で返却する
これが意味を持つ座標として返却される

## トランスフォーマーとは？
文の中の言葉同士の関係をうまく掴む仕組み

|トランスフォーマーは2017年にGoogleが発表した「Attention is All You Need」という論文で提案された、自然言語処理（NLP）のための画期的なアーキテクチャです。現在のChatGPTやBERTなどの大規模言語モデルの基盤となっています。

- セルフアテンション機構
分の中の単語が他の単語とどのように関連しているか計算

- 並列処理が可能
従来のRNNと異なり、分全体を同時に処理可能

- 長距離依存関係の把握
分の最初と最後の単語の関係性も直接計算可能






