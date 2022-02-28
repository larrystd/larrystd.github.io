---
title: pytorch
date: 2021-07-16
updated: 2021-07-16
tags: 
  - python
  - machine learning
categories:
  - math science
aplayer: true
---

### pytorch

#### 基本知识

创建和操作Tensor
```py
import torch
print("版本号为: {}".format(torch.__version__))

tensor_a = torch.arange(0,12)
print (tensor_a)

print ("张量存储位置: {}".format(tensor_a.device))

tensor([ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11])
张量存储位置: cpu

tensor_a = torch.arange(0,12)
print (tensor_a)

print ("张量存储位置: {}".format(tensor_a.device))

print (tensor_a.shape)

print (torch.zeros((2,3,4)))

print (torch.tensor([[1,2],[3,4]]))

print (torch.randn((3,4)))

torch.Size([12])
tensor([[[0., 0., 0., 0.],
         [0., 0., 0., 0.],
         [0., 0., 0., 0.]],

        [[0., 0., 0., 0.],
         [0., 0., 0., 0.],
         [0., 0., 0., 0.]]])
tensor([[1, 2],
        [3, 4]])
tensor([[-0.7588, -0.7888, -0.3240, -0.1797],
        [ 0.2589,  0.0594,  1.4420, -0.0355],
        [-0.7512,  0.7934,  2.4866, -0.5525]])

X = torch.Tensor([[1,2],[3,4],[5,6]])
Y = torch.Tensor([[11,12],[13,14],[15,16]])
Z = torch.Tensor([[2,3,4],[5,6,7]])

X+Y
X*Y 
X.float()/Y.float()
torch.exp(Y.float())
X @ Z  # 矩阵乘法

torch.cat((X, Y), 0)

tensor([[ 1.,  2.],
        [ 3.,  4.],
        [ 5.,  6.],
        [11., 12.],
        [13., 14.],
        [15., 16.]])

条件判别式，得到对应位置元素为0或1的Tensor
X == Y
tensor([[False, False],
        [False, False],
        [False, False]])

torch.sum(X)
tensor(21.)

item() 函数将单元素的tensor转换为标量, norm范数
torch.norm(X).item()

.viem() 相当于numpy的reshape

当然tensor也是可用切片的
```

<!-- more -->

reshape和view

torch的view()与reshape()方法都可以用来重塑tensor的shape，区别就是使用的条件不一样。view()方法只适用于满足连续性条件的tensor，并且该操作不会开辟新的内存空间，只是产生了对原存储空间的一个新别称和引用，返回值是视图。而reshape()方法的返回值既可以是视图，也可以是副本，当满足连续性条件时返回view，否则返回副本

tensor与numpy转换 
```py
import numpy as np
p = np.ones((2,3))
d = torch.from_numpy(p)

d.numpy()
```

自动求梯度

```py

import torch
from torch import autograd

x = torch.arange(4).float()


x.requires_grad_(True)

y = 2*x@x.t()

# y - 2 * torch.dot(x, x.t())
y

y.backward()

print (x.grad)

# 结果
tensor([ 0.,  4.,  8., 12.])

2*x*x 对x的梯度为4x, 结果为正确的
```

### 模型构造
#### 线性回归

```py
# 线性回归，数据集准备
import matplotlib.pyplot as plt 
import random
import numpy as np
import torch 

num_inputs = 2
num_examples = 1000
true_w = [2, -3.4] 
true_b = 4.2 
features = torch.randn(num_examples, num_inputs, dtype=torch.float32) 
labels = true_w[0] * features[:, 0] + true_w[1] * features[:,1]+true_b
labels += torch.tensor(np.random.normal(0, 0.01, size=labels.size()), dtype=torch.float32)

features[0], labels[0]、

from IPython import display


def use_svg_display():
    display.set_matplotlib_formats('svg')

def set_figsize(figsize=(3.5,2.5)):
    use_svg_display()
    plt.rcParams['figure.figsize'] = figsize

set_figsize()

plt.scatter(features[:, 1].numpy(), labels.numpy(), 1)

# 形成一个batch
def data_iter(batch_size, features, labels):
    num_examples = len(features)
    indices=list(range(num_examples))
    # 样本读取顺序随机, 索引
    random.shuffle(indices)
    for i in range(0, num_examples, batch_size):
        # 最后一次可能不够一个batch
        j = torch.LongTensor(indices[i : min(i+batch_size, num_examples)])

        yield features.index_select(0, j), labels.index_select(0, j)

# 一个batch
batch_size = 10
for X, y in data_iter(batch_size, features, labels):
    print (X ,y)
    break
# X 是一个batch的数据, batchsize*2的矩阵
# w 是2*1的tensor
def linreg(X, w, b):
    return torch.mm(X, w) + b

def square_loss(y_hat, y):
    return (y_hat - y.view(y_hat.size()))**2
# 优化算法
def sgd(params, lr, batch_size):
    for param in params:
        # 自动求梯度模块是批量样本的梯度和，除以batch大小作为batch梯度平均值
        param.data -= lr*param.grad / batch_size

# 初始化参数
w = torch.tensor(np.random.normal(0, 0.01, (num_inputs, 1)), dtype=torch.float32)
b = torch.zeros(1, dtype=torch.float32)

w.requires_grad_(requires_grad = True)
b.requires_grad_(requires_grad=True)


lr = 0.03
num_epochs = 3
net = linreg
loss = square_loss

for epoch in range(num_epochs):
    for X, y in data_iter(batch_size, features, labels):
        # 关于X, y的损失。一次求一个batch_size
        # loss返回batchsize的行向量，加个sum是损失和
        l = loss(net(X, w, b), y).sum()
        l.backward() # 参数求梯度
        sgd([w, b], lr, batch_size)

        # 梯度清零
        w.grad.data.zero_()
        b.grad.data.zero_()
    train_l = loss(net(features, w, b), labels)
    print ('epoch %d, loss %f' % (epoch+1, train_l.mean().item()))

```

模块型写法
```py
num_inputs = 2
num_examples = 1000
true_w = [2, -3.4]
true_b = 4.2

features = torch.tensor(
    np.random.normal(0, 1, (num_examples, num_inputs)),
    dtype = torch.float32
)
labels = true_w[0] * features[:, 0] + true_w[1] * features[:, 1] + true_b
labels += torch.tensor(np.random.normal(0, 0.01, size=labels.size()), dtype=torch.float32)


import torch.utils.data as Data

batch_size = 10
# 特征标签组合
dataset = Data.TensorDataset(features, labels)
# 随机选取小批量
data_iter = Data.DataLoader(dataset, batch_size, shuffle=True)

from torch import nn 
class LinearNet(nn.Module):
    def __init__(self, n_feature):
        super(LinearNet, self).__init__()
        self.linear = nn.Linear(n_feature, 1)

    def forward(self, x):
        y = self.linear(x)
        return y
net = LinearNet(num_inputs)

print (net)


net2 = nn.Sequential()
net2.add_module('linear', nn.Linear(num_inputs, 1))

# net.parameters()可查看模型所有可学习参数，函数返回一个生成器
for param in net.parameters():
    print (param)

# 初始化模型参数
from torch.nn import init

init.normal_(net.linear.weight, mean=0, std=0.01)
init.constant_(net.linear.bias, val=0)

loss = nn.MSELoss()

import torch.optim as optim

optimizer = optim.SGD(net.parameters(), lr=0.03)
print (optimizer)

num_epoch = 3
for epoch in range(1, num_epochs+1):
    for X, y in data_iter:
        output = net(X)
        l = loss(output, y.view(-1, 1))
        # 梯度清零
        optimizer.zero_grad()
        # 反向传播得到梯度
        l.backward()
        # 更新参数
        optimizer.step()
    print ('epoch %d, loss %f' % (epoch+1, train_l.mean().item()))
```
其中`nn.Sequential`可用看作有序容器，网络层将根据传入`Sequential`的顺序依次添加入网络层中，给定输入数据时，容器某一层将依次计算并输出作为下一层输入。

#### softmax模型

softmax回归和线性回归一样将输入特征与权重做线性累加，但不同之处是输出值个数等于标签里的类别数。也叫做全连接层。

softmax运算将输出值变成正且和为1的概率分布。


#### 继承Mudule类来构造模型

Module类是nn模块提供的模型构造类，是所有神经网络模块的基类
```py
import torch
from torch import nn
class MLP(nn.Module):
    def __init__(self, **kwargs):
        super(MLP, self).__init__(**kwargs)

        self.hidden = nn.Linear(784, 256)
        self.act = nn.Relu()
        self.output = nn.Linear(256,10)
    def forward(self, x):
        a = self.act(self.hidden(x))
        return self.output(a)

x = torch.randn(2, 784)
net = MLP()
print (net)
net(x)
```
`net(x)`会调用MLP继承自Mudule类的`__call__`函数，该函数调用MLP定义的forward函数来完成前向计算。


#### GPU计算
```py
a = torch.tensor([1,2,3], device = torch.device('cuda:1'))

import os
os.environ["CUDA_VISIBLE_DEVICES"] = "2"

device = torch.device(
    'cuda' if torch.cuda.is_available() else 'cpu'
)

cuda_gpu = torch.cuda.is_available()
if(cuda_gpu):
    net = torch.nn.DataParallel(net).cuda()
```
注意存储在不同位置的数据不能直接计算，不同GPU的数据也不能直接计算。pytorch要求所有输入数据都在内存或者同一块显卡的显存上。 device='cpu'说明Tensor存储在内存上。

#### RNN

```py
import time
import math
import numpy as np
import torch
from torch import nn, optim
import torch.nn.functional as F

import sys
sys.path.append("..") 
import d2lzh_pytorch as d2l
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

print(torch.__version__)
print(device)

(corpus_indices, char_to_idx, idx_to_char, vocab_size) = d2l.load_data_jay_lyrics()

# one-hot编码
def one_hot(x, n_class, dtype=torch.float32): 
    # X shape: (batch), output shape: (batch, n_class)
    x = x.long()
    res = torch.zeros(x.shape[0], n_class, dtype=dtype, device=x.device)
    # scatter_(dim, index, src)
    # dim 改数据的维度， 索引， 替换值
    res.scatter_(1, x.view(-1, 1), 1)
    return res
    
x = torch.tensor([0, 2])
one_hot(x, vocab_size)

def to_onehot(X, n_class):  
    # X shape: (batch, seq_len), output: seq_len elements of (batch, n_class)
    return [one_hot(X[:, i], n_class) for i in range(X.shape[1])]

X = torch.arange(10).view(2, 5)
inputs = to_onehot(X, vocab_size)
print(len(inputs), inputs[0].shape)
# 输出
5 torch.Size([2, 1027]) torch.Size([2, 256])

# 2*5 -> 5*2*256
# 尺寸变化 batch_size * seq_len ->  seq_len * batch * vocab_size
# num_inputs = vocab_size

num_inputs, num_hiddens, num_outputs = vocab_size, 256, vocab_size
print('will use', device)

def get_params():
    def _one(shape):
        ts = torch.tensor(np.random.normal(0, 0.01, size=shape), device=device, dtype=torch.float32)
        return torch.nn.Parameter(ts, requires_grad=True)

    # 隐藏层参数
    W_xh = _one((num_inputs, num_hiddens))
    W_hh = _one((num_hiddens, num_hiddens))
    b_h = torch.nn.Parameter(torch.zeros(num_hiddens, device=device, requires_grad=True))
    # 输出层参数
    W_hq = _one((num_hiddens, num_outputs))
    b_q = torch.nn.Parameter(torch.zeros(num_outputs, device=device, requires_grad=True))
    return nn.ParameterList([W_xh, W_hh, b_h, W_hq, b_q])

# 初始化
def init_rnn_state(batch_size, num_hiddens, device):
    return (torch.zeros((batch_size, num_hiddens), device=device))

def rnn(inputs, state, params):
    # inputs和outputs皆为num_steps个形状为(batch_size, vocab_size)的矩阵， 参数
    W_xh, W_hh, b_h, W_hq, b_q = params
    H, = state
    outputs = []
    for X in inputs:
        # 处理每一个seq字符
        # batch_size * hidden_size
        H = torch.tanh(torch.matmul(X, W_xh) + torch.matmul(H, W_hh) + b_h)
        Y = torch.matmul(H, W_hq) + b_q
        outputs.append(Y)
    # seq_len * batch_size * num_outputs(vocab_size)
    return outputs, (H,)

# batch_size * hidden_size
# input seq_len * batch * vocab_size
state = init_rnn_state(X.shape[0], num_hiddens, device)
inputs = to_onehot(X.to(device), vocab_size)
params = get_params()
outputs, state_new = rnn(inputs, state, params)
print(len(outputs), outputs[0].shape, state_new[0].shape)

# 输出
5 torch.Size([2, 1027]) torch.Size([2, 256])

# 裁剪梯度
def grad_clipping(params, theta, device):
    norm = torch.tensor([0.0], device=device)
    for param in params:
        norm += (param.grad.data ** 2).sum()
    norm = norm.sqrt().item()
    if norm > theta:
        for param in params:
            param.grad.data *= (theta / norm)
# 预测
def predict_rnn(prefix, num_chars, rnn, params, init_rnn_state,
                num_hiddens, vocab_size, device, idx_to_char, char_to_idx):
    state = init_rnn_state(1, num_hiddens, device)
    output = [char_to_idx[prefix[0]]]
    for t in range(num_chars + len(prefix) - 1):
        # 将上一时间步的输出作为当前时间步的输入
        X = to_onehot(torch.tensor([[output[-1]]], device=device), vocab_size)
        # 计算输出和更新隐藏状态
        (Y, state) = rnn(X, state, params)
        # 下一个时间步的输入是prefix里的字符或者当前的最佳预测字符
        if t < len(prefix) - 1:
            output.append(char_to_idx[prefix[t + 1]])
        else:
            output.append(int(Y[0].argmax(dim=1).item()))
    return ''.join([idx_to_char[i] for i in output])

predict_rnn('分开', 10, rnn, params, init_rnn_state, num_hiddens, vocab_size,device, idx_to_char, char_to_idx)

'分开养近朋身倒爽抱疑棒邻'

def train_and_predict_rnn(rnn, get_params, init_rnn_state, num_hiddens,
                          vocab_size, device, corpus_indices, idx_to_char,
                          char_to_idx, is_random_iter, num_epochs, num_steps,
                          lr, clipping_theta, batch_size, pred_period,
                          pred_len, prefixes):
    if is_random_iter:
        data_iter_fn = d2l.data_iter_random
    else:
        data_iter_fn = d2l.data_iter_consecutive
    params = get_params()
    loss = nn.CrossEntropyLoss()

    for epoch in range(num_epochs):
        if not is_random_iter:  # 如使用相邻采样，在epoch开始时初始化隐藏状态
            state = init_rnn_state(batch_size, num_hiddens, device)
        l_sum, n, start = 0.0, 0, time.time()
        data_iter = data_iter_fn(corpus_indices, batch_size, num_steps, device)
        for X, Y in data_iter:
            if is_random_iter:  # 如使用随机采样，在每个小批量更新前初始化隐藏状态
                state = init_rnn_state(batch_size, num_hiddens, device)
            else:  # 否则需要使用detach函数从计算图分离隐藏状态
                for s in state:
                    s.detach_()
            inputs = to_onehot(X, vocab_size)
            # outputs有num_steps个形状为(batch_size, vocab_size)的矩阵
            (outputs, state) = rnn(inputs, state, params)
            # 拼接之后形状为(num_steps * batch_size, vocab_size)
            outputs = torch.cat(outputs, dim=0)
            # Y的形状是(batch_size, num_steps)，转置后再变成长度为
            # batch * num_steps 的向量，这样跟输出的行一一对应
            y = torch.transpose(Y, 0, 1).contiguous().view(-1)
            # 使用交叉熵损失计算平均分类误差
            l = loss(outputs, y.long())
            
            # 梯度清0
            if params[0].grad is not None:
                for param in params:
                    param.grad.data.zero_()
            l.backward()
            grad_clipping(params, clipping_theta, device)  # 裁剪梯度
            d2l.sgd(params, lr, 1)  # 因为误差已经取过均值，梯度不用再做平均
            l_sum += l.item() * y.shape[0]
            n += y.shape[0]

        if (epoch + 1) % pred_period == 0:
            print('epoch %d, perplexity %f, time %.2f sec' % (
                epoch + 1, math.exp(l_sum / n), time.time() - start))
            for prefix in prefixes:
                print(' -', predict_rnn(prefix, pred_len, rnn, params, init_rnn_state,
                    num_hiddens, vocab_size, device, idx_to_char, char_to_idx))
```

#### 注意力机制

```py
def attention_model(input_size, attention_size):
    model = nn.Sequential(nn.Linear(input_size, attention_size, bias=False),
                          nn.Tanh(),
                          nn.Linear(attention_size, 1, bias=False))
    return model

def attention_forward(model, enc_states, dec_state):
    """
    enc_states: (时间步数, 批量大小, 隐藏单元个数)
    dec_state: (批量大小, 隐藏单元个数)
    """
    # 将解码器隐藏状态广播到和编码器隐藏状态形状相同后进行连结
    dec_states = dec_state.unsqueeze(dim=0).expand_as(enc_states)
    enc_and_dec_states = torch.cat((enc_states, dec_states), dim=2)
    # 输入连接后通入多层感知机
    e = model(enc_and_dec_states)  # 形状为(时间步数, 批量大小, 1)
    alpha = F.softmax(e, dim=0)  # 在时间步维度做softmax运算，结果仍为(时间步数, 批量大小, 1) 

    # 对enc_states的加权求和
    return (alpha * enc_states).sum(dim=0)  # 返回背景变量

seq_len, batch_size, num_hiddens = 10, 4, 8
model = attention_model(2*num_hiddens, 10) 
# (时间步数, 批量大小, 隐藏单元个数)
enc_states = torch.zeros((seq_len, batch_size, num_hiddens))
dec_state = torch.zeros((batch_size, num_hiddens))
attention_forward(model, enc_states, dec_state).shape

torch.Size([4, 8])


```

#### 编码器和解码器

```py
#三种特殊字符
PAD, BOS, EOS = '<pad>', '<bos>', '<eos>'
# 将一个序列中所有的词记录在all_tokens中以便之后构造词典，然后在该序列后面添加PAD直到序列
# 长度变为max_seq_len，然后将序列保存在all_seqs中
def process_one_seq(seq_tokens, all_tokens, all_seqs, max_seq_len):
    all_tokens.extend(seq_tokens)
    seq_tokens += [EOS] + [PAD] * (max_seq_len - len(seq_tokens) - 1)
    all_seqs.append(seq_tokens)

# 使用所有的词来构造词典。并将所有序列中的词变换为词索引后构造Tensor
def build_data(all_tokens, all_seqs):
    vocab = Vocab.Vocab(collections.Counter(all_tokens),
                        specials=[PAD, BOS, EOS])
    indices = [[vocab.stoi[w] for w in seq] for seq in all_seqs]
    return vocab, torch.tensor(indices)

def read_data(max_seq_len):
    # in和out分别是input和output的缩写
    in_tokens, out_tokens, in_seqs, out_seqs = [], [], [], []
    with io.open('../../data/fr-en-small.txt') as f:
        lines = f.readlines()
    for line in lines:
        in_seq, out_seq = line.rstrip().split('\t')
        in_seq_tokens, out_seq_tokens = in_seq.split(' '), out_seq.split(' ')
        if max(len(in_seq_tokens), len(out_seq_tokens)) > max_seq_len - 1:
            continue  # 如果加上EOS后长于max_seq_len，则忽略掉此样本
        process_one_seq(in_seq_tokens, in_tokens, in_seqs, max_seq_len)
        process_one_seq(out_seq_tokens, out_tokens, out_seqs, max_seq_len)
    in_vocab, in_data = build_data(in_tokens, in_seqs)
    out_vocab, out_data = build_data(out_tokens, out_seqs)
    return in_vocab, out_vocab, Data.TensorDataset(in_data, out_data)

class Encoder(nn.Module):
    def __init__(self, vocab_size, embed_size, num_hiddens, num_layers,
                 drop_prob=0, **kwargs):
        super(Encoder, self).__init__(**kwargs)
        self.embedding = nn.Embedding(vocab_size, embed_size)
        self.rnn = nn.GRU(embed_size, num_hiddens, num_layers, dropout=drop_prob)

    def forward(self, inputs, state):
        # 输入形状是(批量大小, 时间步数)。将输出互换样本维和时间步维
        embedding = self.embedding(inputs.long()).permute(1, 0, 2) # (seq_len, batch, input_size)
        return self.rnn(embedding, state)

    def begin_state(self):
        return None
encoder = Encoder(vocab_size=10, embed_size=8, num_hiddens=16, num_layers=2)
output, state = encoder(torch.zeros((4, 7)), encoder.begin_state())
output.shape, state.shape # GRU的state是h, 而LSTM的是一个元组(h, c)

(torch.Size([7, 4, 16]), torch.Size([2, 4, 16]))


# 解码器
class Decoder(nn.Module):
    def __init__(self, vocab_size, embed_size, num_hiddens, num_layers,
                 attention_size, drop_prob=0):
        super(Decoder, self).__init__()
        self.embedding = nn.Embedding(vocab_size, embed_size)
        self.attention = attention_model(2*num_hiddens, attention_size)
        # GRU的输入包含attention输出的c和实际输入, 所以尺寸是 num_hiddens+embed_size
        self.rnn = nn.GRU(num_hiddens + embed_size, num_hiddens, 
                          num_layers, dropout=drop_prob)
        self.out = nn.Linear(num_hiddens, vocab_size)

    def forward(self, cur_input, state, enc_states):
        """
        cur_input shape: (batch, )
        state shape: (num_layers, batch, num_hiddens)
        """
        # 使用注意力机制计算背景向量
        c = attention_forward(self.attention, enc_states, state[-1])
        # 将嵌入后的输入和背景向量在特征维连结, (批量大小, num_hiddens+embed_size)
        input_and_c = torch.cat((self.embedding(cur_input), c), dim=1) 
        # 为输入和背景向量的连结增加时间步维，时间步个数为1
        output, state = self.rnn(input_and_c.unsqueeze(0), state)
        # 移除时间步维，输出形状为(批量大小, 输出词典大小)
        output = self.out(output).squeeze(dim=0)
        return output, state

    def begin_state(self, enc_state):
        # 直接将编码器最终时间步的隐藏状态作为解码器的初始隐藏状态
        return enc_state

def batch_loss(encoder, decoder, X, Y, loss):
    # (1) 编码得到向量c
    # (2) 基于向量c等解码一遍
    # (3) 求损失函数，更新参数
    batch_size = X.shape[0]
    enc_state = encoder.begin_state()
    # 编码器，同时更新当前编码状态
    enc_outputs, enc_state = encoder(X, enc_state)
    # 初始化解码器的隐藏状态
    dec_state = decoder.begin_state(enc_state)
    # 解码器在最初时间步的输入是BOS
    dec_input = torch.tensor([out_vocab.stoi[BOS]] * batch_size)
    # 我们将使用掩码变量mask来忽略掉标签为填充项PAD的损失, 初始全1
    # 注意一次性处理的是一个batchsize
    mask, num_not_pad_tokens = torch.ones(batch_size,), 0
    l = torch.tensor([0.0])
    for y in Y.permute(1,0): #维度换位。 Y shape: (batch, seq_len)
    # 基于当前y字符，状态，enc_outputs对y进行解码
    # 同时更新当前解码输入和解码状态
        dec_output, dec_state = decoder(dec_input, dec_state, enc_outputs)
        l = l + (mask * loss(dec_output, y)).sum()
        dec_input = y  # 使用强制教学

        num_not_pad_tokens += mask.sum().item()
        # EOS后面全是PAD. 下面一行保证一旦遇到EOS接下来的循环中mask就一直是0
        # 一旦遇到EOS停止解码
        mask = mask * (y != out_vocab.stoi[EOS]).float()
    # 损失函数，分母是非padding的字符数量
    return l / num_not_pad_tokens

def train(encoder, decoder, dataset, lr, batch_size, num_epochs):
    enc_optimizer = torch.optim.Adam(encoder.parameters(), lr=lr)
    dec_optimizer = torch.optim.Adam(decoder.parameters(), lr=lr)

    loss = nn.CrossEntropyLoss(reduction='none')
    data_iter = Data.DataLoader(dataset, batch_size, shuffle=True)
    for epoch in range(num_epochs):
        l_sum = 0.0
        for X, Y in data_iter:
            enc_optimizer.zero_grad()
            dec_optimizer.zero_grad()
            l = batch_loss(encoder, decoder, X, Y, loss)
            l.backward()
            enc_optimizer.step()
            dec_optimizer.step()
            l_sum += l.item()
        if (epoch + 1) % 10 == 0:
            print("epoch %d, loss %.3f" % (epoch + 1, l_sum / len(data_iter)))

embed_size, num_hiddens, num_layers = 64, 64, 2
attention_size, drop_prob, lr, batch_size, num_epochs = 10, 0.5, 0.01, 2, 50
encoder = Encoder(len(in_vocab), embed_size, num_hiddens, num_layers,
                  drop_prob)
decoder = Decoder(len(out_vocab), embed_size, num_hiddens, num_layers,
                  attention_size, drop_prob)
train(encoder, decoder, dataset, lr, batch_size, num_epochs)


# 预测
def translate(encoder, decoder, input_seq, max_seq_len):
    in_tokens = input_seq.split(' ')
    in_tokens += [EOS] + [PAD] * (max_seq_len - len(in_tokens) - 1)
    enc_input = torch.tensor([[in_vocab.stoi[tk] for tk in in_tokens]]) # batch=1
    enc_state = encoder.begin_state()
    # 编码，基于训练好的参数
    enc_output, enc_state = encoder(enc_input, enc_state)
    # 起始单词BOS
    dec_input = torch.tensor([out_vocab.stoi[BOS]])
    dec_state = decoder.begin_state(enc_state)
    output_tokens = []

    # 解码，获取预测的序列
    for _ in range(max_seq_len):
        dec_output, dec_state = decoder(dec_input, dec_state, enc_output)
        pred = dec_output.argmax(dim=1)

        pred_token = out_vocab.itos[int(pred.item())]
        if pred_token == EOS:  # 当任一时间步搜索出EOS时，输出序列即完成
            break
        else:
            output_tokens.append(pred_token)
            dec_input = pred
    return output_tokens

input_seq = 'ils regardent .'
translate(encoder, decoder, input_seq, max_seq_len)

['they', 'are', 'watching', '.']
```