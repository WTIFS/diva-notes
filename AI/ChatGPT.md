ChatGPT 其实本质上非常简单，它没有理解你的问题，也没有去真正查阅什么资料来回答你，它的本质就是一个词语接龙。而且说词语接龙还高看它了，它实际上就是一个字一个字往下接。

比如它先输出一个"很"字，然后它根据"很"作为上下文，计算出下一个字是"高"。

然后它又用"很高"作为上下文，计算出下一个字是"兴"。以此类推，直到输出一段完整的话，比如"很高兴见到你"。

所以它实际上并没有理解你的问题，也没有构思出回答你问题的解答思路，它就是一个字一个字地往下猜，直到拼接成一句话或一段话。



它是怎么做到一个字一个字往下猜，却能如此像人说的话呢？一个是模型，一个是数据，数据是用来训练模型的，这个大家都知道。

而训练过程分为三个阶段，第一阶段是投喂给它大量杂乱无章的信息，这叫做**无监督学习**。第二阶段是给它一些规范的对话范本，这叫做**监督学习**。第三阶段是让他自己回答问题，然后人们给出每个答案的评分，让它根据评分再次调整模型参数，这个叫做**强化学习**。

无监督学习虽然可以吸收海量的知识，但回答问题的时候可能会少了点规矩，因此需要监督学习给它提供格式规范的回答范本。但这些都是投喂固定的信息，可能导致少了创造性的答案，因此需要强化学习来让 AI 看起来真正掌握这些知识，可以举一反三，但本质上其实还是投喂信息，只不过这个信息源是 AI 自己生成的。

所以它的回答，只是根据给它投喂的材料训练出的模型计算出来的，根本没有所谓的数据库，也不会去查阅哪些资料。



数据量和模型参数的数量，在一定程度上决定了 AI 的上限。下面是 GPT-1 到 GPT-3 的数据量和模型参数。

GPT-1  5GB  1.17亿
GPT-2  40GB 15亿
GPT-3  45TB  1750亿

最新出来的 GPT-4 的模型参数据说达到了 10 - 100 万亿。



当数据量和模型参数比较少的时候，你还能看出它是根据已有的资料强行训练出的结果，但现在的数据量和模型已经只能用超大模型来描述了，模型的参数已经大到让人无法想象，也无法真正理解模型的计算结果。

所以看似产生了学习新知识的能力，但其实仍然是模型根据已有的信息训练出的结果，在数据量足够大的时候，产生了质变，似乎产生了智能。