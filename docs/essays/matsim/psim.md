### 仿真的效率魔咒
在各类交通仿真软件中，制约整体运算效率提升的关键「短板」是网络加载（network loading）过程。因此，整体计算性能的提高主要通过多个中央处理单元或多个核心（计算节点）来实现。在MATSim一次迭代中，尽管类似于Routing/Scoring/Innovation这样的任务可以直接分布在多个计算节点上，这一方式对于网络加载来说却并不适用。原因是在仿真过程中，物理系统是紧密关联和相互影响的，要保证仿真系统的高效和准确运转，所有车辆的仿真状态同步必须要满足尽可能小的时间间隔（如vissim中，默认车辆仿真运行状态在1s内会刷新100次，对应同步时间间隔为0.01秒），否则在微观上可能导致车辆之间状态冲突，同时影响整个系统的稳定性（就算不发生撞车，因为状态同步的问题导致部分车辆之间异常靠近或疏远，因此产生非常规加减速动作），而相比于单CPU完全基于内存的计算，并行机制下IO和对齐开销过大，不足以支持快速高频次的全局状态同步。在这种情况下，可以考虑结合相互作用最薄弱的边界进行空间分解——尤其当网络链接足够长时，可能允许同步延迟稍长——从而实现一定程度的并行。然而在软件工程方面，网络加载的并行实现很难保持稳定，使其更加稳定会影响性能。可以说，对于network loading，在仿真精细程度不变的前提下，**并行计算-IO效率-仿真过程稳定性构成了不可能三角**，MATSIM中使用的基于队列的模拟（QSim）也不例外。

### 替代方案——加速方案进化的伪仿真器PSIM
在MATSim的co-evolution机制中，每轮迭代时的agent方案的「突变」是随机的。对于系统而言，优秀的方案（基因型）在通过网络加载过程完成「表达」后，通过评分机制有更大概率被保留，反之则被逐渐淘汰。仿真加载的终极目的是为方案提供用于评估的可靠信息，既然无法在仿真环节提升效率，那么能否通过减少加载的次数，采用替代方式来「预测」方案得分，通过加速优胜劣汰的筛选过程，为系统进化提供加速器呢？
MATSim的开发研究人员Pieter J. Fourie, Johannes Illenberger, 和Kai Nagel在这一设想下开发了PSIM（Pseudo Simulation）模块。该模块的引入大大 提升了方案改进效率，从而使得系统只需通过不到    %的加载次数就能获得同等的表现评分，极大提升了模型收敛速度。
![Operation of a MATSim run implementing pseudo-simulation](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1664156783099-faf2540f-aa9c-4c77-84a2-2a6f1efd2040.png "Operation of a MATSim run implementing pseudo-simulation")
#### MATSim事件
在MATSIM中，QSim输出有时间戳的原子信息单位，称为事件。通过对这些事件的检索，可以重建每个agent在交通系统中的轨迹，以及他们在各个活动地点花费的时间。
#### PSim运行机制
PSim从最近一次网络加载结果中获取网络的分时段平均出行时长信息——从QSim事件流（event flow）中，我们可以获取出每个agent在每个环节上的时间消耗，从而计算每个路段不同时间片的平均旅行时间，并将这些值存储在一个查询表中。PSim桥接了replanning模块，更新后的方案不再直接输出给QSim用于网络加载，而是传递给PSim，PSim模块从分时段查询表中读取方案路线中每个环节的可能消耗的时间，从而构建了「预估」的事件流。它将这个事件流传递给评分模块，来为新的计划产生一个预估分数，完成评分的计划重新进入了agent的plan set中。在重复多次后，plan set中的可选方案将达到上限，每次迭代结束后，得分最差的计划被丢弃。
PSim可以视为QSim的一个代理模型（surrogate model）。系统不再是每轮迭代均从基于QSim的队列模拟中获取信息来实现进化，而是从它的简化表示中预测模拟结果，获取预估信息，来完成方案的比选。在经过几次PSim迭代后，agent的方案会再被传回QSim进行网络加载，以更新的出行时间查询表。在PSim中，不同的agent之间不再有物理上的影响关系，因此不再需要考虑线程之间的同步，从而最大程度地压榨计算机并行潜能。
由于QSim总是需要全套的plan进行加载，而在PSim中agent之间没有互动，所以只需要对新产生的尚获得评分的计划进行模拟，这就进一步减少了预期的计算负荷，因为每次迭代只产生少量的新计划（这取决于重新规划策略所规定的重新规划占比）。


### 案例测试——PSIM提升功效

![Average executed score versus QSim iterations for two ratios of QSim:PSim, compared with reference QSim-only run (black line) (all simulation runs have total replanning rate of 0.3).](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1664442551518-bfe2b528-e32b-4cd8-88a7-9b69fecd0f4f.png "Average executed score versus QSim iterations for two ratios of QSim:PSim, compared with reference QSim-only run (black line) (all simulation runs have total replanning rate of 0.3).")

![Score evolution versus time for large-scale scenario, comparing influence
 of QSim:PSim ratio, number of computational cores, and replanning rate.](https://cdn.nlark.com/yuque/0/2022/png/28348597/1664442584392-92d62493-e01c-4631-a194-dc90d8f0e697.png#averageHue=%23cacac2&clientId=u6c4c3d1e-482e-4&from=paste&height=354&id=ufb4d2b86&originHeight=921&originWidth=966&originalType=binary&ratio=1&rotation=0&showTitle=true&size=149977&status=done&style=none&taskId=u697c6c9d-1c22-4acc-ba57-6b2318af650&title=Score%20evolution%20versus%20time%20for%20large-scale%20scenario%2C%20comparing%20influence%0D%20of%20QSim%3APSim%20ratio%2C%20number%20of%20computational%20cores%2C%20and%20replanning%20rate.&width=371 "Score evolution versus time for large-scale scenario, comparing influence
 of QSim:PSim ratio, number of computational cores, and replanning rate.")

  ![Computation time contributions versus number of cores for QSim only (0.3 replanning rate), QSim:PSim = 1:9 (0.3 replanning rate), and QSim:PSim = 1:24 (0.1 replanning rate) at reference score (gray line in Figure 3).](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1664442711329-fbbaf0e4-3c18-4e5d-988a-14c529f4aa40.png "Computation time contributions versus number of cores for QSim only (0.3 replanning rate), QSim:PSim = 1:9 (0.3 replanning rate), and QSim:PSim = 1:24 (0.1 replanning rate) at reference score (gray line in Figure 3).")


## 配置和使用
1、更新依赖
在POM中增加对应版本依赖（注意PIM版本需要和MTASIM版本对应，否则可能冲突无法运行）
```
		<dependency>
			<groupId>org.matsim.contrib</groupId>
			<artifactId>pseudosimulation</artifactId>
			<version>12.0</version>
		</dependency>
```

