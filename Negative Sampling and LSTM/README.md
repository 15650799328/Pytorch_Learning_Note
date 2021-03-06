# Seq2Seq with Attention on Chinese Poetry Generation

## Example
```
柚子水
夏空日月明山色，
彼方美人不可爲。
天神上下不相知，
亂漫山中有水流。
魔女人間不可尋，
夜宴深溪上清風。
千戀里中無一事，
萬花頃刻一枝新。'
```

随机生成
```
洋亭 
江頭風雨不知秋，
一夜風聲不可留。
一夜風聲吹不盡，
一聲風雨一聲愁
```


## Model Structure 

The structure are modified from Pytorch tutorial on [Translation with a Sequence to Sequence Network and Attention](https://pytorch.org/tutorials/intermediate/seq2seq_translation_tutorial.html#evaluation). The attention model is simplified from _Chinese Poetry Generation with Planning based Neural Network_ by Wang, He, Wu, etc. 

I use a GRU in the encoder and a LSTM in the decoder, and I treat them as block boxes without modification. The output of GRU is used for the attention model for two reasons: looping over each sequence step in Pytorch is around 10x slower compared to the one step computation; GRU's output activation is tanh and its hidden state is a linear combination of previous output state.

The input data is a padded matrix: batch size x 100. Each row is a poetry and has the same format: ```title \<SOP> context \<EOP> (with punctuations)```, and the length is capped at 100 words. Shorter poetry are padded by 0 at the end.  

The runtime per batch is 10 minutes in Google's Nvidia P-100, or around 45 minutes in Google's Collaborator with Nvidia K-80. 
![Overview](Model.png)

* note: Dense layers use SELU activation. Dropout rates are 50%. Text rank's weights are replaced by cosine similarity from the word embedding.

## Problems
As shown in the examples, the quality of random poetry is very bad. The potential culprit is the encoder. The key words extracted by text rank don't make sense on the word level. A phrase level text rank might be helpful. I try jieba to split poetry phrases, but they don't look good. Another problem is in the attention model since I use the output only. I try to use Pytorch's LSTM Ceil to loop over each sequence step to get the hidden state, and I then create another Ceil to loop over the poetry with attention, 10 mini batch (2560 examples) takes 10x more time in Nvidia P-100. It takes around $5 to train a model, so I will not fix the problems at this stage. 

## Details

### Pre-process data
The data comes from [a github's repo](https://github.com/chinese-poetry/chinese-poetry). I use all ```poetry_tang_* ```and ```poetry_song_* ```, totaling 30K+ poetry. They are stored in a nested list, each sublist is a list of words in a poetry. The word dictionary is obtained from Gensim's Word2Vect modual. 

```python
# preprocess_data.py

patterns = ['（.*）', "{.*}", "《.*》", "\[.*\]", "<.*>", "）", "』", "：", "“.*”",
            '\[', '\」', '；', '》', '（', '）', '/', '`', '、', '：',
            '《', '\*', '-', '=', '{', '}']

def clean_sentence(sentence):
    for x in patterns:
        sentence = re.sub(x, '', sentence)
    return sentence.strip()


def split_poetry_2_list(poetry):
    # one entry in a json file
    # return  a flatten list of words
    text = poetry.get('paragraphs')  # may be []
    if text:
        text = [clean_sentence(x.strip()) for x in text]
        text = list(chain.from_iterable(text))  # flatten list of sentence
        text = ['<SOP>'] + text
        text[-1] = "<EOP>"

        title = poetry.get('title')
        title = "".join(title.split())
        title = clean_sentence(title)
        text = list(title) + text
    return text


def process_data(json_file):
    """
    :param json_file:
    :return: nested list of poetry
    """
    with open(json_file, 'rb') as f:
        data = json.load(f)
    poetry_text = []  # nested list
    word_set = set()
    for poetry in data:
        text = split_poetry_2_list(poetry)  # flatten list
        if text:
            word_set.update(text)
            poetry_text.append(text)
    return poetry_text, word_set
```

### Word2Vec
I code skip gram with negative sampling from scratch . The main model structure is two word embedding. Word2vec model can be viewed as a standard two-layer neural net. The hidden layer transforms the input one-hot word vectors into word embedding vectors, and the output layer is used to predict words during training only. Using two word embedding for the output layer might be faster than the linear dense layer (not tested) because Pytorch's matrix-matrix product (```bmm```) function is optimized. 

The input data are pre-processed at the poetry level, so the combination of a content word and a target word comes from the same poetry.

Training time is around 11 hours per epoch in Nvidia K-80 in Google's Collaborator and around 24 hours in CPU-only environment. 
 
```python
# model_negative_sampling.py

class SkipGramNegaSampling(nn.Module):
    def __init__(self, vocab_size, embed_dim):
        super(SkipGramNegaSampling, self).__init__()
        self.embed_hidden = nn.Embedding(vocab_size, embed_dim, sparse=True)
        self.embed_output = nn.Embedding(vocab_size, embed_dim, sparse=True)
        self.log_sigmoid = nn.LogSigmoid()

    def forward(self, input_batch, negative_batch):
        # input_batch (N x 2) [x, y]
        # negative_batch (N x k)
        x, y = input_batch
        embed_hidden = self.embed_hidden(x)  # N x 1 x D
        embed_target = self.embed_output(y)  # N x 1 x D
        embed_neg = -self.embed_output(negative_batch)  # N x k x D
        positive_score = embed_target.bmm(embed_hidden.transpose(1, 2)).squeeze(2)  # N x 1
        negative_score = embed_neg.bmm(embed_hidden.transpose(1, 2)).squeeze(2).sum(dim=1, keepdim=True)  # N x 1

        loss = self.log_sigmoid(positive_score) + self.log_sigmoid(negative_score)
        return -torch.mean(loss)
```

 But for efficient, I then use Gensim to generate the word embedding. I am not able to obtain the loss for each iteration. 
 If I loop over the model and manually decrease the learning rate, the loss from ```get_latest_training_loss``` differs significantly from the origin.  Hierarchical softmax is about 4-5x faster than negative sampling in Gensim.
 
 ```python
# word2vec_gensim.py

word2vec_params = {
    'sg': 1,  # 0 ： CBOW； 1 : skip-gram
    "size": 300,
    "alpha": 0.01,
    "min_alpha": 0.0005,
    'window': 10,
    'min_count': 1,
    'seed': 1,
    "workers": 6,
    "negative": 0,
    "hs": 1,  # 0: negative sampling, 1:Hierarchical  softmax
    'compute_loss': True,
    'iter': 50,
    'cbow_mean': 0,
}

with open('./data/poetry.json', 'rb') as f:
    sentences = json.load(f)

model = Word2Vec(**word2vec_params)
model.build_vocab(sentences)
trained_word_count, raw_word_count = model.train(sentences, compute_loss=True,
                                                 total_examples=model.corpus_count,
                                                 epochs=model.epochs)
```

### RNN models  
The three main building blocks are wrapped in ```PoetryRNN(*args)```. They are defined in the model graph above. 

```python
# model_rnn.py

PoetryEncoder(*args)

KeywordAttention(*args)

PoetryDecoder(*args)
```

### Generation

I use beam search algorithm for the LSTM output. The encoder and attention model output into the LSTM model and geenrate the first word. The subsequent inputs are previous words and hidden state. Words are selected by the highest scores from log softmax. Instead of choosing the single highest possible word in each step, the top N words (beam size) are selected and cached for the recurrent input/output. The final outcome is selected by the highest scores on the whole sentence .

```python
# generate.py

def beam_search_forward(model, cache, encoder_output, skip_words=non_words, max_trial=100, remove_punctuation=True):
    caches = []
    trial = 0
    sorted_lens = torch.LongTensor([1])
    for score, init, hidden in cache:
        # target_score, hidden_state = model.predict_softmax_score(init[-1], hidden)
        target_score, hidden_state = model.decoder.predict_softmax_score(init[-1], sorted_lens,
                                                                         encoder_output, hidden,
                                                                         model.attention)

        best_score, index = torch.max(target_score.squeeze(), dim=0)
        while remove_punctuation and index in skip_words and trial < max_trial:
            word = torch.LongTensor([index]).view(1, 1)
            target_score, hidden_state = model.decoder.predict_softmax_score(word, sorted_lens,
                                                                             encoder_output, hidden_state,
                                                                             model.attention)
            best_score, index = torch.max(target_score.squeeze(), dim=0)
            trial += 1
        chosen_word = torch.LongTensor([index]).view(1, 1)
        caches.append([best_score + score, init + [chosen_word], hidden_state])
    return caches


def beam_search(model, init_word, encoder_output, rnn_hidden,
                beam_size=3, text_length=100, skip_words=non_words, remove_punctuation=True):
    sorted_lens = torch.LongTensor([init_word.size(1)])
    target_score, hidden_state = model.decoder.predict_softmax_score(init_word, sorted_lens,
                                                                     encoder_output, rnn_hidden,
                                                                     model.attention)
    target_score = torch.sum(target_score, dim=0).squeeze()
    sorted_score, index = torch.sort(target_score, descending=True)

    cache = [[sorted_score[i],
              [torch.LongTensor([index[i]]).view(1, 1)],
              hidden_state] for i in range(beam_size)]

    for i in range(text_length):
        cache = beam_search_forward(model, cache, encoder_output, remove_punctuation=remove_punctuation)
    scores = [cache[i][0] for i in range(beam_size)]
    max_id = scores.index(max(scores))
    text = [i.item() for i in cache[max_id][1]]
    return text, cache[max_id][2]  # text, last_hidden_state
```
