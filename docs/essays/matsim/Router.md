「Personal understanding & questions」
Router是中-微观仿真中用于路径规划的核心工具，在车辆或行人加载进入模型网络前，Router会为每个agent指定一条确定的路径，因此router求解速度实际很大程度上影响了仿真在iteration之间的处理时间（所以router实际是在一个iter之前就已经做完功课？还是在实时评估路网的状态？中观和微观之间是否有区别？理论上，schedule based 肯定是提前评估好全部路线，但对于小汽车的loading，路网状态是随时间变化的，中微观之间是否存在差异，微观是实时的，中观matsim是预先计算好的？）

## 1  PT ROUTER
### 1.1  SBPTR & EBPTR
MATSim目前使用了一套SBPTR来进行路径规划

在公共交通路线选择中，特定用户的决定和行动不仅取决于他/她自己的偏好，例如时间价值、人群回避或支付意愿。它们还取决于许多其他公共交通用户、运营商和当局的决定和行动。甚至私人交通用户的决策也参与其中，因共享了相同的基础设施。

如上所述，MATSim 目前使用了 SBPTR，这意味着当agent需要给定开始时间、起点和目的地的路线时，SBPTR 会在基于时间表的网络中找到最短路径（假设公共交通工具总是准时准点到达）。在实际动态仿真中，车辆可能会提前或迟到和/或可能已满，因此不允许额外的乘客上车。如果乘客无法按照理想方式乘坐交通工具，则agent获得了不好的分数，在迭代学习过程中，原有计划可能会被更有利的计划所取代。当agent试图找一条新的路线，起始时间、始发地和目的地相同，公交预定网络最短路径保持不变；代理商无法通过更改路线来改善他们的体验。

为了解决这个缺点，一种新的 EBPTR（基于事件的公共交通路径求解器）诞生了。它以给定的时间表作为第一次迭代的基础，但在后续迭代之间继承和更新了有关旅行时间、公共交通车辆占用率和等待时间的信息。因此，当执行同一天的计划时，EBPTR可以在相同的开始时间、起点和目的地的条件下生成新路线，因为系统会记住延迟的公共汽车服务（更长的旅行时间），或车辆满载的火车服务（更长的等待时间）。

然而，用于为agent提供路径的网络需要一个新的拓扑来解存储这些变量。这种方法可以解释出现的现象，在过度拥挤，车辆无法正常上车的情况下，一些agent会首先反向乘坐几站。然后转移到具有足够容量可以正常登车的目的地方向车辆上去。尽管这种方式消耗了更多内存，由于采用了更简单的网络拓扑，当执行最短路径计算时，EBPTR可以实现相似甚至更好的计算时间。此外，该算法下系统达到用户均衡需要的迭代次数也明显减少了。


### 1.2  SWISS RAIL RAPTOR
#### 1.2.1  介绍
SwissRailRaptor是一个快速公共交通路径求解器。它基于Raptor算法 (Delling et al, 2012, Round-Based Public Transit Routing)，并应用了一些优化，即在选择中，需要添加哪些传输，哪些传输可以在不影响路由过程结果的情况下被忽略。

SwissRailRaptor实际性能提升因方案而异，但与MATSim中的默认pt路径求解器相比，通常在一到两个数量级之间。当应用于瑞士完整的公共交通时刻表时 (包括火车、公共汽车、电车、轮船、缆车等)，SwissRailRaptor比MATSim的默认matrix based pt路径求解器快95倍。在模型较小的情况下，SwissRailRaptor的速度可能快20-30倍。SwissRailRaptor的内存消耗也比原有的pt求解器至少降低一个数量级，初始化求解器的预处理时间也大致如此。默认情况下，SwissRailRaptor可以作为MATSim中包含的pt求解器的嵌入式替代品，当它在没有进一步配置的情况下使用时，可以重新使用默认值中的配置参数`transitRouter`配置组。一个特殊的配置组允许配置在SwissRailRaptor中配置MATSim的默认pt router不包含的高级功能 。与MATSim中默认router的主要区别在于，SwissRailRaptor在搜索路线时，在0点以后不会重复公交时刻表，只考虑时间表中规定的实际出发时间。这是因为并非所有的时间表都有24小时的周期。当应用于MATSim时，它可能导致agent在深夜离开时不再找到路线。考虑到这些agent无论如何都会被困在模拟中，因为那个时候没有预定的pt车辆在运行，这应该不会造成任何真正的问题。

#### 1.2.2  部分高级功能
SRR支持一些更加高级的功能，可以在配置文件中进行设定。
#### Intermodal Access and Egress「多模式接驳」
SRR默认的公交接驳方式为步行，但也可以指定采用自行车等方式完成两端的接驳。在这种前提下，算法对潜在站点的搜索半径可以进一步扩大（如bike的接驳半径可以高达3km）。同时，为了在增加搜索半径时减少潜在的起点和目的地停靠点的数量，SwissRailRouter 允许根据站点属性对可能的站点进行过滤。
```xml
<module name="swissRailRaptor">
  <param name="useIntermodalAccessEgress" value="true" />
  
  <paramset type="intermodalAccessEgress">
    <param name="mode" value="walk" />
    <param name="radius" value="1000" />
  </paramset>
  <paramset type="intermodalAccessEgress">
    <param name="mode" value="bike" />
    <param name="maxRadius" value="3000" />
    <param name="linkIdAttribute" value="accessLinkId_bike" />
    <param name="personFilterAttribute" value="hasBike" />
    <param name="personFilterValue" value="true" />
    <param name="stopFilterAttribute" value="bikeAccessible" />
    <param name="stopFilterValue" value="true" />
  </paramset>
</module>
```
#### Differentiating PT Sub-Modes  区分「PT 子方式」
默认情况下，srr-pt路径求解器使用模式pt 创建leg。但当某些服务以特殊（或差异很大）的价格或速度运行时，有必要对公共交通服务进行细分。例如，速度慢但豪华的旅游列车、需要特殊票的高速列车，或者比同类列车服务更便宜但速度更慢的长途汽车。在这些情况下，有必要对不同的服务应用不同的成本，以便找到符合实际需求的路线。同时需要使用不同的成本参数对执行完成的plans进行评分。

SwissRailRaptor 通过将公交线路和路线「存在于schedule文件中」的 transportMode（以下简称“route mode”）映射到“passenger mode”，支持区分 pt 子模式。然后可以在普通 planCalcScore 配置组中配置使用此类乘客模式的成本。

要配置乘客模式映射，需要将以下部分添加到的 config.xml：
```xml
<module name="swissRailRaptor">
  <param name="useModeMappingForPassengers" value="true" />
  
  <paramset type="modeMapping">
    <param name="routeMode" value="train" />
    <param name="passengerMode" value="rail" />
  </paramset>
  <paramset type="modeMapping">
    <param name="routeMode" value="tram" />
    <param name="passengerMode" value="rail" />
  </paramset>
  
  <paramset type="modeMapping">
    <param name="routeMode" value="bus" />
    <param name="passengerMode" value="road" />
  </paramset>
</module>
```

在上面的示例中，假设在 transitSchedule.xml 中，train、tram和bus用作公交运输模式。 在router进行路线搜索过程中，rail和road方式的评分参数用于计算相应线路的使用成本。在构成路线结果的leg中，模式 rail 和 road 也将被使用（可以在结果plan中查看到），而不是 MATSim 的默认 pt 路由器用来描述公共交通leg的默认标识「pt」。 这意味着除了为passenger mode（在上面的rail和road示例中）提供评分参数之外，这些passenger mode还必须在公交配置中列为公交模式，因此它们将被正确识别并作为 pt  leg处理 ：
```xml
 <module name="transit">
   <param name="transitModes" value="rail,road" />
 </module>
```

#### Deterministic Public Transport Simulation
确定性 pt 仿真是一个 QSim 引擎，在 MATSim 中处理公共交通车辆的运动。 默认的 TransitQSimEngine 仿真基于队列的网络上的所有 pt 车辆。 虽然这适用于与私家车交通共享道路基础设施的公共汽车，但在仿真铁路交通时它有一些缺点。 最值得注意的是，火车在现实中并不总是以link（轨道）上允许的最高速度运行「QSim在加载时，agent移动速度主要受载具速度和网络允许的速度共同影响」，在仿真时通常会导致提前到达。
确定性 pt 仿真不模拟队列网络上的 pt 车辆，而是使用其自己的数据结构，并根据时间表中指定的出发和到达时间将车辆从一个站点“传送（teleport）”到另一个站点。 因此，车辆严格按照运输时间表运行（因此得名“确定性”pt 模拟）。
可以通过以下方式配置确定性 pt 模拟：并非所有 pt 车辆都是确定性模拟的，但某些（例如公共汽车）仍然在队列网络上进行模拟，因此能够与私家车交通进行交互。

**使用**
要使用确定性 pt 模拟，需要应用以下几处设定：

- **在schedule文件中为线路增加<transportMode>标记**
```xml
<transitLine id="1">
  <transitRoute id="1">
    <transportMode>train</transportMode>
    <routeProfile>...</routeProfile>
    ...
  </transitRoute>
</transitLine>
```
在公交路线中指定的 transportMode 用于确定服务该路线的车辆是应该使用确定性 pt 模拟还是在QSim队列网络上进行模拟。您可以指定应该确定地模拟哪些方式（例如火车和地铁），以及哪些应该在网络上模拟（例如公共汽车）。
不要在运输路线中使用 pt 作为 transportMode。 这会干扰乘客用来指定他们想要使用公共交通服务的模式 pt。 如果应该对模式为 pt 的公交路线进行确定性模拟，则确定性模拟将引发异常。

- **config.xml**

您需要在 config.xml 中添加一个额外的配置模块：
```xml
<module name="SBBPt" >
  <param name="deterministicServiceModes" value="train,metro" />
  <param name="createLinkEventsInterval" value="10" />
</module>
```
第一个参数deterministicServiceModes 列出了确定性模拟对应的所有的运输方式。多个方式在参数值中用逗号分隔。此列表中未指定的所有运输路线的运输模式将照常在队列网络上模拟。

第二个参数 createLinkEventsInterval 指定应该在哪个迭代中为确定性 pt 引擎模拟的车辆生成 LinkEnter 和 LinkLeave 事件。由于 pt 车辆通过确定性 pt 模拟在停靠点之间传送，因此默认情况下它们不会创建任何LINK事件。但出于可视化或分析目的，让车辆实际沿着路段行驶的事件可能仍然有用。将该参数设置为 0 以禁用链接事件的创建。如果参数设置为 >0，确定性 pt 模拟将在每第 n 次迭代中创建适当的链接事件，类似于控制器的 writeEventsInterval。

### 1.3  EBPTR
MATSim 开发的 EBPTR可以更真实地模拟公共交通路线选择，随着时间的推移，agent可以学习哪些线路是不是经常不准时，以及是否拥挤，从而允许专项更准时和舒适的线路。

**存储旅行时间、等待时间和车辆占用率的结构** 
如前所述，MATSim 的 mobsim 生成称为事件的原子信息单元，它描述了每个agent的关键动作，例如上车或下车，进入和离开一个路段。在仿真过程中，EBPTR目标是在一次模拟中保存有关公共交通服务状态的信息，并在下一次迭代中为agent找到更好的公共交通路线。 这种反馈机制已经在 MATSim 中用于私人交通； 汽车路由器使用前一次迭代中保存的每个link的行程时间，通过更改对应的成本来寻找网络中更好的路线。 为了让 EBPTR 从之前的迭代中学习，关于 (a) 站到站行程时间、(b) 站点的线路等待时间和 (c) 线路中站点之间的车辆占用率信息，是必需的。

**停靠站的旅行时间。**为了考虑公共交通车辆的延误，必须保存连续站点之间的旅行时间。如果两站至少在一条公共交通线路上是连续的，那么它们就是连续的。第一种选择是使用之前讨论过的旅行时间结构，该结构保存了每个道路网络链接的随时间变化的旅行时间。因为车辆在两个连续的站点之间必须遵循已知的道路链接，这些旅行时间可以被加起来。一个问题是：这种结构考虑了网络中的所有车辆，但汽车和公共汽车的旅行时间是非常不同的，特别是在有公共交通站点的环节。因此，我们采用了一种特殊的结构来保存这些站点的旅行时间。更具体地说，每个值包括从车辆到达某一站到下一站的时间，在模拟中用连续的VehicleArrivesAtFacility事件表示。这意味着第一站的等待时间和所有的排队时间（如果车辆在车位或站台允许服务之前必须排队）都包括在内。换句话说，当agent对每个行程的第一个车内环节进行寻路时，全部停留时间将被包括在内。因此，这个agent假定它是第一个进入车辆的乘客。对于所有其他车内环节，车内等待时间也包括在内。这些停留时间是车内环节费用的主要组成部分。

**停靠路线的等待时间。**等待时间是公共交通路线选择的一个基本方面，由于车辆延误（即由于站点位置），或一个或几个连续服务的公共交通车辆满员，等待时间可能会很长。出于这个原因，等待时间被保存为每个站台-路线关系。同样，该结构对一天中某一时间段内某一路线的某一站点的所有代理等待时间进行了平均。更具体地说，每个值包括从代理人到达公共交通站点到进入车辆的时间，在模拟中由连续的AgentArrivesToFacility和PersonEnterVehicle事件表示。这些等待时间是登车环节费用的主要组成部分。如果没有发现某个站台-路线-时间的观察结果，模型就会返回相应的按发车间隔1/2计算的平均等车时间，这是由交通时间表规定的。

**路线-站点占用率。**通过考虑占用率水平，人们可以建立路线选择模型，即人们走较长/较慢的路线，在较空的车辆中感到更舒适，也就是说，重视坐着旅行的较高机会。占用率取决于具体的路线需求和路线中的站点位置。这里，占用率被假定为在两个连续的站点之间是恒定的。当一辆车从某一站出发时（在模拟中表示为VehicleDepartsFromFacility事件），这个结构将占用率与同一时间段内同一路线上从同一站点出发的其他车辆平均起来。由于每个时间段只有少数车辆被记录下来，所以不太可能对某个特定的时间段进行观察。在这种情况下，该结构返回下一个时间段的值，在这个时间段内，至少有一个观察值是针对相应的站点和路线的。

#### 1.3.1 关于EBPTR效果的分析
EBPTR有效地减少了PT达到平衡所需的迭代次数。使用新加坡scenario的25%的样本，下图显示了355 207个代理在100次迭代中的平均得分计划演变。这100次迭代被执行了四次，以使用两个路由器的两种不同的重新规划策略。代理人在内存中保存了计划。在迭代0时，EBPTR和SBPTR都以计划中描述的路线开始；然而，EBPTR返回的路线在这第一次模拟中表现更好。这是因为，对于每一对连续的站点，EBPTR使用包含这一对的所有计划路线时间的平均值作为第一次估计。另一方面，SBPTR使用的是相应路线的特定计划时间。结果表明，平均停靠时间似乎是这个第一次迭代的更可靠的估计。

为了进行更现实的比较，只有20%的agent活动被重新规划，同时以半小时为限制内对10%的agent活动开始时间进行了随机修改。下图显示了两种pt router是如何提高agent的计划得分的。对于EBPTR来说，只需要迭代5次就可以达到即SBPTR的100次迭代才能达到的平均分数。 

![image.png](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1663558658737-72941e72-4102-4076-9c96-0a62d1b18714.png)
EBPTR由于采用了更好的网络拓扑结构，尽管需要存储、获取和更新更多的信息，其在mobsim环节耗时并没有拖累整体的改进。在新加坡scenario中，EBPTR每一个iter消耗的时间可以比SBPTR节省10%-20%。但考虑到SCORE改进的效率（即模型收敛速度），实际效果提升明显。
![image.png](https://raw.githubusercontent.com/RGB3Q/imgbed/master/1663559800989-2fb9f1c4-79ce-49e2-ae31-b5d212ede8fe.png)

