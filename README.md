# CPM

## 项目描述
CPM（Chinese Pretrained Models）模型是北京智源人工智能研究院和清华大学发布的中文大规模预训练模型。官方发布了三种规模的模型，参数量分别为109M、334M、2.6B，用户需申请与通过审核，方可下载。
由于原项目需要考虑大模型的训练和使用，需要安装较为复杂的环境依赖，使用上也较为复杂。
本项目采用了109M的CPM模型（若资源允许也可以考虑334M的模型），并且简化了模型的训练和使用。

本项目是基于CPM模型的中文文本生成项目，可用于作文、小说、新闻、古诗等中文生成任务，并且训练和分享了[中文作文生成模型](#model_share)，取得了不错的[生成效果](#sample)。
本项目提供了数据预处理、模型训练、文本生成、Http服务等代码模块。
详情可参考[CPM模型论文](https://arxiv.org/abs/2012.00413), [CPM官网](https://cpm.baai.ac.cn/), [项目源码](https://github.com/TsinghuaAI/CPM-Generate) 。


## 运行环境
python==3.6、transformers==4.6.0、sentencepiece==0.1.94、torch==1.7.0、Flask==1.1.2


## 项目结构
用户可自行创建以下目录。
- config:存放模型的配置文件
- data：存放训练数据
- model:存放模型 
- log：存放日志文件
- vocab：
  - chinese_voca.model：sentencepiece模型
  - vocab.json:分词与id的键值对
- data_parallel.py：解决pytorch的GPU负载不均衡的问题
- generate.py：生成代码
- http_service.py：封装成http服务，支持post与get请求
- preprocess.py：数据预处理代码
- utils.py：存放一些工具代码

## 模型参数与训练细节
由于GPU资源有限，本项目使用cpm-small.json中的模型参数，若资源充足，可尝试cpm-medium.json中的参数配置。

本项目的部分模型参数如下：
- n_ctx: 1024
- n_embd: 768
- n_head: 12
- n_layer: 12
- n_positions: 1024
- vocab_size: 30000

对26w篇作文进行预处理之后，得到60w+长度为200的训练数据。显卡为三张GTX 1080Ti，batch_size=50，三张卡显存满载,一轮训练大约需要3个小时。训练40轮之后，loss降到2.1左右，单词预测准确率大约为54%。

## 使用方法
### Quick Start
在[模型分享](#model_share)中下载模型，将模型文件夹zuowen_epoch40放到model目录下,执行如下命令，指定作文标题、作文开头和长度，进行生成。
```
python generate.py --model_path model/zuowen_epoch40 --title 家乡的四季 --context 家乡的四季,最美不过了 --max_len 200
```

### 数据预处理
每篇作文对应一个txt文件，txt内容格式如下：
```
---
标题：徜徉在书籍的阳光世界
日期：xxxx-xx-xx xx:xx:xx
作者：佚名
---


一本书是一个人的眼睛，它可以让你看到另一个世界的奇妙；一本书是一个人的耳朵，它可以让你听到大自然的呼唤，听到社会的声音。

《森林报》是苏联著名科普作家维。比安基的代表作品，他以春夏秋冬四季为序，有层次、有类别地将森林里动植物的新鲜事描写得栩栩如生，引人入胜。这本书教会我们如何去观察、认识大自然。这本书教会我们感悟生命，体验生动的愉快探速之旅，激发我们热爱科学的兴趣。

《三字经》、《弟子规》、《论语》这样的国学经典，我也会拿来阅读，虽然似懂非懂，但读起来朗朗上口，觉得挺有趣。读着读着，好似开始了一场时空之旅，与古代圣贤结为知己，进行心与心之间的倾听与问候。这些书籍让我们在阅读中品味高雅。

在成长的过程中，每个人都有着自己不一样的心路历程。阳光姐姐写的《成长的秘密》一书让我们获益不浅。作者用简单生动的文字，把温馨感人、新鲜快乐、爆笑的校园生活展现在我们眼前。书中的人物宁佳鑫看上去弱小，但她实际却很坚强，在她身上，我看到了她散发出的正能量和她在逆境中奋起的精神。她的经历告诉我：无论遇到什么样的挫折与坎坷，都不要气馁。阳光总在风雨后，只要我们坚持不懈地去想办法克服困难，并付诸行动，就一定会柳暗花明！

法国作家德尔伦曾说过“智慧可以转化成力量，完成你认为不可能完成的事。”是啊，智慧的力量很强大，这些力量隐藏在书中。当我们在阅读之际，这些知识就偷偷地跑进我们的脑海里，渐渐地，渐渐地，它们就永远地保存下来，显示出无穷的魅力，让我们的未来畅通无阻。

书籍，用爱和勇气唤醒每个孩子的心灵；书籍让我们感受到温暖与力量；书籍，教我们用心灵在文字间快乐舞蹈。

让我们走进书籍的阳光世界，获取成长的力量。
```
对于每个txt文件，首先取出标题与内容，将标题与内容按照"title[sep]content[eod]"的方式拼接起来，然后对其进行tokenize，最后使用滑动窗口对内容进行截断，得到训练数据。
运行如下命令，进行数据预处理。注：预处理之后的数据保存为train.pkl，这是一个list，list中每个元素表示一条训练数据。
```
python preprocess.py --data_path data/zuowen --save_path data/train.pkl --win_size 200 --step 200
```
超参数说明：
- vocab_file：sentencepiece模型路径，用于tokenize
- log_path：日志存放位置
- data_path：数据集存放位置
- save_path：对训练数据集进行tokenize之后的数据存放位置
- win_size：滑动窗口的大小，相当于每条数据的最大长度
- step：滑动窗口的滑动步幅

用户也可以根据自身的需求，对预处理的代码进行相应的修改。后续将更新项目代码，以便用于处理各种数据集。

### 训练模型
运行如下命令，使用预处理后的数据训练模型。
```
python train.py --epochs 100 --batch_size 16 --device 0,1 --gpu0_bsz 5 --train_path data/train.pkl
```
超参数说明：
- device：设置使用哪些GPU
- no_cuda：设为True时，不使用GPU
- vocab_path：sentencepiece模型路径，用于tokenize
- model_config：需要从头训练一个模型时，模型参数的配置文件
- train_path：经过预处理之后的数据存放路径
- max_len：训练时，输入数据的最大长度。
- log_path：训练日志存放位置
- ignore_index：对于该token_id，不计算loss，默认为-100
- epochs：训练的最大轮次
- batch_size：训练的batch size
- gpu0_bsz：pytorch使用多GPU并行训练时，存在负载不均衡的问题，即0号卡满载了，其他卡还存在很多空间，抛出OOM异常。该参数可以设置分配到0号卡上的数据数量。 
- lr：学习率
- eps：AdamW优化器的衰减率
- log_step：多少步汇报一次loss
- gradient_accumulation_steps：梯度累计的步数。当显存空间不足，batch_size无法设置为较大时，通过梯度累计，缓解batch_size较小的问题。
- save_model_path：模型输出路径
- pretrained_model：预训练的模型的路径
- num_workers：dataloader加载数据时使用的线程数量
- warmup_steps：训练时的warm up步数


### 文本生成
运行如下命令，进行文本生成。
```
python generate.py --device 0 --max_len 200 --title 家乡的四季 --context 家乡的四季,最美不过了
```
超参数说明：
- device：使用哪个GPU进行生成 
- temperature:详情可参考temperature sampling的思想
- topk:top-k采样（注：topp为0，topk不为0时采用top-k采样）
- topp:核采样（注：topk为0，topp不为0时，采用核采样）
- max_len：生成的最长长度
- log_path：生成日志存放位置
- no_cuda：设为True时，不使用GPU
- model_path：模型存放路径
- title：作文标题
- context：作文上文
- context_len：每一步生成时，参考的上文的长度

### Http服务
将模型生成能力封装成Http服务，支持Post与Get请求。运行如下命令，启动服务。
```
python http_service.py --port 8085 --model_path model/zuowen_epoch40 
```
Get请求：
```
http://localhost:8085/zuowen?title="家乡的四季"&context="家乡的四季,最美不过了"&max_len=200
```
Post请求
```
localhost:8085/zuowen
{
    'title':'家乡的四季',
    'context':'家乡的四季,最美不过了',
    'max_len':200
}
```

超参数说明：
- device：使用哪个GPU进行生成 
- temperature:详情可参考temperature sampling的思想
- topk:top-k采样（注：topp为0，topk不为0时采用top-k采样）
- topp:核采样（注：topk为0，topp不为0时，采用核采样）
- port：服务绑定的端口号
- log_path：生成日志存放位置
- no_cuda：设为True时，不使用GPU
- model_path：模型存放路径
- context_len：每一步生成时，参考的上文的长度


<h2 id="model_share">模型分享</h2>

|模型 | 共享地址 |模型描述|
|---------|--------|--------|
|zuowen_epoch40 | [百度网盘【提取码:8o3v】](https://pan.baidu.com/s/1nwyqQ6WyE0mE0U6OVlThEQ) |使用26w篇中文作文语料训练了40个epoch，loss降到2.1左右，单词预测准确率大约为54%|

## Future Work
- 使用3张1080Ti进行训练，由于显卡资源有限，在数据预处理时，使用了大小为200的滑动窗口对数据进行截断，batch_size设为50。没有充分使用模型1024的最大输入长度，导致训练不够充分。若有充足的显卡资源，可以使用1024的滑动窗口对数据进行截断，提高模型的生成效果。
- 当前代码主要针对作文数据集进行数据预处理、训练、生成。后续将会更新代码,以便用于处理各种数据集。

<h2 id="sample">生成样例</h2>
以下生成样例，生成长度默认为200。

### 家乡的四季
```
title:家乡的四季
context:家乡的四季,最美不过了

result:
家乡的四季,最美不过了。家乡的四季,是令人沉醉的。
春天,万物复苏,冰雪融化,万物复苏。树枝抽出了嫩芽,花朵绽放了笑脸,树木吐出了嫩芽,春笋也破土而出,像是迎接春天的到来。小鸟们也在枝头唱起了动听的歌曲,周围的一切都变成了春的样子。
夏天,荷塘里的荷花开了,散发出阵阵清香。远处,山的颜色深浅不一,像是穿着一件翠绿的长裙,在荷塘的衬托下显得更加美,更加翠绿。微风拂过,荷花轻轻地摆动着,像是在和我打招呼呢!
秋天,

result:
家乡的四季,最美不过了。
家乡的春天,柳树发芽了,小草从泥土里探出头来,小花也张开了笑脸,小草偷偷地探出头来。我小时候,经常到那里玩,在那里捉迷藏,去田野里捉迷藏。到了晚上,爷爷便去田野里找蟋蟀,等到第二天早上,爷爷就去捉蟋蟀了。
家乡的夏天,荷塘里开满了荷花,碧绿的荷叶,荷花都开了,荷叶上还有青蛙王子,他们正在开大会呢!
家乡的秋天,果实累累,果园里更是瓜果飘香。你看,农民伯伯正忙着摘果实呢!爷爷会摘苹果,苹果熟了,

result:
家乡的四季,最美不过了。
春天,嫩芽破土而出,焕发出生机。每当春姑娘来临之际,小草就会脱下旧衣服,冲出家门,迫不及待地站在土地上,感受春风亲吻着自己的脸庞,贪婪地吸吮着甘甜的露水。春姑娘来到田野里,到处都是一片嫩绿,一派盎然的景象。柳树姑娘刚刚梳理好头发,甩动着长长的头发,伴随着阵阵春风,跳起了欢快的舞蹈。此时此刻,春雨也来凑热闹了,她滴落在溪水中,随着春风舞动起来,漾起一圈圈水纹。在河边,长满了一串串一串串鲜艳的鲜花,

result:
家乡的四季,最美不过了,四季各有特色。
春天,小草探出了它那绿绿的小脑袋,柳树的枝条随风飘动,好像正在给春姑娘梳头发。桃花、杏花、梨花争先恐后的开放,如同一个个粉红的小精灵在枝头跳着美丽的舞蹈。小燕子从南方飞来,在空中快乐的飞来飞去,非常动听。
夏天,骄阳似火,树木葱葱笼,在骄阳的照耀下,鸟儿也在树上唱着动听的歌。小孩子们穿着短袖,在大树下坐着乘凉,偶尔会出现几个小朋友在那里捉迷藏,嬉戏。
秋天,

result:
家乡的四季,最美不过了,我家乡的四季是如此美丽。
春天到了,小草从泥土里钻出来了,正东张西望地观察着四周,像是在寻找着什么。大树也绽开了笑脸,开出了许多颜色各异的花,有黄色、红色、紫色、绿色,真是色色俱全啊!花儿在春雨的滋润下,绽放出了自己美丽的花朵,散发出了迷人的芳香,那花儿就像一位位亭亭玉立的少女,娇艳迷人,美丽极了。那嫩绿的小草,铺满了大地,让我们感到生命的希望。
夏天,小草长得郁郁葱葱,到处都是绿茵茵的,走在路上,

result:
家乡的四季,最美不过了。
春天,到处充满了生机勃勃。风和日丽,万物复苏,柳树那碧绿的头发被风吹得翩翩起舞,像一个亭亭玉立的少女在对我招手。
夏天,太阳高高地挂在天空,灿烂的阳光照耀着大地,我看见农民伯伯忙碌的身影,心想:这么热的天,还要干什么?我要帮助他们干农活。我想着想着,又想到了奶奶家,我就跑到奶奶家的西瓜地里,去拿奶奶的小锄头,把小锄头递到奶奶的手里,奶奶一边干活,一边说:“你可真棒!”
秋天,

result:
家乡的四季,最美不过了。
春天到了,花儿苏醒了,小草冒出了头,树木抽出了新的枝条,燕子又飞回来了。远处,连绵起伏的高山一座连着一座,就像一座座大山。山下有一条清澈的小溪,水哗啦啦地流着,就像一匹美丽的蓝丝绸。山上的树木也抽出了新的枝条,长出了嫩绿的叶子,叶子好像一块块绿宝石。燕子从南方飞回来了,站在枝头上,叽叽喳喳地叫着,好像在唱着春天的赞歌。
夏天到了,太阳像个大火球,照着地面,我和小伙伴们经常到小溪里游泳、打水仗,

result:
家乡的四季,最美不过了。
春天,那条河流解冻了。岸边的柳树随风飘动,那优美的身姿,加上那动人的歌喉,简直是春姑娘的杰作。小燕子从南方赶来,叽叽喳喳的叫着,好像在和柳树说:“柳树姐姐,你的头发可真美呀!”小草偷偷的从土里钻了出来,那嫩嫩的、绿绿的,就像刚打过仗的战士一样。花儿们,现在都刚刚长出来,那些花,就像刚睡醒的小婴儿,那些花,有红的、黄的、紫的......五彩缤纷,美丽极了。夏天,

result:
家乡的四季,最美不过了,无论是万物复苏的春天,还是烈日炎炎的夏天,抑或是硕果累累的秋天,每一个季节都有它们的特点。
春姑娘刚走,夏姐姐就来了。太阳公公热情的烘烤着大地,知了在树上“知了知了”的叫着,仿佛在说:“好热啊!好热啊!”荷花池塘也“出淤泥而不染”,荷叶“接天莲叶无穷碧”的的清香随风飘来,整个家乡弥漫着沁人心脾的花香,让人陶醉。
春姑娘刚刚走,夏姑娘就来了。太阳像一个又大又圆的火球挂在天空中,
```

### 徜徉在书籍的阳光世界
```
title: 徜徉在书籍的阳光世界
contetx: 一本书是一个人的眼睛，它可以让你看到另一个世界的奇妙

result:
一本书是一个人的眼睛,它可以让你看到另一个世界的奇妙。
你可曾品尝过文学的魅力?
唐诗宋词,群星璀璨,给我们无尽的想象与愉悦。
“明月几时有,把酒问青天。不知天上宫阙,今夕是何年。”它描绘了一个美好的时代。苏轼在赤壁赏月时,不禁为这美景感叹。“明月几时有,把酒问青天。”它告诉了我们人生的哲理。
文学作品,不但丰富了我们的知识,也为我们描绘了一幅幅优美的山水画。
语文书中的婉约柔情,让我感受到世间的人情冷暖,

result:
一本书是一个人的眼睛,它可以让你看到另一个世界的奇妙;一本好书是一个人的眸子,它可以让你看清世界的脉络;一本好书是一把钥匙,它可以打开你心灵的窗户。我徜徉在书的世界里,在阅读中,我找到了梦想。
一本好书,犹如一泓清泉,流入我干渴的心田;一本好书,犹如一只小舟,载着我遨游在知识的海洋;一本好书,犹如一缕阳光,照亮我的心房。
记得在我很小的时候,我每天都要缠着妈妈给我讲故事,每次妈妈讲完故事,我都会依偎在妈妈的怀里,

result:
一本书是一个人的眼睛,它可以让你看到另一个世界的奇妙;一本书是一场细雨,滋润你的心田;一本书是你的拐杖,带你走进这个美妙的世界。
在我很小的时候,就开始接触书籍了,我有一个非常要好的朋友,叫做书。在我很小的时候,书还是不可缺少的。
在我不认字的时候,我就会捧着《格林童话》,开始认真地看书,我看的津津有味。《格林童话》让我明白了做人的道理,《白雪公主》让我知道了善良的重要;《卖火柴的小女孩》让我明白了人间的幸福是美好的,

result:
一本书是一个人的眼睛,它可以让你看到另一个世界的奇妙。书就像是一颗闪烁的星星,给你引航;书就像一汪清泉,给你洗涤心灵;书就像一束阳光,给你带来无穷的温暖......
我从小就喜欢读书。一个冬天的下午,我在家楼下的小广场上坐着,静静地享受着小时候的乐趣。突然,一位老爷爷从远处走了过来,手里拿着一本厚厚的《安徒生童话》,我拿起这本书,心想:这书可是我的心爱之物啊!
于是,我跑到他身边,与他交谈起来。原来,这位老爷爷就是在我六岁时,

reslut:
一本书是一个人的眼睛,它可以让你看到另一个世界的奇妙,每一本都有着不一样的内涵。
——题记
在某个宁静的午后,沉醉在书本的世界里,沉醉在阅读的魅力里,沉醉在阅读的心灵深处。
坐在一望无际的草原上,静静地读书。我像一匹饿狼,贪婪地读着,不一会儿,我就沉浸在书中。不知不觉,太阳已落下去,不知不觉,天色已晚,我们只好依依不舍地收起书本。
夕阳西下,落日把天空染成了红色,火烧云像一只只巨象,汹涌澎湃,在天空中横飞,

result:
一本书是一个人的眼睛,它可以让你看到另一个世界的奇妙之处,更能让你认识到书本的神奇之处,而我则常常沉浸在那散发着淡淡书香的阅读之中。
从幼儿园开始,我便爱上了读书,最开始是读绘本,后来爱上了古诗文。记得,一开始是爸爸妈妈带我去书店买诗。“床前明月光,疑是地上霜。举头望明月,低头思故乡。”那时,我对诗的理解是这样的:月光皎洁的夜晚,举头望明月,低头思故乡。“水光,山色空蒙雨亦奇。”那时的我对诗的理解是“帘卷西风寒鸦知人不识月,

result1:
一本书是一个人的眼睛,它可以让你看到另一个世界的奇妙。书,是一个精神世界的源泉,在我们的精神世界里,书是一位无声的老师,也是一个最忠实的朋友。
书是人们的良师益友,是精神世界的指南针,有了书,我们便知道了知识,有了知识,才会使我们做事情变得更有趣味,有了书,我们才能做更多的事。书是我的伴侣,书是我的好老师。
高尔基曾经说过:"书籍是人类进步的阶梯。莎士比亚曾经说过"书籍是全世界的营养品"

```

### 我最敬佩的一个人
```
title: 我最敬佩的一个人
context:在我的生活中,有外冷内热的妈妈,有拼命工作的爸爸

result:
在我的生活中,有外冷内热的妈妈,有拼命工作的爸爸,还有勤劳朴素的爷爷奶奶。但是,我最敬佩的是我的妈妈。
我的妈妈,有一双炯炯有神的眼睛,高高的鼻梁上架着一副眼镜,她很爱笑,很爱笑。妈妈的头发非常多,细长的柳叶眉下镶嵌着一双炯炯有神的眼睛,好像一个宝石。妈妈长的高高的个子,鼻梁上架着一副眼镜。

result:
在我的生活中,有外冷内热的妈妈,有拼命工作的爸爸,有工作努力的奶奶,有勤奋好学的姐姐......但是,我最敬佩的是那位平凡的清洁工人。
一天,我和妈妈一起乘坐公交车回家。到达了车站,我和妈妈下了车,就急匆匆地跑向家附近的早餐店。吃完早餐后,我们正准备上公交车,可是我发现有一位清洁工老爷爷正在寒风中扫地。他身穿一件单薄的衣服,衣服上沾满了灰尘,他却是用手把垃圾一点一点地扫起来,垃圾车井然有序地行驶着。我心想:这位清洁工真不容易啊!

result:
在我的生活中,有外冷内热的妈妈,有拼命工作的爸爸,有幽默搞笑的爷爷,有坚持不懈的老师......但是,我最敬佩的人是我的奶奶。
我的奶奶非常爱美,喜欢穿红色衣服。有一天,奶奶过生日,我早早地起了床,迫不及待地对奶奶说:“奶奶,奶奶,祝你生日快乐,身体健康,万事如意!”奶奶开心地说:“谢谢你,宝贝,你真是长大了。”
我的奶奶又很勤劳。她很会做家务活,做家务活,她把家里打扫得干干净净,就像一个小家一样。她不仅把家里打扫得干净,

result:
在我的生活中,有外冷内热的妈妈,有拼命工作的爸爸,有知识渊博的爷爷,但是我最敬佩的还是我的妈妈。
我的妈妈长着一头乌黑发亮的头发,又短又黑,一双炯炯有神的大眼睛,笑起来特别好看,小小的嘴巴一笑起来就露出两个甜甜的酒窝,非常迷人。
妈妈的个子比较高,还稍微有点胖,这都是为什么妈妈很胖的原因。但是妈妈每天都非常累,她是一个非常勤劳的人,每天都要很早起床,为了家人做出更多的早餐,她总是天不亮就起床,然后再叫醒我。

result:
在我的生活中,有外冷内热的妈妈,有拼命工作的爸爸,还有日夜奔波的老师......而我最敬佩的人就是我的英语老师--彭老师。
她有一头乌黑亮丽的长发,弯弯的眉毛下面是一双炯炯有神的大眼睛。上课时,她的声音很小,经常在黑板上点出枯燥的英语单词。如果我们在写字,她就用眼睛一遍一遍地望着我们,就像在注视着自己的孩子一样。彭老师十分严格。

result:
在我的生活中,有外冷内热的妈妈,有拼命工作的爸爸,还有一个吃苦耐劳的爷爷,但是最令我敬佩的是我的爷爷。
爷爷的身高已经一米七了,已经有80多岁了,他虽然已经退休了,但仍然坚持每天给我做可口的饭菜,陪我玩耍,照顾我的生活起居,而且还坚持每天接送我上下学,爸爸妈妈很爱我。
我的爷爷长着一张圆圆的脸,一双炯炯有神的眼睛,

result:
在我的生活中,有外冷内热的妈妈,有拼命工作的爸爸"我最敬佩的一个人",其中,有一个人我最敬佩。
她长着一双炯炯有神的眼睛,高高的鼻梁上架着一副黑色的眼镜,给人一种文雅大气的感觉,一张樱桃小嘴一张一合,给人一种读书的感觉,她就是我的妈妈。
我的妈妈是一个喜欢化妆的人。她每次都会把自己打扮得漂漂亮亮的,

result:
在我的生活中,有外冷内热的妈妈,有拼命工作的爸爸,但最敬佩我的哥哥。
哥哥是一个人见人爱,花见花开,车见车爆胎的卖菜小贩。哥哥的头上会扎成一个三角形,眉毛下面长着一双明亮的大眼睛。鼻子很小巧,还有一个樱桃小嘴。他的嘴巴虽然小,但是能说会道。你看,他的脸蛋上还长了一对小酒窝。
```


