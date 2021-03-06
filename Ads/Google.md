# 谷歌的广告业务是如何赚钱的

先来看一个公式：

$$
CPM = coverage * depth * CTR * CPC * 1000
$$

公式的最左边是 **CPM**，意思是 **每千次展示所产生的收入**，**CPM** 越高，意味着谷歌赚的钱就越多。

**Coverage** 指的是 **广告覆盖率**。广告覆盖率 = 出现广告的搜索页面 / 所有搜索页面。

**Depth** 指的是 **平均每页广告数**。即谷歌页面最顶端、最有价值的广告位，如果这个数字是3，那一个页面上出现的广告条数就是3条。

**CTR** 指的是 **点击率**。

**CPC** 指的是 **每次点击产生的费用**。



现在问题来了。我们要提高 CPM（也就是要赚更多钱），应该去提升哪些变量？

许多搜索引擎的选择是头两个。因为CTR（广告点击率）取决于用户，CPC（每次点击产生的费用）取决于广告主，这两者都难以控制，但 coverage（广告覆盖率）和 depth（广告显示条数）很容易提升。

但这也导致用户看到的广告大大增多。如果这些广告是无用的，甚至是负面的，那它会极大地干扰用户获取正常的信息。

同时，这四个变量也不是互相孤立的。举个例子，不管你搜什么词，我都给你推送一堆安全套广告，那点击率一定会下降。同时，广告主发现投了广告没什么用，他们的付费意愿也一定会下降。久而久之，就没人上你的网站了，也不会有人愿意投广告了。



**所以，谷歌选择严格控制coverage和depth，而去从更难提升的CTR和CPC入手。**

谷歌的coverage大约是三分之一，有三分之二的页面是一个广告都没有的。Depth是1-3，每个页面上展示的广告条数是1-3条。这个数值远远低于其它搜索引擎。

我们希望通过技术达到这样的效果：每展示一条广告，用户就会去点击一条，并且点开后会觉得，哎呀这正是我需要的东西。同时，广告主会发现在谷歌投广告非常高效，转化率很高，他们自然也更愿意付费。

于是，我们在早期做了许多开拓性的工作，比如将机器学习运用于广告系统，去预测和分析用户的行为。在很多时候，页面上显示的广告，甚至比自然搜索结果更匹配、更有用、质量更高。

比如，我想要搜索“去旧金山的廉价机票”，自然搜索的结果中会有trip advisor之类的攻略，告诉你可以去哪个地方买廉价机票。但是广告里直接展示了一家航空公司的打折机票——而这真的就是用户想要找的机票。

在我的印象中，谷歌综合的CTR是 3%-4%，其中TOP位的广告点击率高达30%-40%。在业界，这个数字比别家要高一个数量级。



#### 接下来说CPC。我们究竟应该对广告主采取怎样的收费策略呢？

谷歌用的不是公开竞价模式，而是 **Generalized Second Price（GSP，广义第二出价，二价）** 模式。

公开竞价可能导致哄抬广告位价格，引发恶性竞价。另一方面，它会导致某些冷门广告位的贱卖。如果是公开竞拍，广告主发现这类广告位没有人买，他们就会以非常便宜的价格（比如一分钱）去买下来。

而第二私密竞价模式则不一样。你出价的时候，并不知道别人的价格是多少。大家一起来买这个广告位，时间到，一起举牌，但只有谷歌会看到价格。如果出价的人分别举：10万、20万、100万，谷歌就会告知出价100万的广告主，你赢了，但你需要支付的不是100万，而是21万——也就是在第二高的价格上，再加一个最小单位。

这个方式非常高效，它会激励所有人出一个自己心目中的合理价位。并且，最后的成交价总是会低于赢家能够承受的最高价格。



#### 广告质量

最后还要解决一个问题。如何防止财大气粗的广告主垄断广告位？比如，有一个卖医疗用品的广告主，他就是想通过购买热门词汇，让自己的广告在搜索页面里铺天盖地的出现。如果让他得逞，那用户搜“杯子”，出现的是医疗用品，用户搜“奥运”，出现的也是医疗用品。

谷歌引入了广告质量得分，它将决定你在广告栏上的最终排名。而 CTR 点击率越高的，广告质量得分就越高，排名也就越高。

我们再来看一个公式：

`广告排名分数 = 最高点击成本 * 广告质量得分`

广告排名分数越高，这条广告的排名就越靠前。这样即使一个客户出价很高，但他的广告点击率很低，也会排在别人后面。

这意味着，相关性越强的、质量越高的广告，就能获得越低的价格，在谷歌也越有竞争力。也就是说，当广告主不断优化自己的广告质量，提高 CTR 以后，他就能出更少的钱，获得更高的排名。

靠着这个机制，优质的广告主得到了更高的转化率，因此也更有在谷歌投广告的意愿；与此同时，用户也看到了更好、更有用的广告。





#### 参考

[郄小虎Tiger - 谷歌的广告业务是如何赚钱的](https://www.zhihu.com/question/32221970/answer/119083085)

