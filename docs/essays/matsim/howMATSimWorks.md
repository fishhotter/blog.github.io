### 前 言
宏-中-微融合作为交通建模仿真发展的大趋势，在应用领域早已形成了业内共识。然而，受制于宏观静态交通分配和微观动态路径求解+仿真在底层数学与工程实现上的巨大差异，以及微观模型在算力、精度约束下，无法实现大范围建模，在这样的背景下，要达到不同城市建模尺度和粒度下模型的完美融合是一项理论上可行但工程上不可能完成的壮举。
尽管如此，行业软件商仍在可行的空间内推动弱耦合一体化模型的软件化，总部位于西班牙的交通软件公司[Aimsun](https://www.bing.com/ck/a?!&&p=6d90f4a78633155bJmltdHM9MTcwNzc4MjQwMCZpZ3VpZD0yMTYyZmViNi1lMGQ5LTYyMDktMzcxZi1lZDdiZTE5YTYzMWEmaW5zaWQ9NTIxMA&ptn=3&ver=2&hsh=3&fclid=2162feb6-e0d9-6209-371f-ed7be19a631a&psq=aimsum&u=a1aHR0cHM6Ly93d3cuYWltc3VuLmNvbS8&ntb=1)旗下的Aimsun Next软件引入了宏-中/中-微一体化的功能，在行业内独树一帜。德国老牌软件商PTV也在旗下的VISSIM软件中提供了基于简化交通流模拟的仿真功能，以实现更大范围的仿真能力。德国宇航局（[German Aerospace Center - DLR](https://www.bing.com/ck/a?!&&p=63caa62721c7ca1bJmltdHM9MTcwNzc4MjQwMCZpZ3VpZD0yMTYyZmViNi1lMGQ5LTYyMDktMzcxZi1lZDdiZTE5YTYzMWEmaW5zaWQ9NTE5OQ&ptn=3&ver=2&hsh=3&fclid=2162feb6-e0d9-6209-371f-ed7be19a631a&psq=%e5%be%b7%e5%9b%bd%e5%ae%87%e8%88%aa%e5%b1%80&u=a1aHR0cHM6Ly93d3cuZGxyLmRlL0VOL0hvbWUvaG9tZV9ub2RlLmh0bWw&ntb=1)）研发的开源微观交通仿真软件SUMO也提供了相似的插件。由于行业对模型精细化、一体化要求不断提升，掌握大规模精细化仿真建模的能力，等同于把握了交通模型领域的时代脉搏。
基于传统交通流理论的宏观模型在ABM建模的精细化需求面前缺乏微观洞察力，学界需要一种更接近“微观”的方式进行宏观建模。但在算力约束下，现有纯微观的仿真过程在短时间内无法实现。因此，如果能够通过“类微观”的方法，在微观层面上赋予模型更多的精细刻画空间，又不超出大规模仿真下的算力约束——用仿真的思维来运行宏观模型，在进一步提升宏观模型精细化应用潜力的同时，搭建宏-中-微一体化融合的桥梁。
MATSim的诞生回应了这一思想，它以一种“pseudo-micro”的方式运行。在传统微观模型中，仿真状态可以在一秒之中更新多次，车辆或人的位置是“实时”更新的。在MATSIM中，仿真状态可以采用事件驱动（Event Based）的更新方式，即只在关键的时间点（开始活动|结束活动|上下车等）更新状态，因此避免了微观仿真采用的step-by-step模式解算中间态带来的运算负载。
同时，通过模块化的软件框架，MATSim能面向不同的建模和功能应用需求，调用不同的开源模块，从而赋予了模型灵活的定制空间和扩展能力。在MATSim的生态中，越来越多封装了先进算法的模块在不断被引入，开源体系内部的“协同进化”有着强大的生命力，在技术迭代升级层面，是的MATSim具备目前同类商业软件难以比拟的优势。
![image.png](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1670209477268-5694f7c8-55bf-4326-9f52-46b306db2a73.png)
**图 1 MATSim模块化框架**
### MATSim运行框架
MATSim本身并不包含原始的活动需求模型，而是在给定的计划（Plan）基础上提供仿真。（这里的需求不一定指宏观模型如ABM的全方式需求，也可以只面向公交系统、城市道路。）需求以PLAN的形式作为MATSim的输入，包含了每一个Agent日常活动链的详细描述（开始和结束活动的时间，中途使用的方式以及过程中其他的关键时间节点）。MATSim基于输入的PLAN来进行后续的仿真流程。
![image.png](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1670209477711-0112ad78-4990-44a2-9dba-24ee5ed77e26.png)
**图 2构成PLAN的事件**
MATSim仿真的三大核心环节是：网络加载（network loading）；评分（scoring）和方案更新（innovation）。这三个过程会不断循环，直到模型到达“准稳定”态或设定的循环次数。
![image.png](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1670209477132-1b885a4f-465f-4150-ac5e-f4c97c140074.png)
**图 3 MATSim三大核心模块循环流程**
### 核心模块原理浅析
MATSim采用了mobsim->scoring->replanning的循环流程，其中mobsim可以理解成网络加载，加载完成后scoring模块会对agent的活动进行打分，这一分数会作为replanning模块的输入，在该过程中对agent的活动列表进行裁剪（plans removal step）、优化（innovation step），并最终完成全部agent在下一轮loop中的活动选择（choice）。
#### MOBSIM
MATSIM中的交通流模型集成在MOBSIM模块中，Mobsim（mobility simuation）在MATSIM中扮演了路网上agent的分配者角色。通过mobsim可以产生事件（events），这正是ABM中ACTIVITY的核心元素，也是构成matsim的基石。Mobsim目前有多个子模块可选，默认是QSIM（queue based simulation）。
QSIM专为大规模场景而设计，它采用了基于队列的高效算法。从上个交叉路口进入路段的汽车会被添加到排队队列的尾部。并在这里持续排队，直到同时满足以下三个条件，车辆才可以进入到下一个路段：

- 经过了目前所在link一个完整的free flow time周期；
- 车辆已经位于当前队列的最前端；
- 下一个路段允许进入；

![image.png](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1670209477695-4e8d4943-754e-4e79-9c69-8c4260381591.png)
**图 4 MATSim中基于排队的交通流理论**
#### SCORING
如何实现Scoring？我们天然地可以联想到计量经济学中的效用。MATSim采用了Charypar-Nagel Utility Function来作为活动链的“得分机制”。
Nagel效用方程（Splan）由两部分组成，S act,q代表活动本身带来的（正向）效用，S trav,mode(q)则体现了参加活动“在路上”的效用损失（disutility）。N是一天中的活动数量（由0到N-1）。
![](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1e093ac3cc52966d428623cad6820cc1.svg)
在MATSim中，活动持续时长是可以进行调整的，而调整目标则是最大化当天的栋效用。MATSim引入了经济学中的两个经典假设：1、边际效用递减。活动产生的效用边际增量会随着活动时长的增加逐渐衰减（但始终为正）；2、理性人考虑边际量。当agent一天要参加的活动以及出行过程中的时长确定的情况下，出行构成最终都会达到这样一种平衡态——这一天中的每个活动对效用的边际贡献，在活动结束的时刻点是相等的。因为假设不相等，那么agent就有动机去调整各个活动的时长，以最大化效用收益。
![image.png](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1670209478427-a5b14af1-9f31-456e-a3f7-5bb2aa26c00e.png)
**图 5 MATSim中随着AGENT活动的效用变化示例**
#### REPLANNING
在MATSim中，当完成当前一轮scoring后，模型开始对下一轮中agent的activity进行调整。在建模术语中，这一过程分为两大部分：选择集生成（choice set generation）和选择（choice）。MATSim中通过策略模块（Strategy modules）来实现这一过程。
在MATSim中，每个Agent都对应着一个构成日常出行的选择空间，在这个空间的基础上，仿真实质上构建了一个子空间的实例（有限的选择集），agent在每轮迭代中都会从这个动态的选择集中，结合Monte-carlo方法抽取一个具体的方案用于网络加载和评分，该选择集会在replanning阶段发生“变异”，这个变异过程允许对出行路径、出行时间、出行方式的全方位调整。在完成调整后，MATSim使用选择器（selector）抽取下一轮循环中该agent实际采用的方案。
![image.png](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1670209478495-556509a1-e9fd-4a8d-b918-5fc3d7f55a68.png)
**图 6 MATSim中replanning的结构**
### 技术应用前景
#### 极具细节的方案评估
基于Agent仿真可以输出仿真过程中全部事件（行为）的关键时间，与现有方式相比，具备两方面的优势：1）可结合agent进行多样化的动态场景呈现；2）评估指标范围和时间颗粒度极大提升，如可评估拥堵或事故对道路通行能力的影响随时间的变化情况，也可评估收费措施对agent路径选择以及出发时间的变化影响。
#### 进一步拓展和融合现有专项模型技术
基于MATSim可以和现有轨道公交模型、碳排、货车模型等无缝衔接融合，同时结合开源算法库，持续提升模型性能。同时，未来还应探索MATSim和城市三维仿真的融合，实现更佳的视觉展示效果。
#### 为后续自主开发宏中微融合套件完成技术储备
在软件层级，MATSim具备真正实现宏-中-微一体化的巨大潜力（matsim的仿真模式本质上更接近中观，或者成为“pseudo-micro”）。在微观层面未来可以进一步和SUMO对接，不同层级模型输出结果可以向上反馈，完成多层级的模型-仿真推演闭环。
