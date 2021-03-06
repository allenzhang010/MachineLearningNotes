
#Build Kaldi

```sh
git clone https://github.com/kaldi-asr/kaldi.git kaldi --origin upstream
cd kaldi
cd tools
extras/check_dependencies.sh

# install sugested packages

make

cd ../src
./configure --shared

make depend
make all
```
#Run Example YesNo

**运行**
在yesno的s5下有几个文件:
1. path.sh - 用来设置kaldi的路径
2. run.sh - 就是用来执行训练及预测的脚本

我们可以修改在run.sh中的数据的位置，也可以使用缺省的位置

```sh
cd ../egs
cd yesno/s5
./run.sh
```
yesno是个最简单的例子，运行很快就结束了。接下来看看这个脚本做了什么
1. 从sourceforge上下载yesno的音频文件
音频文件名就代表了音频的label，每个音频有8个单词，分别为yes或者no，文件名中1代表yes，0代表no
2. 调用prepare_data.sh准备数据
3. 特征提取
4. mono training
5. graph compiler
6. decode
```sh
# Feature extraction
for x in train_yesno test_yesno; do
 steps/make_mfcc.sh --nj 1 data/$x exp/make_mfcc/$x mfcc
 steps/compute_cmvn_stats.sh data/$x exp/make_mfcc/$x mfcc
 utils/fix_data_dir.sh data/$x
done

# Mono training
steps/train_mono.sh --nj 1 --cmd "$train_cmd" \
  --totgauss 400 \
  data/train_yesno data/lang exp/mono0a

# Graph compilation
utils/mkgraph.sh data/lang_test_tg exp/mono0a exp/mono0a/graph_tgpr

# Decoding
steps/decode.sh --nj 1 --cmd "$decode_cmd" \
    exp/mono0a/graph_tgpr data/test_yesno exp/mono0a/decode_test_yesno

for x in exp/*/decode*; do [ -d $x ] && grep WER $x/wer_* | utils/best_wer.sh; done
```


**数据准备**
数据准备阶段需要原始语音，标注以及语言模型(发音字典，因素集合等)

```sh
root@localhost s5]# cd data
[root@localhost data]# ls
lang  lang_test_tg  local  test_yesno  train_yesno

[root@localhost data]# cd train_yesno/
[root@localhost train_yesno]# ls
cmvn.scp  feats.scp  spk2utt  split1  text  utt2spk  wav.scp

[root@localhost train_yesno]# cd ../lang
[root@localhost lang]# ls
L_disambig.fst  L.fst  oov.int  oov.txt  phones  phones.txt  topo  words.txt
```

* utt2spk - 声音文件到说话人的映射
* spk2utt - 说话人到声音文件的映射
* text - 标注发音内容，第一列是发音编号
* cmvn.scp - 统计说话人的语音信息（倒谱均值，方差归一化）

* L.fst - FST形式的发音字典
* L_disambig.fst - 带消岐信息的发音字典
* oov.txt - 词汇表以外的词
* oov.int - 词汇表以外的词对应的int表示
* topo - HMM模型的拓扑结构
* phones目录 - 包含音素集合的各种信息

data下的dict目录需要手工创建，指明静音音素和实际的音素


**Feature Extraction**
特征只提取了mfcc
```sh
 steps/make_mfcc.sh --nj 1 data/$x exp/make_mfcc/$x mfcc
 steps/compute_cmvn_stats.sh data/$x exp/make_mfcc/$x mfcc
```
其中data是输入路径，exp和mfcc是输出目录。exp下记录的是log，包括运行中的各种命令。

在mcc下生成两个文件：scp和ark文件。关于mfcc的介绍参考<a href="http://blog.csdn.net/hlx371240/article/details/45009415">这里</a>

Kaldi源代码的feat/feature_mfcc.h中定义了MfccComputer类，用来从波形文件中计算音频feature。


**单音素训练和解码**

**编译解码图**
Kaldi的工作核心是解码网络，整个解码网络都是围绕HCLG的图来建立的。

* G是用来编码语法或者语言模型的接收器(i.e：它的输入和输出符号是一样的)
* L是发声词典；输出是词，输入是音素。
* C表示上下文关系，输出是音素，输入是上下文相关的音素例如N个音素组成的窗。具体看Phonetic context windows里面的介绍。
* H包含HMM的定义；输出符表示上下文相关的音素，输入是transitions-ids(转移id)，transitions-ids是编码pdf-id或者其他信息(自转或向后转)，具体看( Integer identifiers used by TransitionModel)

**解码**

#在线解码及识别

kaldi里有一些旧版本的代码在src/online和src/onlinebin里，可以为在线解码提供一些示范性的工具。
但是同时kaldi还提供了另外一些的代码在src/online2和src/onlinebin下，根据官方文档，Kaldi希望为用户
提供一些构建在线解码用的库，而不是提供完整的工具。Kaldi希望用户自己根据这些库区构建自己的工具。

缺省编译时并不会编译src/online和src/onlinebin里的代码，我们可以手工编译。
```sh
cd tools
sh install_portaudio.sh

cd ../src/online
make

cd ../onlinebin
make
```
编译后再src/online下产生一个文件libkaldi-online.so。在onlinebin里编译时需要用到这个so文件，同时会编译出一些可执行文件。


#Reference

1. <a href="http://kaldi-asr.org/doc/">Kaldi官方网站</a>
2. <a href="https://shiweipku.gitbooks.io/chinese-doc-of-kaldi/content/acoustic_model.html">Kaldi中文文档</a>
3. <a href="https://www.gitbook.com/book/shiweipku/chinese-doc-of-kaldi/details">Kaldi中文文档pdf下载</a>
4. <a href="http://wenku.baidu.com/link?url=8jTZ88oD_2OBpgUzeEmv9iaU5WKTW0WDdU7kKMQdjOFhHaF1UG54xtCUmXzPuQ1sG488qCEXkwwgSMLoHUGQ65hx9_GW2ustrZS4ohh4O2K">比较详尽的介绍 - kaldi的全部资料_v0.4</a>
5. <a href="http://blog.csdn.net/wbgxx333">上文作者的博客</a>
6. <a href="http://threedweb.cn/forum-76-1.html">语音识别论坛</a>