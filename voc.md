# iBooks高光词汇自动提取器

我平时用苹果阅读器看书的时候遇到不会的单词会用高光标记，之后再集中导出配上释义。但是苹果的导出非常麻烦，看了几百本书以后显然手动导出速度太慢，于是我在想能不能一键生成。

## 提取标记
这一部分参考了Kevin Cunningham (@doingandlearning)的[node程序](https://www.kevincunningham.co.uk)。苹果阅读器用iCloud同步，而iCloud在本地的sqlite里储存了对应的书本信息和每本书里的笔记信息。这个程序需要手动找到两个数据库的位置，然后填好路径以后程序就会自动提取出信息，输出是一系列md文件。

比如Theodore Sturgeon的小说A Saucer of Loneliness被转化成了以下文件。标题以"#"开头，作者以"By"开头，被高光标记的生词前面会跟一个"-"，有了这些文件以后，下一步就可以开始分隔处理。
```md
# A Saucer of Loneliness

By Theodore Sturgeon

## My notes <a name="my_notes_dont_delete"></a>

- nictated
- carcass
- synapse
- integument
- cowl
- burnoose
- fleshpots
- shekel’s
- humanoids
- fairies
- dun
```

## 文字分割转化
我们的目标是把每个单词转化成这种格式：[单词, 释义, 音标，书籍，作者]，并储存在csv文件里

对于一个单词，我们得到中文释义可以使用pygtrans，这是一个由谷歌翻译支持的库，它的使用方法如下。在实际代码中，由于取得一个单词的意思需要发送http请求，我们用多线程可以让速度提升10倍以上。对于英文释义，我们用了韦氏词典，有人做了词典的[json格式](https://github.com/matthewreagan/WebstersEnglishDictionary)，对应英文释义。我的处理方法是如果韦氏词典有这个单词就用韦氏，如果没有就换谷歌翻译。

```py
from pygtrans import Translate

word = "apple"
client = Translate()
chinese = client.translate(word).translatedText
```

由于书籍里的单词常会有变形，我们还需要提前对英文进行处理，nltk库能把不同词性的单词转化为它们的原型，比如复数转化为单数，过去时转化为原型。同时因为我标记了不止单词，有时候还会标记一些句子，程序还需要把句子过滤掉，判断标准是看长度有没有超过25个字符。

```py
import nltk.stem as ns

lemmalizer = ns.WordNetLemmatizer()
word = "birds"
# the string "birds" becomes "bird"
word = lemmalizer.lemmatize(word)
```

音标可以用eng_to_ipa库，它返回的是国内学习的国际音标，而不是韦氏词典那种美式音标：
```py
import eng_to_ipa

word = "cat"
ipa = eng_to_ipa.convert(word)
```

我写了一个python程序把上面的md文件提取出书名、作者、单词，每一个单词用库得到对应的释义，音标后就可以使用pandas库导出到csv文件了。这种方法共导出约7000个单词，然后保存到excel里。有了csv文件后，一些开源代码也提供了一键制作anki卡片等操作。由于数据里包含了每个单词对应的书籍和作者，也很容易写一个程序专门抽取某本书或某作者的单词。


## 缺陷
由于标记本身的缺陷(不一定都标记生词)和自然语言的复杂，常见错误包括：

• 翻译错误，韦氏词典的释义一般是对的，但中文翻译偶尔会出现错误，比如"浸湿"soused被翻译成了"邻居" <br>
• 词形变化错误，乐章partitas没有被去掉复数<br>
• 缺乏书籍语境，比如singular被取了单数的意思，但原文取spectacular意<br>
• 误录标记，比如最终的输出里有人名Katy Perry, 短句"I’m not gay, Tyler"

但绝大多数单词都是对的，错误的单词一般也属于生僻词汇，所以考虑到效率，对于笔记而言影响这个错误率可以接受






