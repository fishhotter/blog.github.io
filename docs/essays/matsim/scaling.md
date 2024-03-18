前言：本文结合《Effects of scaling down the population for agent-based traffic simulations》中研究内容进行梳理。主要包含：1、仿真缩放参数的选取和计算；2、不同仿真缩放参数和模拟次数下的结果对比；3、不同精度网络对仿真结果的影响；4、scaling对仿真效率的提升效果

尽管MATSim拥有较高的仿真效率，在面对百万乃至千万量级大规模出行的仿真时，仍需要消耗非常多的时间和算力。因此，对仿真进行合理**「缩放」**可以在对仿真结果不产生明显影响的前提下大大提升仿真速度，以及为城市网络细节的进一步完善提供宝贵空间。
**「缩放」**的本质是对于输入的Agent数量和模型网络的容量/通行能力进行同步缩减。其中Agent数量缩减主要采用抽样（subsample）的方式，在原有出行中随机抽取一定比例作为输入。
MATSim中采用了基于排队的仿真方式，对通行能力的定义主要包含两个方面：1）link车流的通过能力，即单位时间内有多少辆车可以离开当前路段；2）link的容量，即该link下最多同时能够容纳多少辆车。
![image.png](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1661223057554-01b4e66a-0f46-40cc-a57c-1918b1bec1ff.png)
上图中记录了在不同仿真次数下，抽样率5%和100%时TTD的差异。随着仿真次数增加（500次），出行时间分布曲线达到了高度一致性。

![image.png](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1661223473416-06176f2a-016d-4127-91a5-6d48b1f574b6.png)
上图中，精细化网络的路段数量是粗糙路网的3倍，说明精细化的网络并不会对仿真效率造成明显影响。
![image.png](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1661223613134-ddee3d83-76f1-45ab-95a8-8ba7c17b7bde.png)
上图中，（a）在两种网络精度下不同的缩放比对平均出行时间产生的影响。当网络更精细时，平均出行时长随抽样率变化的波动会更小。
MATSim中的scoring机制可以用于评估模型迭代的稳定性（或作为模型收敛指标）。下图展示了5%抽样和全样本下模型评分的变化情况。5%的抽样率下模型基本在100次迭代后评分基本达到稳定，而全样本模型在350次迭代后评分仍然保持了向上的斜率。（注意在这些实验中后30%的迭代不再允许生成新的plan，因而从第二张图开始，全样本仿真的评分最后一段均可以达到平稳，这并不代表模型实际的收敛水平）

![image.png](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1661245239783-e6ddc0e0-9690-4717-96a0-2f87fb4f20b9.png)
