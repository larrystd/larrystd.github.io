---
title: tensorflow
date: 2021-07-18
updated: 2021-07-18
tags: 
  - python
  - machine learning
categories:
  - math science
aplayer: true
---


### tensorflow2.0

#### 基本知识

创建和操作Tensor

```py
import tensorflow as tf 
print (tf.__version__)



x = tf.constant(range(12))
print(x.shape)

# (12,)

x

<tf.Tensor: id=1, shape=(12,), dtype=int32, numpy=array([ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11], dtype=int32)>

X = tf.reshape(x,(3,4))
<tf.Tensor: id=2, shape=(3, 4), dtype=int32, numpy=
array([[ 0,  1,  2,  3],
       [ 4,  5,  6,  7],
       [ 8,  9, 10, 11]])>


tf.zeros((2,3,4))

Y = tf.constant([[2,1,4,3],[1,2,3,4],[4,3,2,1]])

# 正态分布
tf.random.normal(shape=[3,4], mean=0, stddev=1)

# 按元素乘法：
X * Y
# 按元素指数运算
Y = tf.cast(Y, tf.float32)
tf.exp(Y)


# 矩阵乘法
Y = tf.cast(Y, tf.int32)
tf.matmul(X, tf.transpose(Y))

# 按列连接
tf.concat([X,Y],axis = 0)
# 按行连接
tf.concat([X,Y],axis = 1)

# 对tensor中的所有元素求和得到只有一个元素的tensor。
tf.reduce_sum(X)

# 范数
X = tf.cast(X, tf.float32)
tf.norm(X)

# 支持切片

X[1:3]

X = tf.Variable(X)

<tf.Variable 'Variable:0' shape=(3, 4) dtype=float32, numpy=
array([[ 0.,  1.,  2.,  3.],
       [ 4.,  5.,  9.,  7.],
       [ 8.,  9., 10., 11.]], dtype=float32)>

X[1,2].assign(9) # 重新赋值

# 即使像Y = X + Y这样的运算，我们也会新开内存，然后将Y指向新内存。

X = tf.Variable(X)
Y = tf.cast(Y, dtype=tf.float32)

before = id(Y)
Y = Y + X
id(Y) == before

False
```

<!-- more -->

与numpy转换
```
import numpy as np

P = np.ones((2,3))
D = tf.constant(P)

np.array(D)
```

tensor与numpy转换 
```py
import numpy as np
p = np.ones((2,3))
d = torch.from_numpy(p)

d.numpy()
```

自动求梯度

tensorflow2.0提供的GradientTape来自动求梯度。
```py
x = tf.reshape(tf.Variable(range(4), dtype=tf.float32),(4,1))
x

<tf.Tensor: id=10, shape=(4, 1), dtype=float32, numpy=
array([[0.],
       [1.],
       [2.],
       [3.]], dtype=float32)>

with tf.GradientTape() as t:
    t.watch(x)
    y = 2 * tf.matmul(tf.transpose(x), x)
    
dy_dx = t.gradient(y, x)
dy_dx

<tf.Tensor: id=30, shape=(4, 1), dtype=float32, numpy=
array([[ 0.],
       [ 4.],
       [ 8.],
       [12.]], dtype=float32)>


def f(a):
    b = a * 2
    while tf.norm(b) < 1000:
        b = b * 2
    if tf.reduce_sum(b) > 0:
        c = b
    else:
        c = 100 * b
    return c
a = tf.random.normal((1,1),dtype=tf.float32)
with tf.GradientTape() as t:
    t.watch(a)
    c = f(a)
t.gradient(c,a) == c/a

<tf.Tensor: id=201, shape=(1, 1), dtype=bool, numpy=array([[ True]])>
```

#### 线性回归

```py
import tensorflow as tf
print(tf.__version__)
from matplotlib import pyplot as plt
import random

# 生成数据集
num_inputs = 2
num_examples = 1000
true_w = [2, -3.4]
true_b = 4.2
features = tf.random.normal((num_examples, num_inputs),stddev = 1)
labels = true_w[0] * features[:,0] + true_w[1] * features[:,1] + true_b
labels += tf.random.normal(labels.shape,stddev=0.01)

# 读取数据集
def data_iter(batch_size, features, labels):
    num_examples = len(features)
    indices = list(range(num_examples))
    random.shuffle(indices)
    # 依次读取一个batch
    for i in range(0, num_examples, batch_size):
        j = indices[i: min(i+batch_size, num_examples)]
        # tf.gather抽取出params的第axis维度上在indices里面所有的index
        yield tf.gather(features, axis=0, indices=j), tf.gather(labels, axis=0, indices=j)

# 初始化模型参数

w = tf.Variable(tf.random.normal((num_inputs, 1), stddev=0.01))
b = tf.Variable(tf.zeros((1,)))

# 定义模型
def linreg(X, w, b):
    return tf.matmul(X, w) + b
# 损失函数
def squared_loss(y_hat, y):
    return (y_hat - tf.reshape(y, y_hat.shape)) ** 2 /2
# 优化函数
def sgd(params, lr, batch_size, grads):
    """Mini-batch stochastic gradient descent."""
    for i, param in enumerate(params):
        # 更新参数
        # assign_sub 变量 ref 减去 value值，即 ref = ref - value 并赋值
        param.assign_sub(lr * grads[i] / batch_size)

# 模型训练
lr = 0.03
num_epochs = 3
net = linreg
loss = squared_loss

for epoch in range(num_epochs):
    for X, y in data_iter(batch_size, features, labels):
        with tf.GradientTape() as t:
            t.watch([w,b])
            l = tf.reduce_sum(loss(net(X, w, b), y))
        # 自动求梯度
        # 二维的向量，分别表示[w,b]的梯度
        grads = t.gradient(l, [w, b])
        sgd([w, b], lr, batch_size, grads)
    train_l = loss(net(features, w, b), labels)
    print('epoch %d, loss %f' % (epoch + 1, tf.reduce_mean(train_l)))
```

使用模块写线性回归

```py

import tensorflow as tf

# 生成数据集
num_inputs = 2
num_examples = 1000
true_w = [2, -3.4]
true_b = 4.2
features = tf.random.normal(shape=(num_examples, num_inputs), stddev=1)
labels = true_w[0] * features[:, 0] + true_w[1] * features[:, 1] + true_b
labels += tf.random.normal(labels.shape, stddev=0.01)


from tensorflow import data as tfdata

batch_size = 10
# 将训练数据的特征和标签组合
dataset = tfdata.Dataset.from_tensor_slices((features, labels))
# 随机读取小批量
dataset = dataset.shuffle(buffer_size=num_examples) 
dataset = dataset.batch(batch_size)
data_iter = iter(dataset) # 迭代器读取每个batch

from tensorflow import keras
from tensorflow.keras import layers
from tensorflow import initializers as init

# 模型容器
model = keras.Sequential()
model.add(layers.Dense(1, kernel_initializer=init.RandomNormal(stddev=0.01)))

# 损失函数
from tensorflow import losses
loss = losses.MeanSquaredError()

# 优化器
from tensorflow.keras import optimizers
trainer = optimizers.SGD(learning_rate=0.03)

# 训练模型
num_epochs = 3
for epoch in range(1, num_epochs + 1):
    for (batch, (X, y)) in enumerate(dataset):
        # 记录梯度
        with tf.GradientTape() as tape:
            l = loss(model(X, training=True), y)
        # 获取变量梯度
        grads = tape.gradient(l, model.trainable_variables)
        # 更新参数
        # trainer为优化器
        trainer.apply_gradients(zip(grads, model.trainable_variables))
    
    l = loss(model(features), labels)
    print('epoch %d, loss: %f' % (epoch, l))
```
通过调用`tensorflow.GradientTape`记录动态图梯度，执行`tape.gradient`获得动态图中各变量梯度。通过 `model.trainable_variables` 找到需要更新的变量，并用 `trainer.apply_gradients` 更新权重，完成一步训练。


#### 文本分析

```py
import collections
from collections import defaultdict
import os
import random
import tarfile
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import preprocessing
from tensorflow.keras import layers
from tensorflow.keras.preprocessing.text import text_to_word_sequence, one_hot, Tokenizer
from tensorflow.keras.preprocessing.sequence import pad_sequences
import numpy as np
import sys
import time
import os
sys.path.append("..")
import d2lzh_tensorflow2 as d2l
print(tf.test.gpu_device_name())
DATA_ROOT = "../../data"

import torchtext.vocab as Vocab

port tensorflow as tf
AUTO = tf.data.experimental.AUTOTUNE
physical_devices = tf.config.list_physical_devices('GPU')
tf.config.experimental.set_memory_growth(physical_devices[0], enable=True)

#数据放入data目录下，代码解压速度较慢，如果不想用代码解压，也可直接手动解压，跳过这一步
import os
path = os.getcwd()#返回当前进程的工作目录
a_path = os.path.abspath(os.path.join(path, "../../data/aclImdb_v1.tar.gz"))
with tarfile.open(a_path, 'r') as f:
    f.extractall(DATA_ROOT)

def read_imdb(folder='train', data_root="../../data/aclImdb/"):
    data = []
    for label in ['pos', 'neg']:
        folder_name = os.path.join(data_root, folder, label)
        for file in (os.listdir(folder_name)):
            with open(os.path.join(folder_name, file), 'rb') as f:
                review = f.read().decode('utf-8').replace('\n', '').lower()
                data.append([review, 1 if label == 'pos' else 0])
    random.shuffle(data)
    return data
# 读取数据
train_data, test_data = read_imdb('train'), read_imdb('test')

# 预处理
def get_tokenized_imdb(data):
    """
    data: list of [string, label]
    返回: 每一条评论的单词所组成的列表
    """
    def tokenizer(text):
        #基于空格进行分词，并都转换为小写
        return [tok.lower() for tok in text.split(' ')]

    return [tokenizer(review) for review, _ in data]

text = get_tokenized_imdb(train_data)

# Counter对列表中的单词进行计数，并返回一个字典
counter = collections.Counter([tk for st in text for tk in st])
# vocab是一个字典，键表示单词，值表示单词出现的频率
vocab = {w: freq for w, freq in counter.most_common() if freq > 5}

def get_vocab_imdb(data):
    tokenized_data = get_tokenized_imdb(data)
    #counter已经创建了一个词典,统计了每个词出现的频率
    counter = collections.Counter([tk for st in tokenized_data for tk in st])
    return Vocab.Vocab(counter, min_freq=5)
    #text值保存counter中出现频率大于等于五的词
#     text = {w: freq for w, freq in counter.most_common() if freq >= 5}
#     vocab ={index:word for word,index in enumerate(text.keys())}
#     return vocab
    #return Vocab.Vocab(counter, min_freq=5)


vocab = get_vocab_imdb(train_data)
'# words in vocab:', len(vocab)

#在此处使用tensorflow2的填充函数进行填充
def preprocess_imdb(data, vocab):  # 本函数已保存在d2lzh_tensorflow2包中方便以后使用
    max_l = 500

    # 将每条评论通过截断或者补0，使得长度变成500
    def pad(x):
        return x[:max_l] if len(x) > max_l else x + [0] * (max_l - len(x))
    
    #tokenized_data为一个二维的列表,里面有我们分好的词
    tokenized_data = get_tokenized_imdb(data)
    
     #将每个词转换为词索引并进行截断或补0
    features = tf.Variable([pad([vocab[word] for word in words] ) for words in tokenized_data])
    labels = tf.Variable([score for _, score in data])
    return features, labels

data, label = preprocess_imdb(train_data, vocab)

batch_size = 64

# 训练集
train_set = (tf.data.Dataset.from_tensor_slices(
    ((preprocess_imdb(train_data, vocab))))
    .repeat()
    .shuffle(2048)
    .batch(batch_size)
    .prefetch(AUTO))
# 测试集
test_set = (tf.data.Dataset.from_tensor_slices(
    ((preprocess_imdb(test_data, vocab))))
    .shuffle(2048)
    .batch(batch_size)
    .prefetch(AUTO))

#因为tensorflow并没有像pytorch，mxnet关于glove接口的api，所以必须要重写一个

def load_embedding_from_disks(glove_filename, with_indexes=True):
    """
    Read a GloVe txt file. If `with_indexes=True`, we return a tuple of two dictionnaries
    `(word_to_index_dict, index_to_embedding_array)`, otherwise we return only a direct 
    `word_to_embedding_dict` dictionnary mapping from a string to a numpy array.
    """
    if with_indexes:
        word_to_index_dict = dict()
        index_to_embedding_array = []
        index_to_word_dict = dict()
        word_to_embedding = dict()
    else:
        word_to_embedding_dict = dict()

    
    with open(glove_filename, 'r',encoding='utf-8') as glove_file:
        for (i, line) in enumerate(glove_file):
            
            split = line.split(' ')
            
            word = split[0]
            
            representation = split[1:]
            representation = np.array(
                [float(val) for val in representation]
            )
            
            if with_indexes:
                word_to_index_dict[word] = i
                index_to_word_dict[i] = word
                word_to_embedding[word] = representation
                index_to_embedding_array.append(representation)
            else:
                word_to_embedding_dict[word] = representation

    _WORD_NOT_FOUND = [0.0]* len(representation)  # Empty representation for unknown words.
    if with_indexes:
        _LAST_INDEX = i + 1
        word_to_index_dict = defaultdict(lambda: _LAST_INDEX, word_to_index_dict)
        index_to_embedding_array = np.array(index_to_embedding_array + [_WORD_NOT_FOUND])
        return word_to_index_dict, index_to_embedding_array,index_to_word_dict,word_to_embedding
    else:
        word_to_embedding_dict = defaultdict(lambda: _WORD_NOT_FOUND)
        return word_to_embedding_dict
    
word_to_index, index_to_embedding, index_to_word,word_to_embedding = load_embedding_from_disks("C:/Users/HP/dive into d2l/code/chapter10_natural-language-processing/embeddings/ GloVe.6B/glove.6B.50d.txt", with_indexes=True)

def get_weights(vocab, word_to_embedding,embedding_dim,word_to_index,index_to_embedding):
    """从预训练好的vocab中提取出words对应的词向量"""
    embedding_matrix = np.zeros((len(vocab), embedding_dim))
#     embedding_matrix = np.zeros((len(vocab), embedding_dim))
    for index, word in enumerate(vocab.itos):
        if word in word_to_embedding.keys():
            embedding_matrix[index] = index_to_embedding[index]
    return embedding_matrix
embedding_matrix = get_weights(vocab,word_to_embedding,50,word_to_index,index_to_embedding)
# net.embedding.set_weights([embedding_matrix])
# net.trainable = False

embed_size, num_hiddens, max_len = 50, 100, 500
num_epochs = 5

model = tf.keras.Sequential([
    layers.Embedding(len(vocab), embed_size,weights=[embedding_matrix],input_length=500),
    layers.Bidirectional(layers.LSTM(num_hiddens)),
    tf.keras.layers.Dense(2,activation='softmax')
])

model.layers[0].trainable = False
model.summary()

model.compile(tf.keras.optimizers.Adam(0.01),
            loss='sparse_categorical_crossentropy',
            metrics=['sparse_categorical_accuracy'])

model.fit(
    train_set,
    steps_per_epoch=data.shape[0]//batch_size,
    validation_data= test_set,
    epochs=5
    )

```

### tensorflow 1.0

一个TensorFlow图由下面几个部分组成

* 占位符变量（Placeholder）用来改变图的输入。
* 模型变量（Model）将会被优化，使得模型表现得更好。
* 模型本质上就是一些数学函数，它根据Placeholder和模型的输入变量来计算一些输出。
* 一个cost度量用来指导变量的优化。
* 一个优化策略会更新模型的变量。
* 另外，TensorFlow图也包含了一些调试状态，比如用TensorBoard打印log数据

Placeholder是作为图的输入，每次我们运行图的时候都可能会改变它们。
```py
# 数据类型设置为float32，形状设为[None, img_size_flat]，None代表tensor可能保存着任意数量的图像，每张图象是一个长度为img_size_flat的向量。
x = tf.placeholder(tf.float32, [None, img_size_flat])
y_true_cls = tf.placeholder(tf.int64, [None])

# 变量的形状是[None, num_classes]，这代表着它保存了任意数量的标签，每个标签是长度为num_classes的向量，本例中长度为10。
y_true = tf.placeholder(tf.float32, [None, num_classes])

weights = tf.Variable(tf.zeros([img_size_flat, num_classes]))

logits = tf.matmul(x, weights) + biases
y_pred = tf.nn.softmax(logits)
y_pred_cls = tf.argmax(y_pred, axis=1)

cross_entropy = tf.nn.softmax_cross_entropy_with_logits_v2(logits=logits,
                                                           labels=y_true)
cost = tf.reduce_mean(cross_entropy)
optimizer = tf.train.GradientDescentOptimizer(learning_rate=0.5).minimize(cost)

correct_prediction = tf.equal(y_pred_cls, y_true_cls)
accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))

# 会话
session = tf.Session()
# 会对weights和biases变量初始化
session.run(tf.global_variables_initializer())、

batch_size = 100
def optimize(num_iterations):
    for i in range(num_iterations):
        # Get a batch of training examples.
        # x_batch now holds a batch of images and
        # y_true_batch are the true labels for those images.
        x_batch, y_true_batch, _ = data.random_batch(batch_size=batch_size)
        
        # Put the batch into a dict with the proper names
        # for placeholder variables in the TensorFlow graph.
        # Note that the placeholder for y_true_cls is not set
        # because it is not used during training.
        feed_dict_train = {x: x_batch,
                           y_true: y_true_batch}

        # Run the optimizer using this batch of training data.
        # TensorFlow assigns the variables in feed_dict_train
        # to the placeholder variables and then runs the optimizer.
        session.run(optimizer, feed_dict=feed_dict_train)
```

会话是用来执行定义好的运算，会话拥有并管理TensorFlow程序运行时的所有资源，所有计算完成之后需要关闭会话来帮助系统回收资源。用户通过placeholder定义占位符并构建完整Graph后，利用Session实例.run将训练/测试数据注入到图中，驱动任务的实际运行。

TensorFlow的设计处处体现了一个延后计算的思想。从而可以做更多静态分析，图优化，从而达到高运行效率。而pytorch图动态每一次前向时构建graph，反向时销毁。