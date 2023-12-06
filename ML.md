## Linking Writing Processes to Writing Quality

### Use typing behavior to predict essay quality

根据输入时的行为预测文章数量，感觉这个相对简单一些，数据集大小为470MB

- **train_logs.csv** - 用作训练数据的输入日志。为了防止文章文本的复制，所有字母数字字符输入都被替换为"anonymous"字符(即`q`)；标点符号和其他特殊字符没有被匿名化。
  - `id` - 文章的唯一ID
  
  - `event_id` - 事件的索引，按时间顺序排序
  
  - `down_time` - 按下事件的时间（毫秒）
  
  - `up_time` - 松开事件的时间（毫秒）
  
  - `action_time` - 事件的持续时间（`down_time`和`up_time`之间的差值）
  
  - `activity` - 事件所属的活动类别
    - `Nonproduction` - 该事件不会改变文本
    - `Input` - 该事件向文章中添加文本
    - `Remove/Cut` - 该事件从文章中删除文本
    - `Paste` - 通过粘贴输入，该事件改变文本
    - `Replace` - 该事件用另一个字符串替换文本的一部分
    - `Move From [x1, y1] To [x2, y2]` - 该事件将跨越字符索引`x1`到`y1`的文本段移动到新位置`x2`到`y2`
  
- `down_event` - 按下键盘/鼠标时的事件名称

- `up_event` - 松开键盘/鼠标时的事件名称

- `text_change` - 事件导致的文本更改（如果有）

- `cursor_position` - 事件后文本光标的字符索引

- `word_count` - 事件后文章的字数

**请注意，测试集中可能存在训练集中不存在的事件。您的解决方案应对未见事件具有鲁棒性。**

注意：按下和松开键盘事件的顺序不一定与数据集中呈现的顺序相同。举个例子，一个作者可以在释放"a"之前按下"b"。然而，在数据帧中，所有关于"a"的按键信息都出现在"b"之前。

- **test_logs.csv** - 用作测试数据的输入日志。包含与`train_logs.csv`相同的字段。在此文件的公共版本中提供的日志仅是示例，用于说明格式。

- **train_scores.csv** - 文章的得分数据，用于训练模型。

  - `id` - 文章的唯一ID
  - `score` - 文章的得分，范围为0到6（竞赛的预测目标）

- **sample_submission.csv** - 符合正确格式的提交文件。有关详细信息，请参阅[**Evaluation**](https://www.kaggle.com/competitions/linking-writing-processes-to-writing-quality/overview/evaluation)页面。

请注意，这是一个[**Code Competition**](https://www.kaggle.com/competitions/linking-writing-processes-to-writing-quality/overview/code-requirements)。我们提供了一些示例日志，以帮助您编写解决方案。当对您的提交进行评分时，这些示例测试数据将被完整的测试集替换。测试集中大约有2500篇文章的日志。

目前的一个思路是完全创建出文章再对文章进行分析



trainlog变量中各列的列名以及其含义：

- id - 文章的唯一ID
- event_id - 事件的索引，按时间顺序排序
- down_time - 按下按键的时间（毫秒）
- up_time - 松开按键的时间（毫秒）
- action_time - 事件的持续时间（down_time和up_time之间的差值）
- keyword - 按键的类型：共有keyboard_keyword，media_keyword，other_keyword，unknown_keyword这4种取值
- delta_cursor - 每次行动后光标移动的距离
- movedis - Move操作中移动的距离
- movelength - Move操作中移动的字符串长度
- pastelength - 一次黏贴的字符串长度
- replacelength - 一次replace操作替换的字符串长度差值
- RClength - 一次Remove/Cut操作删去的字符串长度
- thinktime - 思考时间，定义为上一个按键释放到下一个按键释放的时间
- contype - 标记是否为连打，连打的定义即上一个按键还未释放下一个按键就已经按下了
- actype:操作类型，有0~5共6种取值
      0:Nonproduction,
      1:Input,
      2:Remove/Cut,
      3:Paste,
      4:Replace,
      5:Move

数据处理思路：

1. 创建一个新的dataframe变量traindata

2. 以id作为标识遍历trainlog，相同id的行为一个文章写作时的信息，对于每个文章的所有相关信息，作如下处理

   1. 记录文章id，记为id
   2. 统计总单词数，即最后一行的word_count的值，记为word_count
   3. 统计连打次数，即有多少行的contype值为1，记为contype_count
   4. 统计思考总时长，即所有行的thinktime值之和，记为total_think
   5. 统计总用时，即最后一行的up_time减去第一行的down_event，记为total_time
   6. 统计所有操作数量，即有多少行，记为action_count
   7. 统计所有操作总用时，即所有行的actiontime值之和，记为total_actiontime
   8. 统计keyword列中4种数值各自出现的次数，分别记为keyboard_count，media_count，other_count，unknown_count
   9. 统计delta_cursor列中所有大于等于零的数值中在0~1,2~10,11~30,31~80以及大于80这5个区间内的数目，分别记为dc0,dc1,dc2,dc3,dc4；统计delta_cursor列中所有小于零的数值在-1~-1,-2~-10,-11~-30,-31~-80以及小于-80这5个区间内的数目，分别记为dc5,dc6,dc7,dc8,dc9
   10. 统计movedis列中所有数值在0~1,2~10,11~30,31~80以及大于80这5个区间内的数目，分别记为md0,md1,md2,md3,md4
   11. 统计movelength列中所有数值在0~5,6~15,16~30,31~80以及大于80这5个区间内的数目，分别记为ml0,ml1,ml2,ml3,ml4
   12. 统计pastelength列中所有数值在0~1,2~10,11~30,31~80以及大于80这5个区间内的数目，分别记为pl0,pl1,pl2,pl3,pl4
   13. 统计replacelength列中所有数值在0~1,2~10,11~30,31~80以及大于80这5个区间内的数目，分别记为rl0,rl1,rl2,rl3,rl4
   14. 统计RClength列中所有数值在0~1,2~10,11~30,31~80以及大于80这5个区间内的数目，分别记为rc0,rc1,rc2,rc3,rc4
   15. 统计actype列中6种数值各自出现的次数，分别记为ac0,ac1,ac2,ac3,ac4,ac5
   16. 统计连续输入次数，连续输入指的是连续的多行，其actype的值都为1，而这些行组合的前后两行其actype值都不为1
   17. 对于上一条定义的连续输入，统计连续输入长度，即每个连续输入的行数，对于所有统计出的行数，统计其最大值、均值、中位数以及方差
   18. 将以上的所有信息存储到traindata中，列名即为“分别记为”中设定的名字

   
