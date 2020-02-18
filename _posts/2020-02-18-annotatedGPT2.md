# The Annotated GPT-2

1. TOC 
{:toc}

## Introduction
Welcome to "**The Annotated GPT-2**". 

One of the most brilliant and well-explained articles I have ever read is [The Annotated Transformer](https://nlp.seas.harvard.edu/2018/04/03/attention.html). It introduced *attention* like no other post ever written. The simple idea was to present an "annotated" version of the paper [Attention is all you need](https://arxiv.org/abs/1706.03762) along with code.

Something I have come to realize with my little experience in Machine Learning, when you write things in code, the implementation and the secrets become clearer. It is not magic anymore. 

>  There is nothing magic about magic. The magician merely understands something simple which doesn’t appear to be simple or natural to the untrained audience. Once you learn how to hold a card while making your hand look empty, you only need practice before you, too, can “do magic.”
>
> -- Jeffrey Friedl in the book [Mastering Regular Expressions](https://learning.oreilly.com/library/view/mastering-regular-expressions/0596528124/ch01.html)

The **[GPT-2](https://openai.com/blog/better-language-models/)** might seem like magic at first with all it's glitter and beauty too, but hopefully I would have uncovered that magic for you and revealed all the tricks by the time you finish reading this post. That is my goal. To make it as simple as possible for the keen to understand how the **GPT-2** model works underneath. 

**Note:** Pretty much the entirety of the code has been copied, inspired and referenced from [Hugging Face's implementation](https://github.com/huggingface/transformers/blob/master/src/transformers/modeling_gpt2.py) of the GPT-2, keeping merely the essentials for simplicity. If you want to train the GPT-2 model on parallel GPUs, save checkpoints while fine-tuning, run inference tasks on multiple CPUs and much more, I would recommend using the Hugging Face API. A simple tutorial on how to do so was recently released by Hugging Face and can be found [here](https://huggingface.co/blog/how-to-train).
In this post, I am not trying to reinvent the wheel, but merely bringing together a list of prexisting excellent resources to make it easier for the reader to grasp GPT-2. I leave it up to the reader to further build upon these foundations in any area they choose.

> You can't build a great building on a weak foundation. You must have a solid foundation if you're going to have a strong superstructure. 
>
> -- Gordon B. Hinckley

## Prerequisites
To understand the GPT-2 model completely, we will first need to take a deep dive inside Transformers. The GPT-2 is a Transformer based architecture utilizing the Decoder only part. Here is an excellent list of resources to aid your understanding regarding Transformers: 
1. [The illustrated Transformer](http://jalammar.github.io/illustrated-transformer/) by Jay Alammar
1. [The Annotated Transformer](https://nlp.seas.harvard.edu/2018/04/03/attention.html) by Harvard NLP 
1. [Introduction to the Transformer](https://www.youtube.com/watch?v=AFkGPmU16QA&list=PLtmWHNX-gukKocXQOkQjuVxglSDYWsSh9&index=18&t=0s) by Rachel Thomas and Jeremy Howard


---
## Attention is all you need

---
> Let's first look at Attention.

![](/images/Transformer-architecture.PNG "Transformer Architecture")

> The GPT-2 only utilizes the Decoder layer on the right with only 1 Multi Head Attention Layer followed by a FeedForward Network.

## Imports
```python
import torch
import copy
import torch.nn as nn
import torch.nn.functional as F
from torch.nn.modules import ModuleList
from torch.nn.modules.normalization import LayerNorm
import numpy as np
import os
from tqdm import tqdm_notebook, trange
import logging
logging.basicConfig(level = logging.INFO)
logger = logging.getLogger()
```
## Transformer Decoder
To re-use the terminology used to describe the Transformer, the attention is a function of a query (Q) and set of key (K) and value (V) pairs. To handle longer sequences, we modify the multi-head self-attention of the Transformer to reduce memory usage by limiting the dot products between Q and K in:

![](/images/Attention-formula.PNG "Attention as a combination of query, key & value")

```python
class Conv1D(nn.Module):
    def __init__(self, nx, nf):
        super().__init__()
        self.nf = nf
        w = torch.empty(nx, nf)
        nn.init.normal_(w, std=0.02)
        self.weight = nn.Parameter(w)
        self.bias = nn.Parameter(torch.zeros(nf))

    def forward(self, x):
        size_out = x.size()[:-1] + (self.nf,)
        x = torch.addmm(self.bias, x.view(-1, x.size(-1)), self.weight)
        x = x.view(*size_out)
        return x
```
### CONV1D Layer Explained
The `CONV1D` layer can be thought of as a LINEAR layer itself. Essentially, it is casting an initial tensor `x` (having the final dimension of `x.size(-1)`) being passed to it to have a final dimension of size `self.nf`. 

Here's an example output of the same: 
```python
conv1d = CONV1D(768, 768*3)
x      = torch.rand(1,4,768) #represents a sequence of batch_size=1, seq_len=4 and embedding_sz=768, something like "Hello how are you"
x      = conv1d(x)
x.shape

>> torch.size([1, 4, 768*3])
```
As can be seen in the example above, the final dimension of tensor returned by `CONV1D` is 3 times the initial size. We do this to be able to cast the input to query, key and value.

It is possible then to retrieve the query, key and value matrices like so:
```python
query, key, value = x.split(768, dim=-1)

query.shape, key.shape, value.shape 
>> torch.size([1,4,768]), torch.size([1,4,768]), torch.size([1,4,768])
```

> Another way to do this would have been to have separate `Wq`, `Wk` and `Wv` matrices. I have explained this under the **EXTRA** section of this post at the bottom. 

### FEEDFORWARD Layer Explained
```python
class FeedForward(nn.Module):
    def __init__(self, dropout, d_model=768, nx=768*4):
        super().__init__()
        self.c_fc    = Conv1D(d_model, nx)
        self.c_proj  = Conv1D(nx, d_model)
        self.act     = F.gelu
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x):
        return self.dropout(self.c_proj(self.act(self.c_fc(x))))
```
Something, that's just so well explained in Jay Alammar's post - also referenced above, is how the inputs are passed through `ATTENTION` layer first and then on to `FEEDFORWARD` layer. The Feedforward network, is a normal nueral network that accepts the outputs from the `ATTENTION` layer (which are of size 768 - explained in the next section), casts them to `nx` (768*4) dimension, adds an activation function `self.act` (GELU), casts them back to `d_model` (768) and adds dropout (0.1). 

This is also mentioned in the **GPT** research paper referenced below. 
> For the position-wise feed-forward networks, we used 3072 dimensional inner states

At present, there is active research to use `self.attention` based Decoder networks only, but until now networks with the `feedforward` layer seem to do a better job. ##TODO: Add reference

### ATTENTION Layer Explained
```python
class Attention(nn.Module):
    def __init__(self, d_model=768, n_head=12, n_ctx=1024, d_head=64, bias=True, scale=False):
        super().__init__()
        self.n_head  = n_head
        self.d_model = d_model
        self.c_attn  = Conv1D(d_model, d_model*3)
        self.scale   = scale
        self.softmax = nn.Softmax(dim=-1)
        self.register_buffer("bias", torch.tril(torch.ones(n_ctx, n_ctx)).view(1, 1, n_ctx, n_ctx))
        self.dropout = nn.Dropout(0.1)
        self.c_proj  = Conv1D(d_model, d_model)
        
    def split_heads(self, x):
        "return shape [`batch`, `head`, `sequence`, `features`]"
        new_shape = x.size()[:-1] + (self.n_head, x.size(-1)//self.n_head) 
        x = x.view(*new_shape)
        return x.permute(0, 2, 1, 3) 
    
    def _attn(self, q, k, v, attn_mask=None):
        scores  = torch.matmul(q, k.transpose(-2, -1))
        if self.scale: scores = scores/math.sqrt(v.size(-1))
        nd, ns  = scores.size(-2), scores.size(-1)
        bias    = self.bias[:,:, ns-nd:ns, :ns]
        scores  = scores*bias - 1e4*(1-bias) 
        if attn_mask is not None: scores = scores + attn_mask
        scores  = self.softmax(scores)
        scores  = self.dropout(scores)
        outputs = torch.matmul(scores, v)
        return outputs
    
    def merge_heads(self, x):
        x         = x.permute(0, 2, 1, 3).contiguous()
        new_shape = x.size()[:-2] + (x.size(-2)*x.size(-1),)
        return x.view(*new_shape)
        
    def forward(self, x):
        x        = self.c_attn(x) #new `x` shape - `[1,3,2304]`
        q, k, v  = x.split(self.d_model, dim=2)
        q, k, v  = self.split_heads(q), self.split_heads(k), self.split_heads(v)
        out      = self._attn(q, k, v)
        out      = self.merge_heads(out)
        out      = self.c_proj(out)
        return out
```


---
## Language Models are Unsupervised Multitask Learners

---

## Abstract
Natural language processing tasks, such as question answering, machine translation, reading comprehension, and summarization, are typically approached with supervised learning on taskspecific datasets. We demonstrate that language models begin to learn these tasks without any explicit supervision when trained on a new dataset of millions of webpages called WebText. When conditioned on a document plus questions, the answers generated by the language model reach 55 F1 on the CoQA dataset matching or exceeding the performance of 3 out of 4 baseline systems without using the 127,000+ training examples. The capacity of the language model is essential to the success of zero-shot task transfer and increasing it improves performance in a log-linear fashion across tasks. Our largest model, GPT-2, is a 1.5B parameter Transformer that achieves state of the art results on 7 out of 8 tested language modeling datasets in a zero-shot setting but still underfits WebText. Samples from the model reflect these improvements and contain coherent paragraphs of text. These findings suggest a promising path towards building language processing systems which learn to perform tasks from their naturally occurring demonstrations.

## Model
We use a [Transformer](https://arxiv.org/abs/1706.03762) (Vaswani et al., 2017) based architecture for our LMs. The model largely follows the details of the [OpenAI GPT model](https://s3-us-west-2.amazonaws.com/openai-assets/research-covers/language-unsupervised/language_understanding_paper.pdf) (Radford et al., 2018) with a few modifications. [Layer normalization](https://arxiv.org/abs/1607.06450) (Ba et al., 2016) was moved to the input of each sub-block, similar to a pre-activation residual network (He et al., 2016) and an additional layer normalization was added after the final self-attention block. A modified initialization which accounts for the accumulation on the residual path with model depth is used. We scale the weights of residual layers at initialization by a factor of 1/√N where N is the number of residual layers. The vocabulary is expanded to 50,257. We also **increase the context size from 512 to 1024** tokens and a larger batchsize of 512 is used.

> This is the entirety of model explanation inside the `GPT-2` research paper. This warrants a need for us to look at the architecture inside the `GPT` model.

## Model Specifications (GPT)
Our model largely follows the original transformer work. We trained a **12-layer decoder-only transformer** with **masked self-attention heads** (768 dimensional states and 12 attention heads). For the position-wise feed-forward networks, we used 3072 dimensional inner states. We used the Adam optimization scheme with a max learning rate of 2.5e-4. The learning rate was increased linearly from zero over the first 2000 updates and annealed to 0 using a cosine schedule. We train for 100 epochs on minibatches of 64 randomly sampled, contiguous sequences of 512 tokens. Since [layernorm](https://arxiv.org/abs/1607.06450) is used extensively throughout the model, a simple weight initialization of **N(0, 0.02)** was sufficient. We used a **bytepair encoding (BPE)** vocabulary with 40,000 merges and residual, embedding, and attention dropouts with a rate of 0.1 for regularization. We also employed a modified version of L2 regularization proposed in, with w = 0.01 on all non bias or gain weights. For the activation function, we used the [Gaussian Error Linear Unit (GELU)](https://arxiv.org/abs/1606.08415). 

![](/images/gpt-architecture.PNG "GPT Architecture")

> An excellent resource to further explain the model is written by Jay Alammar - [The illustrated GPT-2](http://jalammar.github.io/illustrated-gpt2/). This would be a good time to read the introduction part of this blog post which very clearly explains The Transformer Block. 


```python
class GPT2(nn.Module):
    def __init__(self, nlayers=12, n_ctx=1024, d_model=768, vcb_sz=50257):
        super(GPT2, self).__init__()
        self.nlayers = nlayers
        block        = TransformerBlock(d_model=768, n_head=12, dropout=0.1)
        self.h       = _get_clones(block, 12)
        self.wte     = nn.Embedding(vcb_sz, d_model)
        self.wpe     = nn.Embedding(n_ctx, d_model)
        self.drop    = nn.Dropout(0.1)
        self.ln_f    = LayerNorm(d_model)
        self.out     = nn.Linear(d_model, vcb_sz, bias=False)
        self.loss_fn = nn.CrossEntropyLoss()
        self.init_weights()
    
    def init_weights(self):
        self.out.weight = self.wte.weight
        self.apply(self._init_weights)
    
    def _init_weights(self, module):
        if isinstance(module, (nn.Linear, nn.Embedding, Conv1D)):
            module.weight.data.normal_(mean=0.0, std=0.02)
            if isinstance(module, (nn.Linear, Conv1D)) and module.bias is not None:
                module.bias.data.zero_()
        elif isinstance(module, nn.LayerNorm):
            module.bias.data.zero_()
            module.weight.data.fill_(1.0)
    
    def forward(self, src, labels=None, pos_ids=None):
        if pos_ids is None: pos_ids = torch.arange(0, src.size(-1)).unsqueeze(0)
        inp = self.drop((self.wte(src)+self.wpe(pos_ids)))
        for i in range(self.nlayers): inp = self.h[i](inp)
        inp     = self.ln_f(inp)
        logits  = self.out(inp)
        outputs = (logits,) + (inp,)
        
        if labels is not None:
            shift_logits = logits[..., :-1, :].contiguous()
            shift_labels = labels[..., 1:].contiguous()
            loss = self.loss_fn(shift_logits.view(-1, shift_logits.size(-1)), shift_labels.view(-1))
            outputs = (loss,) + outputs
            return outputs
        return logits
```

