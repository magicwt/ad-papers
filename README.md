![图1 深度学习在计算广告中的应用](https://cdn.nlark.com/yuque/0/2024/svg/1332969/1716085157319-4b056db3-bd9c-499e-9e2b-faa38e79cd40.svg#clientId=udf5f00a9-0f83-4&from=ui&id=u331a7977&originHeight=8354&originWidth=8876&originalType=binary&ratio=2&rotation=0&showTitle=true&size=63419&status=done&style=none&taskId=u61a28ef5-7e42-4da2-ae12-34987b433ec&title=%E5%9B%BE1%20%E6%B7%B1%E5%BA%A6%E5%AD%A6%E4%B9%A0%E5%9C%A8%E8%AE%A1%E7%AE%97%E5%B9%BF%E5%91%8A%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8 "图1 深度学习在计算广告中的应用")
随着机器学习、特别是深度学习的不断发展，其广泛应用于计算广告投放流程的各个阶段。笔者最近对深度学习在计算广告中应用的相关论文进行阅读和梳理，本文是对相关阅读笔记的汇总。文末附录了相关阅读笔记的地址，也附录了笔者所读论文的GitHub地址。欢迎交流和讨论，如有不足之处，请指正。
# 概述
首先介绍一下计算广告中效果类广告的出价方式：

- CPM：广告主在展现上出价，广告平台也在展现上计价；
- oCPM：广告主在转化上出价，广告平台在展现上计价；
- CPC：广告主在点击上出价，广告平台也在点击上计价；
- oCPC：广告主在转化上出价，广告平台在点击上计价；
- CPA：广告主在转化上出价，广告平台在转化上计价。

以上出价方式均是在展现上竞价。本文后续主要基于CPC和oCPC这两种出价方式进行介绍，即广告主在点击或转化上出价，广告平台在展现上竞价，并在点击上计价。
广告投放平台一方面承接众多广告主的投放诉求，存储数百万至数亿的广告物料，另一方面承接众多媒体的广告请求，针对每次广告请求，从海量广告物料中实时检索出相匹配的广告进行投放，整体投放流程类似多级漏斗，一般包含以下步骤：

- 召回（Information Retrieval，IR），从海量广告全集中召回和当前请求相匹配的候选广告集，候选广告集的规模一般是数万；召回一般采用多路归并的方式，并发从多个通道中召回相应的物料，再进行归并，从而能够从多个角度召回和当前用户相关的广告；
- 粗排（Approximate Rank，AR），对召回返回的候选广告集使用相对简单的模型预估每个广告的点击率$\text{pCTR}$，并基于广告主在点击上的出价$\text{bid}$计算$\text{eCPM}$：$\text{eCPM}= \text{bid}\cdot\text{pCTR}$，若广告主在转化上设置出价$\text{tCPA}$，则需要再预估转化率$\text{pCVR}$，并基于以下公式计算$\text{eCPM}$：$\text{eCPM}=\text{tCPA}\cdot\text{pCVR}\cdot\text{pCTR}$，最后，再基于$\text{eCPM}$从大到小进行排序，选取排序靠前的数百个广告；粗排和后续精排的业务逻辑比较类似，但处理的物料量级不同，粗排处理的广告相对精排较多，需要在模型准确度和计算复杂度上进行权衡，以保证整体广告投放链路的实时性，因此粗排的实现思路一种是保持上述和精排类似的业务流程，但使用相对简单的模型预估点击率和转化率，另一种是直接对齐精排排序结果，而对于广告量级较小的广告投放平台，则可以略去粗排阶段，直接将召回的广告进行精排；
- 精排（Full Rank，FR），对粗排返回的数百个广告，采用和粗排类似的方式、但选择相对复杂的模型预估点击率、转化率并计算$\text{eCPM}$，再基于$\text{eCPM}$排序后，选取排序靠前的数十个广告，由于已通过粗排的前置过滤减少进入精排的广告量级，因此，精排可以在保证广告投放链路实时性的前提下，使用相对复杂的模型进行准确地预估，业界有较多的论文和工作探索如何提升点击率和转化率预估的准确性；
- 定价（Price），对精排返回的数十个广告按照一定的机制根据出价计算最后的定价，然后将胜出的广告返回给媒体侧进行展示，当用户点击广告时，广告平台获取相应的点击事件，并按照之前计算的定价计价，扣减广告主的账户余额。精排和定价构成了广告主、投放平台和媒体针对流量的拍卖场景，其中，媒体是供给方，广告主是需求方，而投放平台是撮合方。广告流量的排卖机制包括GFP、GSP和VCG，其中，GSP在业界应用较为广泛，其对当前竞胜广告主的计价采用下一位的出价进行计算，即：$\text{price}=\text{eCPM}_{下一位} / \text{pCTR}=\text{bid}_{下一位}\cdot\text{pCTR}_{下一位}/\text{pCTR}$。而在拍卖机制中，若仅按$\text{eCPM}$排序选取靠前的广告，则流量拍卖的目标是单位流量广告收入（CPM）的最大化，即实现投放平台广告收入最大化这一单目标，而随着计算广告的不断发展，如何实现广告主、投放平台和媒体的三方共赢和可持续发展，这一问题即要求拍卖机制能够实现多目标优化，同时考虑三方的目标，包括广告主由广告带来的成交金额提升（GMV）、投放平台的广告收入提升（CPM）和媒体的用户体验不受影响（CTR）等，业界也有相应的论文和工作探索如何实现拍卖机制的多目标优化。

除了上述介绍的多级漏斗外，广告投放流程一般还会包含以下步骤：

- 过滤：根据一定的规则对广告进行过滤，规则包括广告有效性过滤（如是否超预算等）、广告主设置过滤规则（如否定关键词过滤等）、运营设置过滤规则（如某类流量下屏蔽某行业广告等）等；
- 调控：根据广告投放实际值和期望值的差异进行相关调整，例如，广告投放平台一般都会提供成本保障功能，当广告主实际成本偏离期望成本时，投放平台通常采用控制理论的方法对计价进行反方向调整；
- 平滑：对于有匀速投放诉求的广告主，一般采用控制参竞率的方式，为其实现流量挑选，进而实现平滑投放，避免在每天开始的一段时间，便大量投放广告、消耗预算，导致后续预算不足；
- 改写：根据一定的业务规则或算法优选对广告进行改写，业务规则包括通配符替换等，算法优选包括根据特定的优化目标（如点击率）对广告内容进行局部改写以提升效果；
- 重排：根据一定的业务规则或算法优选对精排后的数十个广告从整体角度进行重排，业务规则包括同类广告打散等，算法优选包括从整体角度对多个广告的排列组合进行优选等，另外，信息流广告还可能进一步和原生内容进行混排，综合商业变现指标和用户体验指标，以确定最终包含商业和非商业内容的整体排序结果；
- 渲染：根据预定义的样式模版对广告的展现样式进行渲染。

投放流程中竞胜的广告最终展示至用户，用户在浏览广告后，可能会点击广告，并进而发生后链路的转化，例如购买商品、下载应用、提交表单等，媒体侧会将广告的展现和点击回传给广告投放平台，同时，广告主侧也会将广告的转化回传给广告投放平台，从而在广告投放平台生成展现、点击和转化日志，广告投放平台根据一定的规则（例如广告链路ID或设备ID）将展现、点击和转化日志拼接成展现样本，每个样本表示一个广告的展现以及是否发生点击和转化，后续召回、粗排和精排等环节所使用的模型便使用上述展现样本进行训练。
广告主投放过程中会经常创编新的广告，这就要求模型具备一定的时效性，做到天级、乃至小时级更新，及时感知新的广告，而展现样本的生成链路中，转化的回收一般存在延迟，例如，游戏付费这一类型转化可能在广告展现数天后才会发生，因此，如何解决模型更新时效性和转化反馈延迟性这一矛盾，业界也有相应的论文和工作探索如何实现反馈延迟建模。
除了上述在线投放流程，广告投放平台还会面向广告主提供各种工具以提升广告创编效率和投放效果，其中，在出价方面，除了广告主人工设置出价，广告投放平台还会提供自动出价功能，其在满足预算约束及其他约束的同时，根据竞价环境自动动态为广告主调整出价，最大化广告主竞得的流量价值之和，业界也有相应的论文和工作探索如何实现自动出价。
从广告主的角度，其通常会在多个广告投放平台、多个媒体流量上进行广告投放，从而能够收集到各个渠道的展现、点击日志，并最终生成用户访问路径样本，每个样本表示某个用户在多个渠道被该广告主的广告触达（每个触达被称为一个触点）以及各触点是否发生点击、最终用户是否发生转化，基于用户访问路径样本，广告主可以进行多触点归因，分析转化在各个触点的归因权重，从而能够量化分析各渠道对转化的影响，以合理分配各渠道的投放预算，业界也有相应的论文和工作探索如何实现多触点归因。
以下就上述介绍的广告投放流程各环节和其中的问题，以及如何将深度学习应用至其中进行稍详细的介绍，更详细的介绍可以浏览笔者在文末所列的相关论文阅读笔记或论文原文。
# 召回
效果类在线广告根据流量类型一般分为搜索广告和推荐广告。在搜索广告中，用户输入搜索词主动表达搜索意图，因此广告主会圈选一批和推广目标相关的关键词，设置出价和匹配方式，传统搜索广告引擎在召回阶段对用户搜索词进行纠错、改写、分词，然后对分词结果，根据精确匹配、短语匹配和宽泛匹配等多种匹配规则，召回相应的关键词，再根据关键词从倒排索引中召回圈选该关键词的广告主的广告，作为广告候选集输入后续流程，最终返回和搜索词相关的广告至用户。在推荐广告中，用户被动浏览系统推荐的内容，因此广告主会圈选一批和推广目标相关的定向条件（如设置目标用户的年龄、性别、地域、兴趣标签等或直接设置目标用户的ID），传统推荐广告引擎根据用户画像中的ID、年龄、性别、地域、兴趣标签等从倒排索引中召回圈选该定向条件的广告主的广告，作为广告候选集输入后续流程，最终返回和定向条件相关的广告至用户。
目前召回阶段一般采用多路召回方式，并发从多个通道中召回相应的物料，再进行归并，从而能够从多个角度召回和当前用户相关的广告，其中除上述传统召回方式外，还会同时引入深度学习进行向量召回。向量召回的一种常用方式是双塔模型，即在搜索广告或推荐广告中，预先将关键词或广告通过广告塔转化为向量，并构建向量索引，将广告请求中的搜索词或用户画像通过用户塔也转化为向量，从向量索引中检索和搜索词或用户画像向量相关的关键词或广告向量，从而将相关的广告（若返回的是关键词，则进一步召回圈选该关键词的广告主的广告）作为广告候选集之一，和其他路召回的广告候选集合并，输入后续流程。
![图2 双塔模型](https://cdn.nlark.com/yuque/0/2024/png/1332969/1716018712968-48580b92-c36d-47e4-a7cb-c3c02f3e8335.png#averageHue=%23eaeaea&clientId=udf5f00a9-0f83-4&from=paste&height=403&id=qbrtr&originHeight=535&originWidth=473&originalType=binary&ratio=2&rotation=0&showTitle=true&size=154092&status=done&style=none&taskId=ud5c27367-2bc6-4c11-bd87-411154a73d6&title=%E5%9B%BE2%20%E5%8F%8C%E5%A1%94%E6%A8%A1%E5%9E%8B&width=356 "图2 双塔模型")
## 双塔模型
### DSSM-双塔模型的起源
微软于2013年发表的论文《Learning Deep Structured Semantic Models for Web Search using Clickthrough Data 》提出了DSSM模型，首次将双塔模型的思想应用在搜索场景下的搜索词和关键词匹配中，后续该思想逐渐被广泛应用于搜索、推荐、广告等场景中。
![图3 DSSM结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1683294071791-46b414b7-e608-4673-b1ea-5152ef6fe7d1.png#averageHue=%23f3f3f3&clientId=u83470369-d5f6-4&from=paste&height=240&id=BJ71i&originHeight=828&originWidth=1938&originalType=binary&ratio=1&rotation=0&showTitle=true&size=231018&status=done&style=none&taskId=u43251c9d-dbc2-41dd-8152-dcb9488d1f9&title=%E5%9B%BE3%20DSSM%E7%BB%93%E6%9E%84&width=561 "图3 DSSM结构")
DSSM的目标是给定搜索词$Q$时，从多个文档$(D_1,D_2,\dots ,D_n)$中召回语义相关的文档。DSSM使用文档的点击日志进行训练（某个文档在某个搜索词下被点击，则认为两者语义相关），并使用深度神经网络挖掘搜索词和文档的相关性。DSSM的结构如图3所示。
而在推理阶段，由于各文档事前可知，可以离线通过模型提前计算出各文档的语义特征向量，并构建向量索引（比如采用Facebook开源的Faiss）。而当用户发起在线搜索请求时，只需对搜索词再次通过模型计算其语义特征向量，然后从向量索引召回余弦相似度相近的若干个文档从而实现文档的语义相关快速召回。
### Mobius-多目标优化的双塔模型
2019年百度搜索广告团队发表的论文《MOBIUS: Towards the Next Generation of Query-Ad Matching in Baidu’s Sponsored Search》规划了新一代的搜索词和广告召回系统——Mobius，并实现了它的第一个版本Mobius V1。论文指出，在搜索广告中，用户输入搜索词，广告系统分两步处理广告请求，如图4左侧所示，第一步是召回（Match），使用相对简单的模型，对于用户输入的搜索词，从数十亿广告集合中查找出数千和搜索词相关的广告，第二步是排序（Rank），进一步划分为粗排和精排，排序对上一步返回的数千广告，使用相对复杂的模型，预测用户的行为，比如点击率（CTR），并计算商业化指标，例如CPM（CPM=CTR×Bid）和ROI，并基于商业化指标对广告进行排序，选取指标最高的若干个广告返回给用户进行展示。
![图5 传统搜索广告系统处理流程和Mobius的对比](https://cdn.nlark.com/yuque/0/2023/png/1332969/1702811466848-4c5ff2ef-e281-427e-a6fb-aeac3d714d0b.png#averageHue=%23f9efdd&clientId=u00e5bf47-d176-4&from=paste&height=258&id=ua8b565b8&originHeight=530&originWidth=1456&originalType=binary&ratio=2&rotation=0&showTitle=true&size=419312&status=done&style=none&taskId=uf3b6c713-d606-43bc-b241-df6fd449747&title=%E5%9B%BE5%20%E4%BC%A0%E7%BB%9F%E6%90%9C%E7%B4%A2%E5%B9%BF%E5%91%8A%E7%B3%BB%E7%BB%9F%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B%E5%92%8CMobius%E7%9A%84%E5%AF%B9%E6%AF%94&width=708 "图5 传统搜索广告系统处理流程和Mobius的对比")
论文指出，将广告系统划分为相互独立的两步，存在两者目标不一致的问题，召回的目标是相关性最高，而排序的目标是商业化指标最大化，因此可能召回时选出的高相关性广告在排序时商业化指标很低。因此，论文提出了Mobius-V1算法框架，在召回时，引入多目标，在保证低延时的前提下，同时考虑搜索词和广告的相关性，以及商业化指标，因此，论文设计了“Teacher-Student”框架对训练数据进行调整，使得排序时的点击率预测模型能够应用于召回时。首先使用一个数据生成器生成搜索词和广告配对，然后使用原有的召回模块作为teacher，判断搜索词和广告的相关性，从而发现bad case，例如高点击率、低相关性的配对。原先排序时的点击率预测模型，作为student，对bad case进行学习，从而能够具备识别相关性的能力。
![图7 “Teacher-Student”算法框架](https://cdn.nlark.com/yuque/0/2023/png/1332969/1702720809180-6878650c-8b36-4de6-9a46-186954878fa7.png#averageHue=%23f6f5f4&clientId=u736a971d-b4b8-4&from=paste&height=437&id=Vr7mx&originHeight=990&originWidth=1528&originalType=binary&ratio=1&rotation=0&showTitle=true&size=813618&status=done&style=none&taskId=u2e606508-f815-4d4d-b062-aeafdbc04dc&title=%E5%9B%BE7%20%E2%80%9CTeacher-Student%E2%80%9D%E7%AE%97%E6%B3%95%E6%A1%86%E6%9E%B6&width=674 "图7 “Teacher-Student”算法框架")
在线服务时需要对数十亿广告和搜索词的配对预测点击率，为保证快速实时召回，无法暴力穷举数十亿广告集合中的每个广告，预测其和搜索词配对的点击率，因此论文引入向量检索技术实现广告的快速召回。从点击率预测模型网络结构可以看出，用户（搜索词）向量和广告向量进行内积（即余弦相似度）后，再由Softmax函数输出点击率预测，因此，用户（搜索词）向量和广告向量的内积（即余弦相似度）和点击率单调相关，可以将召回时查找当前用户（搜索词）下点击率最高的若干个广告，转化为查找当前用户（搜索词）下内积（即余弦相似度）最高的若干个广告。
![图8 快速的广告召回](https://cdn.nlark.com/yuque/0/2023/png/1332969/1702873927004-b90ef836-8a7b-446f-83d4-61159054a047.png#averageHue=%23ebeae9&clientId=u00e5bf47-d176-4&from=paste&height=290&id=ykE0y&originHeight=580&originWidth=712&originalType=binary&ratio=2&rotation=0&showTitle=true&size=240429&status=done&style=none&taskId=ua918c719-f696-4062-a474-0166681f0fe&title=%E5%9B%BE8%20%E5%BF%AB%E9%80%9F%E7%9A%84%E5%B9%BF%E5%91%8A%E5%8F%AC%E5%9B%9E&width=356 "图8 快速的广告召回")
查找的具体实现方案有两种，如图8所示，其中一种方案基于近似最近邻（Approximate Nearest Neighbor，ANN）算法，ANN算法在损失一定查全率的代价下，能够保证大数据量下的快速响应，而常见的ANN算法有基于树的算法（例如ANNOY）、基于图的算法（例如HNSW）、基于哈希的算法（例如LSH）等，论文中采用了基于树的算法——ANNOY。ANNOY将所有向量构造成一个二叉树，其中每个叶子节点对应到一个向量，整个二叉树的构造过程是从根节点逐层向下，在根节点，将所有向量基于聚类算法分成两类，作为根节点的左右子节点，然后对左右子节点中的向量，再分别基于聚类算法分成两类，如此递归划分，直至最后的叶子节点中只有一个向量。通过ANNOY查找向量的时间复杂度是 𝑂(log⁡𝑁) 。
### SDM-引入用户行为序列的双塔模型
2019年阿里巴巴推荐团队发表的论文《SDM: Sequential Deep Matching Model for Online Large-scale Recommender System》提出了SDM（Sequential Deep Matching Model）算法，其在双塔模型的基础上，引入用户行为序列，挖掘其中长、短期兴趣信息，并融合得到用户兴趣表征，从而能够通过用户历史行为进行更加个性化的推荐。
![图9 SDM问题建模和处理流程](https://cdn.nlark.com/yuque/0/2023/png/1332969/1702468570059-75b7ccaf-7ac5-4e4b-9ad6-fcb3a9c30f47.png#averageHue=%23f2f0eb&clientId=u1232d9c5-2647-4&from=paste&height=279&id=EGO3C&originHeight=558&originWidth=756&originalType=binary&ratio=2&rotation=0&showTitle=true&size=309404&status=done&style=none&taskId=u817dabb8-6569-4070-b131-7c79c2de707&title=%E5%9B%BE9%20SDM%E9%97%AE%E9%A2%98%E5%BB%BA%E6%A8%A1%E5%92%8C%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B&width=378 "图9 SDM问题建模和处理流程")
令$\mathcal{U}$表示用户集合，$\mathcal{I}$表示商品集合，模型需要判断在时间$t+1$，用户$u\in\mathcal{U}$是否对商品$i\in\mathcal{I}$感兴趣。令$\mathcal{S}^u=[i_1^u,\dots,i_t^u,\dots,i_m^u]$表示用户$u$的短期行为序列，其中包含了用户最近有交互的$m$个商品。令$\mathcal{L}^u$表示用户$u$的长期行为序列，其中包含了用户最近7天内除$\mathcal{S}^u$外有交互的多个商品。令$s_t^u$、$p^u$分别表示用户$u$在时间$t$的短期行为序列的表征和长期行为序列的表征，这两种表征通过图9中的门控神经网络得到用户行为序列的表征$o_t^u\in\mathbb{R}^{d\times1}$。令$V\in\mathbb{R}^{d\times|\mathcal{I}|}$表示商品集合$\mathcal{I}$中所有商品的Embedding向量。将用户$u$的行为序列表征$o_t^u$和商品$i$的Embedding向量$v_i$的内积作为两者相关性得分：
$z_i=\text{score}(o_t^u,v_i)={o_t^u}^Tv_i$
在时间$t+1$，查询和用户相关性得分靠前的$N$个商品作为召回的商品集合，输入推荐的后续环节。
训练阶段的正样本即在时间$t+1$和用户有交互的商品$i_{t+1}^u$，负样本是从商品集合$\mathcal{I}$中排除商品$i_{t+1}^u$后进行采样的商品。令用户$u$的正负样本集合为$\mathcal{K}$，用户和每个样本的相关性得分集合为$z=[z_1,\dots,z_{|\mathcal{K}|}]$，将相关性得分集合通过Softmax函数得到用户和每个样本在时间$t+1$进行交互的预估概率集合$\hat{y}=[\hat{y}_1,\dots,\hat{y}_{|\mathcal{K}|}]$：
$\hat{y}=\text{softmax}(z)$
损失函数使用交叉熵损失函数：
$L(\hat{y})=-\sum_{i\in\mathcal{K}}{y_i\log(\hat{y}_i)}$
推理时，预先将商品集合$\mathcal{I}$中所有商品的Embedding向量保存至向量检索引擎——FAISS中，另外，系统实时收集用户行为日志，并转化为结构化的数据，在时间$t+1$，用户短期行为序列$\mathcal{S}_u$和长期行为序列$\mathcal{L}_u$被输入SDM，输出用户行为序列表征$o_t^u$，再从FASIS中查询和用户行为序列表征最相关的$N$个商品。
### MIND-多用户向量的双塔模型
2019年阿里巴巴天猫团队发表的论文《Multi-Interest Network with Dynamic Routing for Recommendation at Tmall》提出了MIND（Multi-Interest Network with Dynamic routing）算法，设计了多兴趣抽取层（Multi-Interest Extractor Layer），通过动态路由（Dynamic Routing），自适应地聚合用户历史行为生成用户兴趣表征，将用户历史行为划分为多个聚类，每类的用户历史行为被转化为表征用于表示用户的某一类兴趣，因此，对于一个用户，MIND会输出多个用户表征，用于表达用户多样化的兴趣。
![图10 MIND网络结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1702970303898-8a465417-9553-4b7a-bcf8-ba7c4cf99061.png#averageHue=%23f6f5f4&clientId=udc149094-1916-4&from=paste&height=361&id=Lfd6g&originHeight=722&originWidth=1516&originalType=binary&ratio=2&rotation=0&showTitle=true&size=578006&status=done&style=none&taskId=u0de0c5fa-f52f-40f5-8a13-ea7da7b94a0&title=%E5%9B%BE10%20MIND%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=758 "图10 MIND网络结构")
令商品集合为$\mathcal{I}$，用户集合为$\mathcal{U}$，召回的目标就是对于用户$u\in\mathcal{U}$，从数十亿的商品集合中召回数千个用户感兴趣的商品，且召回的每个目标商品可由三元组$(\mathcal{I}_u,\mathcal{P}_u,\mathcal{F}_i)$表示，其中，$\mathcal{I}_u$表示用户已交互的商品集合（即用户行为序列），$\mathcal{P}_u$表示用户画像（年龄、性别等），$\mathcal{F}_i$表示召回的目标商品的特征（ID、类别等），MIND的任务之一是将用户行为序列和画像通过用户塔转化为用户多兴趣表征：
$V_u=f_{user}(\mathcal{I}_u,\mathcal{P}_u)$
其中，$V_u=(\overrightarrow{v}_u^1,\dots,\overrightarrow{v}_u^K)\in\mathbb{R}^{d\times K}$表示用户多兴趣表征，共有$K$列不同的兴趣，每列兴趣的维度为$d$。
MIND的任务之二是将召回的目标商品的特征通过商品塔转化为商品表征：
$\overrightarrow{e}_i=f_{item}(\mathcal{F}_i)$
其中，$\overrightarrow{e}_i\in\mathbb{R}^{d\times 1}$表示商品表征。
最后，计算用户每个兴趣额表征和商品表征的余弦相似度，取其中最大值作为用户和商品的相关性得分：
$f_{score}(V_u,\overrightarrow{e}_i)=\max_{1\le k\le K}\overrightarrow{e}_i^T\overrightarrow{v}_u^k$
然后取用户和商品相关性得分最高的$N$个商品作为召回的商品。
## 树索引
通过双塔模型，可以分别实现对用户和广告信息的深度挖掘和语义理解，并通过向量索引，实现语义层面和用户相关广告的快速召回，从而突破原有基于文本匹配或定向条件匹配的召回通道的性能天花板。
但双塔模型也存在以下不足：为了能够单独计算用户和广告向量、实现广告向量的预先计算和索引构建，模型中的用户塔和广告塔分别对用户和广告信息进行挖掘，而无法对两者信息进行交叉挖掘，另外，模型最后只能通过用户和广告向量的内积表达两者的相关性，以上在模型结构上的约束限制了模型的表达能力。因此，为突破双塔模型效果天花板，在保证低延时从海量数据实现召回的前提下，提升模型表达能力，近几年多篇论文在索引和模型的充分结合和联合优化上开展了大量工作，引入树、图等索引结构，通过模型实时计算用户和广告（商品）的相关性，从而突破以向量内积表达相关性的效果天花板，同时，通过模型和索引的充分结合和联合优化，使得模型计算的相关性能够满足索引结构的相关约束，从而能够通过索引实现高效剪枝和快速检索，避免因暴力穷举计算用户和所有广告（商品）的相关性而无法满足在线召回的低延时要求。
### TDM-深度学习和树索引的结合
《Learning Tree-based Deep Model for Recommender Systems》是阿里妈妈算法团队于2018年发表的一篇论文，其中提出了TDM，创新性地将树结构索引和深度神经网络结合，在推荐系统召回阶段，通过树结构索引实现海量商品的快速检索和高效剪枝。
![图11 TDM结构](https://cdn.nlark.com/yuque/0/2022/png/1332969/1667697281221-7110b04a-8e23-488a-a443-58410f81775f.png?x-oss-process=image%2Fresize%2Cw_937%2Climit_0#averageHue=%23f3f2f1&from=url&height=406&id=FeZow&originHeight=522&originWidth=937&originalType=binary&ratio=1&rotation=0&showTitle=true&status=done&style=none&title=%E5%9B%BE11%20TDM%E7%BB%93%E6%9E%84&width=728 "图11 TDM结构")
TDM的结构如图11所示，其包含树结构和深度神经网络两部分，对于树中每个节点$n$，可以为其训练一个分类器$P^{(j)}(n|u)$（其中$u$表示用户，$j$表示节点$n$所在的树层级），用于计算用户$u$对于节点$n$（可能是某个商品分类，也可能是某个商品）的感兴趣概率。在此基础上，对于召回问题，其实就是要计算用户$u$最感兴趣的$k$个商品，可以从上到下逐层遍历树，在每一层查找该层用户$u$最感兴趣的$k$个节点，剪枝该层其他节点，然后再从这$k$个节点下一层的子节点中查找用户$u$最感兴趣的$k$个节点，如此循环直至最下一层，查找到用户$u$最感兴趣的$k$个叶子节点（商品），实现召回。
因为树深是$\text{O}(\ln{|C|})$（$|C|$为商品总数），且在每层查找时均可以剪枝，所以上述查找过程是非常高效的，至多遍历$2*k*\log{|C|}$个节点，但需要树结构中的非叶子节点分类器满足以下条件：
$P^{(j)}(n|u)=\frac{\max_{n_c\in\{n\text{'s children nodes in level } j+1\}}P^{(j+1)(n_c|u)}}{\alpha^{(j)}}
\tag{2}$
即每个节点的概率等于其子节点的概率的最大值除以归一化项$\alpha^{(j)}$，这样能保证在任何一层查找概率较大的$k$个节点后，其下一层概率较大的$k$个节点均属于上一层这$k$个节点的子节点。而归一化项的引入，是用于调整$P^{(j)}(n|u)$的大小，保证同一层中各节点的概率和为1。论文中将满足这一特性的树结构称为最大堆树（Max-Heap Like Tree）。
论文中还提到，召回过程中实际并不需要计算出概率真实值，只需要计算出各节点概率的相对大小即可，因此论文使用用户和商品交互这类隐式反馈作为样本，使用一个深度神经网络进行训练，用作各节点的分类器，即全局所有分类器共用一个深度神经网络。
对于树的构建和模型训练，论文的整体方案是，先采用一定的方法初始化树，再按以下的步骤循环多次：

- 基于已构建树和叶子节点样本数据，生成其他每层样本数据，然后输入深度神经网络进行模型训练；
- 模型训练完成后，基于训练后所得叶子节点的Embedding向量重新构建新树；

最终得到线上服务使用的深度神经网络和树结构。
### JTM-深度学习和树索引的联合优化
《Joint Optimization of Tree-based Index and Deep Model for Recommender Systems》是阿里妈妈算法团队于2019年发表的一篇论文。在原TDM的论文中，训练深度神经网络$\mathcal{M}$和构造树索引$\mathcal{T}$相互独立，分别采用了不同的优化目标，模型训练的优化目标是最小化对数似然损失函数，而树索引构建的优化目标是K均值聚类算法中的根据最近距离选择样本类别、再根据样本均值选择聚类中心、如此循环直至样本类别不再变化，模型训练和树索引构建的优化目标不一致，导致整体TDM算法不易达成最优解。因此，在2019年的这篇论文中，阿里妈妈算法团队提出了一个新的联合优化框架——JTM（Joint Optimization of Tree-based Index and Deep Model），使用一个全局损失函数$\mathcal{L}(\theta,\pi)$（其中，$\theta$是用户偏好模型$\mathcal{M}$的参数，$\pi$是将商品映射至树索引$\mathcal{T}$的叶子节点的函数），同时训练$\mathcal{M}$和构造$\mathcal{T}$，从而进一步提升召回的准确性。
![图12 联合优化框架](https://cdn.nlark.com/yuque/0/2023/png/1332969/1702016980795-23260994-c409-4351-9746-08443b16a15e.png#averageHue=%23f3f1f0&clientId=u66df272c-cde7-4&from=paste&height=207&id=ua6740541&originHeight=414&originWidth=782&originalType=binary&ratio=2&rotation=0&showTitle=true&size=214827&status=done&style=none&taskId=ud3af1660-f1ab-4b41-a6af-1a2c801fc74&title=%E5%9B%BE12%20%E8%81%94%E5%90%88%E4%BC%98%E5%8C%96%E6%A1%86%E6%9E%B6&width=391 "图12 联合优化框架")
![图13 商品和叶子节点的映射函数的优化算法](https://cdn.nlark.com/yuque/0/2023/png/1332969/1702023073235-beb45ae5-4e58-4479-a4de-27eba4ddc2f9.png#averageHue=%23f7f6f5&clientId=u66df272c-cde7-4&from=paste&height=386&id=WrRi2&originHeight=772&originWidth=828&originalType=binary&ratio=2&rotation=0&showTitle=true&size=398503&status=done&style=none&taskId=uc19bc99b-9f7d-4be4-9f96-52fc0882117&title=%E5%9B%BE13%20%E5%95%86%E5%93%81%E5%92%8C%E5%8F%B6%E5%AD%90%E8%8A%82%E7%82%B9%E7%9A%84%E6%98%A0%E5%B0%84%E5%87%BD%E6%95%B0%E7%9A%84%E4%BC%98%E5%8C%96%E7%AE%97%E6%B3%95&width=414 "图13 商品和叶子节点的映射函数的优化算法")
基于上述全局损失函数，联合优化框架如图12所示，在每次优化迭代中，先优化模型，按梯度下降调节模型参数，最小化全局损失函数$\mathcal{L}(\theta,\pi)$，再按照图13所示的树索引优化算法优化商品和叶子节点的映射函数$\pi(\cdot)$、并重建树索引，最小化全局损失函数$\mathcal{L}(\theta,\pi)$。每次迭代包含独立的两步、分别优化模型和索引的原因是映射函数$\pi(\cdot)$的优化属于组合优化（Combinational Optimization）问题，较难通过梯度下降同时优化模型参数和映射函数。
## 图索引
### NANN-深度学习和图索引的结合
《Approximate Nearest Neighbor Search under Neural Similarity Metric for Large-Scale Recommendation》是阿里妈妈算法团队于2022年发表的一篇论文。这篇论文提出了一个新的召回方案，即在通过近似最近邻搜索算法快速查找和用户相近的若干个商品时，使用深度神经网络模型的计算输出作为用户和商品的距离度量表示其相关性，替代内积、余弦相似度等度量形式的用户和商品的向量距离。这样，既可以充分使用模型的表达能力保证用户和商品相关性的准确性，也可以通过近似最近邻搜索算法（论文中使用HNSW算法）保证结果的快速返回。论文将该方案称为NANN（Neural Approximate Nearest Neighbor Search）。
TDM方案存在以下不足：一是索引和模型的联合训练比较耗计算资源，二是树结构索引中的每个非叶子节点并不表示具体的某个商品（仅每个叶子节点表示具体的某个商品），因此在模型中，节点特征无法使用商品信息。而NANN解决了上述的两个不足：一是在模型训练和图搜索上进行优化减少计算量，二是图中的节点均表示具体商品，可以充分使用商品信息。
NANN的模型结构如下图所示：
![图14 NANN模型结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1674480494267-0d2e4170-383b-45ac-a785-e9dbddaed9ec.png#averageHue=%23fbfafa&clientId=ubbb84eb2-64dc-4&from=paste&height=286&id=GcWUO&originHeight=782&originWidth=1928&originalType=binary&ratio=1&rotation=0&showTitle=true&size=266220&status=done&style=none&taskId=u449dec09-e849-4abb-94c7-90a35ec139e&title=%E5%9B%BE14%20NANN%E6%A8%A1%E5%9E%8B%E7%BB%93%E6%9E%84&width=704 "图14 NANN模型结构")
遍历图时，将遍历的节点作为模型输入之一，模型的其他输入还包括用户信息、行为序列，输出是给定当前用户时，和该节点的相关性得分。模型结构包括5个部分：

- Embedding层，将原始各特征转化为Embedding向量；
- 商品网络，输入为图搜索当前遍历的节点（商品）各种特征的Embedding向量，输出为商品的Embedding向量；
- 注意力网络，输入为行为序列中各商品各种特征的Embedding向量、以及商品网络输出的商品Embedding向量，通过注意力网络，计算行为序列中各商品和当前商品的相关性权重，然后对行为序列中各商品的Embedding向量进行加权求和作为输出；
- 用户网络，输入为当前用户各种特征的Embedding向量，输出为用户的Embedding向量；
- 评分网络，输入为上述商品网络、注意力网络、用户网络输出的拼接，输出为用户和商品的相关性得分。

对于图的构建，论文直接使用了HNSW算法，并使用商品向量的L2距离作为距离度量。基于HNSW算法进行分层遍历，如下所示：
![图15 Beam-Retrieval算法](https://cdn.nlark.com/yuque/0/2023/png/1332969/1674655654488-8c7441ef-21ce-42d6-8361-8195fc0f02de.png#averageHue=%23d9d8d7&clientId=u51c79fb3-016b-4&from=paste&height=327&id=VYiPY&originHeight=654&originWidth=906&originalType=binary&ratio=1&rotation=0&showTitle=true&size=367493&status=done&style=none&taskId=u009ba816-5458-4d4f-83b6-f394f47ec86&title=%E5%9B%BE15%20Beam-Retrieval%E7%AE%97%E6%B3%95&width=453 "图15 Beam-Retrieval算法")
其中，令构建好的分层图为$G$，图的层数为$L$，用户为$u$，要求返回$K$个和用户最相关（距离最近）的节点，其集合为为$W$。从上到下逐层搜索，在每层，调用SEARCH-LAYER算法，该算法本质是一个贪心算法，具体如下：
![图16 每一层的SEARCH-LAYER算法](https://cdn.nlark.com/yuque/0/2023/png/1332969/1674656982636-4a58972d-2401-4171-a2e0-8f8d764909a2.png#averageHue=%23dededd&clientId=u51c79fb3-016b-4&from=paste&height=325&id=V9WnG&originHeight=650&originWidth=909&originalType=binary&ratio=1&rotation=0&showTitle=true&size=348375&status=done&style=none&taskId=u9a3d4ac1-5eea-4c6a-87b9-2b990cf6edc&title=%E5%9B%BE16%20%E6%AF%8F%E4%B8%80%E5%B1%82%E7%9A%84SEARCH-LAYER%E7%AE%97%E6%B3%95&width=454.5 "图16 每一层的SEARCH-LAYER算法")
令当前层为$l_c$，当前层开始遍历的节点集合为$ep$（即上一层的输出），当前层搜索的步数为$T_c$，当前层已被访问的节点集合为$S$，当前层待搜索的候选节点集合为$C$，当前层和用户最相关（距离最近）的节点集合为$W$。初始时，将$ep$赋值给集合$S$、$C$和$W$。对于搜索的每一步：

- 将候选节点集合$C$的所有邻居节点赋值给集合$N$；
- 从集合$N$中减枝已被访问的节点集合$S$；
- 对集合$N$中减枝后余下的节点和当前层和用户最相关的节点集合$W$取并集，并对这些节点求取$s(u,v)$最大（和用户最相关）的$K$个节点，赋值给集合$W$，即覆盖更新当前层和用户最相关的节点集合；
- 对集合$N$和集合$W$取交集，作为下一步的候选节点集合$C$；
- 对集合$N$和集合$S$取并集，即将本步中访问的节点标记为已被访问，用于下一步的剪枝；
# 精排+定价
精排阶段，对粗排返回的数百个广告，采用和粗排类似的方式、但选择相对复杂的模型预估点击率、转化率并计算$\text{eCPM}$，再基于$\text{eCPM}$排序后，选取排序靠前的数十个广告，由于已通过粗排的前置过滤减少进入精排的广告量级，因此，精排可以在保证广告投放链路实时性的前提下，使用相对复杂的模型进行准确地预估，业界有较多的论文和工作探索如何提升点击率和转化率预估的准确性，下面先分别介绍一下点击率预估和转化率预估的相关工作。
而对于拍卖机制，如何实现广告主、投放平台和媒体的三方共赢和可持续发展，这一问题即要求拍卖机制能够实现多目标优化，同时考虑三方的目标，包括广告主由广告带来的成交金额提升（GMV）、投放平台的广告收入提升（CPM）和媒体的用户体验不受影响（CTR）等，业界也有相应的论文和工作探索如何实现拍卖机制的多目标优化，下面随后介绍一下多目标优化的相关工作。
出价是排序中的一个重要因子，因此，本部分最后再介绍一下自动出价的相关工作。
## 点击率预估
点击率预估（Click Through Rate Prediction）是广告和推荐系统中的核心工作之一。在广告系统中，当用户发起广告请求后，系统需要从全量广告集合中召回相关的若干条候选广告，并对于每个候选广告预估其点击率，最后通过一定的排序公式（如使用点击率乘以出价作为排序分进行排序），筛选排序靠前的候选广告作为最终胜出的广告向用户曝光。
早期，Logistic回归模型由于其解释性好、易于工程实现的优点被广泛应用于点击率预估，而随着深度学习的发展，深度学习逐渐被应用于点击率预估，并在模型结构上不断演进，包括深度学习和浅层学习结合的Wide & Deep，深度学习和特征交叉结合的DeepFM、PNN等，还有一系列工作不只考虑单次广告请求的上下文和候选广告的特征，还会考虑用户历史行为的特征，基于深度学习对用户兴趣进行建模。
### 用户行为序列学习
#### DIN-基于注意力的用户兴趣学习
2018年阿里妈妈发表的论文《Deep Interest Network for Click-Through Rate Prediction》提出了DIN算法。DIN算法将用户历史行为中的商品的Embedding向量作为用户兴趣的表征。由于不同用户历史行为长度不同，涉及的商品数目不同，因此需要将不同数目的商品的Embedding向量通过池化汇总为一个Embedding向量，以满足多层神经网络输入维度固定的要求，而池化会限制用户兴趣的表达，另外，用户兴趣是多样化的，比如某男性用户历史上可能陆续购买过手机和运动鞋，若候选广告是平板电脑，则购买手机这一历史行为所表征的用户兴趣和候选广告更相关，应该有更多的权重，基于这一原则，DIN算法引入注意力机制，计算历史行为中的商品的Embedding向量和候选广告的注意力得分，并基于注意力得分对历史行为中的商品的Embedding向量进行加权求和池化，从而挖掘和候选广告相关的用户兴趣，并满足多层神经网络输入维度固定的要求。
![图17 基线模型和DIN的网络结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1700271261648-8f74cf4e-ee31-46e8-b379-187f5027f8d6.png#averageHue=%23f7f6f6&clientId=udb7e10f4-ff5f-4&from=paste&height=280&id=ub9da3789&originHeight=560&originWidth=1598&originalType=binary&ratio=2&rotation=0&showTitle=true&size=130578&status=done&style=none&taskId=ucec00b0c-90d5-47e1-9215-6e0338d6f00&title=%E5%9B%BE17%20%E5%9F%BA%E7%BA%BF%E6%A8%A1%E5%9E%8B%E5%92%8CDIN%E7%9A%84%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=799 "图17 基线模型和DIN的网络结构")
#### DIEN-基于RNN和注意力的兴趣演进学习
2019年阿里妈妈发表的论文《Deep Interest Evolution Network for Click-Through Rate Prediction》提出了DIEN算法。前述的DIN算法将用户历史行为中的商品的Embedding向量直接作为用户兴趣的表征，并通过注意力机制计算历史行为中的商品的Embedding向量和目标商品的注意力得分，并基于注意力得分对历史行为中的商品的Embedding向量进行加权求和，从而挖掘历史行为中和目标商品相关的用户兴趣的表征。DIEN在此基础上，首先通过兴趣抽取层，使用RNN对用户行为序列进行建模，将RNN输出的隐状态作为兴趣表征，将行为序列转化为兴趣表征序列，再通过兴趣演进层，引入注意力机制计算用户各阶段兴趣表征和候选广告的注意力得分，并结合注意力得分和RNN，对用户和候选广告相关的兴趣演进过程进行建模，得到用户和候选广告相关的兴趣演进表征。
![图18 DIEN网络结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1699322733740-9e4b69fe-e788-40fd-a001-70b9727ff539.png#averageHue=%23f9f8f7&clientId=ub0fa1bc3-b793-4&from=paste&height=366&id=Lf6hH&originHeight=732&originWidth=1548&originalType=binary&ratio=2&rotation=0&showTitle=true&size=158734&status=done&style=none&taskId=u774103de-4de0-46f6-b1fc-cb73625ab46&title=%E5%9B%BE18%20DIEN%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=774 "图18 DIEN网络结构")
#### DSTN-引入注意力的时空信息学习
2019年阿里超级汇川发表的论文《Deep Spatio-Temporal Neural Networks for Click-Through Rate Prediction》提出了DSTN算法。DSTN算法的应用场景是搜索广告。在搜索广告场景下，用户输入搜索词，由广告系统返回若干个相关的广告展示至用户。DSTN算法认为当前候选广告是否被点击会受两类信息影响，如图19所示：

- 时间维度的信息，和DIN、DIEN类似，DSTN认为用户历史点击的广告（Clicked ads）表征了用户正向偏好，可以使用该信息辅助预测候选广告的点击率，同时，DSTN还认为用户历史未点击的广告（Unclicked ads）表征了用户负向偏好，也可以使用该信息辅助预测候选广告的点击率；
- 空间维度的信息，DSTN认为和候选广告在同一页面中的其他广告（Contextual ad）也会影响候选广告是否被点击，例如某男性用户搜索手机，若返回的多个广告中只有一个是其欲购买的品牌，则这个广告较大概率会被点击。

![图19 搜索广告场景下的时空信息](https://cdn.nlark.com/yuque/0/2023/png/1332969/1700025353082-b84b2cd9-c32f-42ff-81ab-d80d59f2428e.png#averageHue=%23ece1b8&clientId=ua8663c00-f40a-4&from=paste&height=235&id=nntGI&originHeight=470&originWidth=780&originalType=binary&ratio=2&rotation=0&showTitle=true&size=373796&status=done&style=none&taskId=u801bc78e-ee4d-420b-b1a2-f99c9decc4d&title=%E5%9B%BE19%20%E6%90%9C%E7%B4%A2%E5%B9%BF%E5%91%8A%E5%9C%BA%E6%99%AF%E4%B8%8B%E7%9A%84%E6%97%B6%E7%A9%BA%E4%BF%A1%E6%81%AF&width=390 "图19 搜索广告场景下的时空信息")
基于上述分析，DSTN将用户历史点击广告序列、历史未点击广告序列、上下文广告序列作为特征，由于点击广告序列、未点击广告序列、上下文广告序列涉及多个广告，而不同广告和候选广告的相关性不同，因此和DIN类似，DSTN使用交互注意力机制（Interactive Attention）分别计算每个点击广告、未点击广告、上下文广告的Embedding向量和候选广告的Embedding向量的注意力得分，基于注意力得分，分别对点击广告序列、未点击广告序列、上下文广告序列中的多个广告的Embeding向量进行加权求和，从而得到点击广告序列、未点击广告序列、上下文广告序列和候选广告相关的表征，最后将点击广告序列、未点击广告序列、上下文广告序列的Embedding向量和候选广告的Embedding向量拼接在一起，输入全连接网络层，并由Sigmoid函数输出候选广告的预测点击率。
![图20 DNN、DSTN-Pooling model、DSTN-Interactive attention model](https://cdn.nlark.com/yuque/0/2023/png/1332969/1700026672627-1ba8b918-f211-48a0-930d-007c70ff8c30.png#averageHue=%23f5f3f2&clientId=ua8663c00-f40a-4&from=paste&height=270&id=ndCbR&originHeight=602&originWidth=1592&originalType=binary&ratio=2&rotation=0&showTitle=true&size=136120&status=done&style=none&taskId=u8774e70d-cdb0-41fd-8652-a7e01d7a6aa&title=%E5%9B%BE20%20DNN%E3%80%81DSTN-Pooling%20model%E3%80%81DSTN-Interactive%20attention%20model&width=715 "图20 DNN、DSTN-Pooling model、DSTN-Interactive attention model")
#### DFN-基于Transformer和注意力的隐式正负反馈、显式负反馈挖掘
2020年腾讯发表的论文《Deep Feedback Network for Recommendation》提出了DFN算法。DFN算法的应用场景是微信中的内容推荐，其将内容推荐问题转化为内容点击率预估问题，而内容推荐场景下，用户对内容有三种反馈，如图21所示：

- 隐式正反馈（Implicit positive feedback），即用户点击内容；
- 隐式负反馈（Implicte negative feedback），即用户未点击内容；
- 显式负反馈（Explicit negative feedback），即用户点击内容的“不喜欢”按钮。

![图21 内容推荐场景下的用户反馈](https://cdn.nlark.com/yuque/0/2023/png/1332969/1700038993938-8ba09e47-78ee-473e-a452-9cd47b86ec6c.png#averageHue=%23e1dace&clientId=ua8663c00-f40a-4&from=paste&height=292&id=A4ABM&originHeight=584&originWidth=948&originalType=binary&ratio=2&rotation=0&showTitle=true&size=174937&status=done&style=none&taskId=u59bbad84-8a1e-4390-b4d2-d92c2739a83&title=%E5%9B%BE21%20%E5%86%85%E5%AE%B9%E6%8E%A8%E8%8D%90%E5%9C%BA%E6%99%AF%E4%B8%8B%E7%9A%84%E7%94%A8%E6%88%B7%E5%8F%8D%E9%A6%88&width=474 "图21 内容推荐场景下的用户反馈")
DFN对用户这三种反馈的行为序列进行建模，从中学习用户的正向和负向偏好。DFN首先使用Transformer中的多头自注意力分别挖掘每种反馈的行为序列和候选内容的相关性，得到每种反馈的行为序列和候选内容相关的Embedding向量。针对用户隐式负反馈多、但噪音也多（用户不点击内容可能有多种原因，并不一定是因为不喜欢），而显式负反馈和隐式正反馈少、但较准确的特点，DFN基于注意力机制，分别计算未点击行为和显式负反馈、隐式正反馈的Embedding向量的注意力得分，并使用上述两种注意力得分分别对未点击行为的Embedding向量进行加权求和，得到未点击行为中负向偏好、正向偏好的Embedding向量，最后将三种反馈的行为序列和候选内容相关的Embedding向量，以及未点击行为中负向偏好、正向偏好的Embedding向量拼接在一起作为用户反馈的表征，连同其他类型的特征，进行特征交互，再由Sigmoid函数输出候选内容的预估点击率。
![图22 DFN的网络结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1700383223944-2b363bcb-6c89-4281-b6d0-58a7571730fd.png#averageHue=%23fcfaf7&clientId=u8a130143-dc5d-4&from=paste&height=830&id=uc61034fd&originHeight=830&originWidth=2294&originalType=binary&ratio=1&rotation=0&showTitle=true&size=259523&status=done&style=none&taskId=uf08e891e-36e7-46e5-a18e-8d07fc89a42&title=%E5%9B%BE22%20DFN%E7%9A%84%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=2294 "图22 DFN的网络结构")
#### RACP-基于RNN和注意力的页内和页间信息学习
2022年阿里巴巴发表的论文《Modeling Users' Contextualized Page-wise Feedback for Click-Through Rate Prediction in E-commerce Search》提出了RACP算法。RACP算法的应用场景是商品搜索。和DSTN算法在搜索广告中挖掘上下文广告、历史点击广告、历史未点击广告的特征类似，RACP算法的作者也认为搜索结果页（Search Result Page）中的多个商品对于用户是否点击存在相互影响，而用户搜索过程是一个兴趣逐渐聚焦的过程，搜索历史中的多个搜索结果页表征了用户兴趣演进过程，如图23所示，影响当前搜索结果页中的商品是否被点击，因此，RACP算法先对搜索历史中每个搜索结果页的多个商品的点击和不点击数据进行建模，以挖掘每个搜索结果页的上下文感知的用户兴趣特征，再对搜索历史中多个搜索结果页的兴趣演进过程进行建模，最后得到用户兴趣演进特征，将其和目标商品特征、用户画像特征拼接在一起，输入多层神经网络，最后由Sigmoid函数输出目标商品的预估点击率。
![图23 用户兴趣演进过程](https://cdn.nlark.com/yuque/0/2023/png/1332969/1699873862211-c386e6b2-4ccc-41ee-a14b-9310e0371d02.png#averageHue=%23f6f2ef&clientId=u1576d8d1-920e-4&from=paste&height=239&id=u4577cc88&originHeight=478&originWidth=974&originalType=binary&ratio=2&rotation=0&showTitle=true&size=163786&status=done&style=none&taskId=uc54c38f2-d01f-4ab1-93cb-e194e79451f&title=%E5%9B%BE23%20%E7%94%A8%E6%88%B7%E5%85%B4%E8%B6%A3%E6%BC%94%E8%BF%9B%E8%BF%87%E7%A8%8B&width=487 "图23 用户兴趣演进过程")
![图24 RACP网络结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1699879893914-c0faf655-6f59-441e-8dee-1fe71df06ebc.png#averageHue=%23f7f3ee&clientId=udca76565-4c25-4&from=paste&height=424&id=ud947171f&originHeight=848&originWidth=1442&originalType=binary&ratio=2&rotation=0&showTitle=true&size=354347&status=done&style=none&taskId=u5b195ffe-731e-4471-bb9d-ab8a6106ccc&title=%E5%9B%BE24%20RACP%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=721 "图24 RACP网络结构")
### 多场景学习
多任务学习、多场景学习的主要区别如图25所示，多任务学习是对同一样本数据集的多个不同类型的任务进行联合学习，而多场景学习是对多个不同场景样本数据集的同一类任务进行联合学习。
![图25 多任务学习、多场景学习的主要区别](https://cdn.nlark.com/yuque/0/2024/png/1332969/1714878772639-2706bc3e-54c2-44fc-a376-54ac8fb56d31.png#averageHue=%23f9f8f8&clientId=u2b33ab06-2677-4&from=paste&height=188&id=Ljk9o&originHeight=217&originWidth=700&originalType=binary&ratio=1&rotation=0&showTitle=true&size=92013&status=done&style=none&taskId=u3d7eb824-fcd6-4ca7-87c2-c5597234300&title=%E5%9B%BE25%20%E5%A4%9A%E4%BB%BB%E5%8A%A1%E5%AD%A6%E4%B9%A0%E3%80%81%E5%A4%9A%E5%9C%BA%E6%99%AF%E5%AD%A6%E4%B9%A0%E7%9A%84%E4%B8%BB%E8%A6%81%E5%8C%BA%E5%88%AB&width=605 "图25 多任务学习、多场景学习的主要区别")
#### STAR-基于星型拓扑的多场景点击率预估学习
《One Model to Serve All: Star Topology Adaptive Recommender for Multi-Domain CTR Prediction》是阿里妈妈广告算法团队于2021年发表的一篇论文，如标题所示，论文提出了STAR模型（Star Topology Adaptive Recommender），用于解决不同流量场景下的广告点击率预估问题。STAR的核心思想是在原有深度神经网络模型的基础上，将模型参数划分为两部分，一部分是所有场景共享的参数，用于挖掘所有场景共性的知识，另一部分是各个场景专有的参数，用于挖掘各个场景个性的知识，如此模型参数构成了以所有场景共享参数为中心、各个场景专有参数为周边的星型拓扑（Star Toplogy），这也是模型名称的由来。STAR在训练和推理时，所有场景共用一个网络模型，只是针对当前训练或推理样本所属的场景，将所有场景共享参数和当前场景专有参数逐个相乘，得到最终的模型参数，从而实现通过一个通用模型，在不过多增加模型参数的基础上，同时挖掘不同流量场景的共性和个性知识，提升不同流量场景下广告点击率预估的准确性。
![图26 单场景广告点击率预估和多场景广告点击率预估（STAR）的网络结构](https://cdn.nlark.com/yuque/0/2024/png/1332969/1709794213916-80833128-3464-4d2f-a9dd-b9a18af9ee9e.png#averageHue=%23efefee&clientId=u134fc435-1062-4&from=paste&height=483&id=nZsy2&originHeight=1140&originWidth=1566&originalType=binary&ratio=2&rotation=0&showTitle=true&size=863112&status=done&style=none&taskId=u3621d9eb-6b47-4841-a82a-dcd0c96161f&title=%E5%9B%BE26%20%E5%8D%95%E5%9C%BA%E6%99%AF%E5%B9%BF%E5%91%8A%E7%82%B9%E5%87%BB%E7%8E%87%E9%A2%84%E4%BC%B0%E5%92%8C%E5%A4%9A%E5%9C%BA%E6%99%AF%E5%B9%BF%E5%91%8A%E7%82%B9%E5%87%BB%E7%8E%87%E9%A2%84%E4%BC%B0%EF%BC%88STAR%EF%BC%89%E7%9A%84%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=664 "图26 单场景广告点击率预估和多场景广告点击率预估（STAR）的网络结构")
## 转化率预估
### ESMM-点击率预估和转化率预估的联合学习
《Entire Space Multi-Task Model: An Effective Approach for Estimating Post-Click Conversion Rate》是阿里妈妈广告算法团队于2018年发表的一篇论文，如标题所示，论文提出了ESMM模型（Entire Space Multi-Task Model），基于用户行为序列和多任务学习思想有效解决广告转化率（Conversion Rate，CVR）预估的样本选择偏差（Sample Selection Bias，SSB）和数据稀疏（Data Sparsity，DS）问题，并在淘宝推荐广告的公开和线上数据集上取得了当时最好的效果。
ESMM的网络结构如图27所示，其属于多任务学习中的Shared Bottom方案，训练样本数据集为广告展现，每个样本包含用户和广告等特征，样本通过共享的Embedding层和池化层得到向量表征，向量表征再分别输入转化塔和点击塔，转化塔和点击塔的网络结构为多层全连接神经网络，由转化塔和点击塔分别输出预估转化率（$\text{pCVR}$）和预估点击率（$\text{pCTR}$），再将两者相乘，得到预估点击转化率（$\text{pCTCVR}$），训练时的两个任务为点击转化率预估和点击率预估，训练时的损失函数也相应为两部分之和，两部分分别是点击转化率预估值和真实值的交叉熵损失函数，以及点击率预估值和真实值的交叉熵损失函数，推理时直接使用转化塔输出的预估转化率。ESMM有效解决了转化率预估中的选择偏差和数据稀疏问题，在转化率预估上取得了较好的效果，因此被业界广泛应用。
![图27 ESMM网络结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1683253630653-29f40b50-1ddd-48eb-833d-6f128e453aa5.png#averageHue=%23ebebeb&clientId=u2f705e69-af91-4&from=paste&height=414&id=l57eX&originHeight=832&originWidth=1066&originalType=binary&ratio=2&rotation=0&showTitle=true&size=265055&status=done&style=none&taskId=u60dea79b-c321-4753-8a3f-354e9d446f6&title=%E5%9B%BE27%20ESMM%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=531 "图27 ESMM网络结构")
### MMM-多流量场景、多转化类型下的转化率预估学习
《Masked Multi-Domain Network: Multi-Type and Multi-Scenario Conversion Rate Prediction with a Single Model》是UC头条广告算法团队于2023年发表的一篇论文，如标题所示，论文提出了MMM模型（Masked Multi-Domain Network），用于解决不同流量场景、不同转化类型下的广告转化率预估问题。MMM的核心思想是在ESMM的基础上，针对每个流量场景和转化类型配对所构成的场景，构建相应的转化塔对该场景的广告转化率进行预估，针对流量场景和转化类型两两配对导致场景过多、模型过大、样本过少的问题，论文将转化塔的每个参数划分为三部分，分别是所有场景共享的参数，每个流量场景专有的参数和每个转化类型专有的参数，通过将上述三部分参数组合，便能得到某个流量场景和转化类型配对所构成场景对应转化塔的参数，这样便能减少模型参数量，同时针对流量场景和转化类型两两配对导致场景过多、训练样本集被按照场景划分为多个子集、较难对模型进行统一训练的问题，论文引入自动掩码（Auto-Masking）技术，在统一使用一个训练样本集进行模型训练的前提下，通过自动掩码控制每个训练样本只对该样本场景所对应转化塔的输出有值，从而减少模型训练的复杂度，最后，论文还对损失函数进行优化，通过对不同场景下训练样本所得的损失值进行加权调整，提升模型预估效果。
![图28 MMN的网络结构](https://cdn.nlark.com/yuque/0/2024/png/1332969/1709794411351-65fe0466-6bb7-4480-aedd-403e3a57053e.png#averageHue=%23f5f4f4&clientId=u134fc435-1062-4&from=paste&height=450&id=Dz3Wl&originHeight=900&originWidth=1142&originalType=binary&ratio=2&rotation=0&showTitle=true&size=614473&status=done&style=none&taskId=u86e9430f-cbb5-4190-901f-890e4271623&title=%E5%9B%BE28%20MMN%E7%9A%84%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=571 "图28 MMN的网络结构")
## 拍卖机制
从参与方的维度来看，效果类在线广告投放平台涉及三方的博弈：广告主、媒体、广告平台。广告主希望在投放成本满足一定的约束条件（如$\text{ROI}$和$\text{tCPA}$）前提下，竞得更多的优质流量，获得更多的$\text{GMV}$增长，媒体希望在获得更多流量变现的同时，能够保持或优化用户的使用体验，对用户展现优质和相关的广告，而广告平台在提升广告主和媒体匹配效率、尽可能满足两者上述诉求的同时，还要提升自身平台的收入。
在线广告投放流程相当于由上述三方参与的广告拍卖流程：在一次广告请求中，由媒体提供广告位作为竞拍品，由广告主出价进行竞拍，而由广告平台根据某种广告拍卖机制，先采用某种分配规则排序、选择胜出的广告主，再采用某种定价规则计算胜出广告主的定价，从而最大化各方的收益，使得整体处于博弈均衡的状态。
令一次广告请求中的广告位个数为$K$。广告平台一般会根据广告主预设的定向条件和当前请求的上下文特征从海量广告全集中召回相匹配的广告候选集，令广告候选集中有$N$个广告主，每个广告主针对广告位的私有估值为$v_i$，对外报价为$b_i$（这里隐含假设广告主对每个广告位的私有估值和对外报价是相同的）。一般情况下，$b_i\le v_i$。令$b_{-i}$表示除广告主$i$外其他广告主的报价，则$b=(b_i,b_{-i})$表示所有广告主的报价。
令广告拍卖机制为$\mathcal{M}(\mathcal{R},\mathcal{P})$，其中$\mathcal{R}$表示分配规则，$\mathcal{P}$表示定价规则。在广告分配时，令$\mathcal{R_i(b_i,b_{-i})}=k$表示广告主$i$竞得第$k$个广告位，$\mathcal{R}_i(b_i,b_{-i})=0$表示广告主$i$竞拍失败，最终会有$K$个广告主竞得广告位。在广告定价时，令$\mathcal{P}_i$表示广告主$i$的定价，则$u_i=v_i-\mathcal{P}_i$表示广告主$i$的效用（utility），即利润。通过设计分配规则和定价规则，使得平台、广告主各方处于博弈均衡的状态。而GSP的分配和计价规则如下：

- 分配规则，将eCPM作为排序分从高到低对候选广告进行排序；
- 计价规则，将下一个广告的出价作为当前广告的计价，其中，$\Delta$一般是货币的最小单位，比如1分：

$p_i=\frac{b_{i+1}*pCTR_{i+1}}{pCTR_i}+\Delta$
可以论证，GSP是单广告位拍卖场景下，社会福利最大化的优势策略激励相容机制，即所有广告主最优策略是如实报价，并通过竞拍广告位能获取最大的效用。而在多广告位拍卖场景下，最优拍卖机制是VCG，但由于其逻辑比较复杂，实际应用比较少。
### 多目标优化
随着计算广告的不断发展，如何实现广告主、投放平台和媒体的三方共赢和可持续发展，这一问题即要求拍卖机制能够实现多目标优化，同时考虑三方的目标，包括广告主由广告带来的成交金额提升（GMV）、投放平台的广告收入提升（CPM）和媒体的用户体验不受影响（CTR）等，业界也有相应的论文和工作探索如何实现拍卖机制的多目标优化。
#### DeepGSP-强化学习和GSP结合的多目标优化的拍卖机制
《Optimizing Multiple Performance Metrics with Deep GSP Auctions for E-commerce Advertising》是阿里妈妈算法团队于2021年发表的论文，其中提出了基于强化学习的多目标优化的广告拍卖机制——Deep GSP。
前面已经提到，广告拍卖涉及三方参与者：广告主、媒体、广告平台。每个参与者都有各自的优化目标，而且相互之间的目标可能存在冲突，例如，对广告分配所使用的eCPM公式，提高出价权重，降低点击率权重，则优先展示高出价、低点击率的广告，从而一方面虽能提高广告平台收入，但另一方面也影响了用户体验。另外有些优化目标在广告请求时较难通过模型实时、准确地预估，例如，电商类广告主的GMV指标，依赖于用户在广告点击后的一系列后链路行为，包括加购、下单等，只有延迟收集到这些行为反馈后，才能准确计算GMV指标。
因此在这篇论文中，不采用固定的公式，将出价和预估点击率、预估转化率相乘作为广告分配时的排序分，而是将出价、预估点击率、预估转化率、广告类别、商品价格等信息作为深度神经网络的特征，将多目标优化作为深度神经网络的目标函数，由深度神经网络充分挖掘特征和目标函数的内在联系，从而通过深度神经网络基于特征预估排序分，实现多目标优化。
另外，论文中还提到，由深度神经网络预估的排序分，后续会用于广告拍卖机制中的分配和定价，因此，排序分需要满足广告拍卖机制中的相关约束：

- 单调分配（Monotone Allocation），即排序分随出价的变化满足单调性，从而使得广告拍卖机制满足博弈均衡的特性，包括单广告位场景下的激励相容（Incentive Compatibility，IC）和多广告位场景下的对称纳什均衡（Symmetric Nash Equilibrium，SNE）；
- 平滑切换（Smooth Transition），即多目标优化中的各指标权重发生变化时，广告主侧的优化目标不会有明显的波动；

Deep GSP中的模型会在广告拍卖场景下根据出价和后验数据（GMV）不断调整，比较类似强化学习中的探索机制，因此，论文采用强化学习进行模型的训练和推理。强化学习是机器通过与环境交互来实现目标的一种计算方法。机器和环境的一轮交互是指，机器在环境的一个状态下做一个动作决策，把这个动作作用到环境当中，这个环境发生相应的改变并且将相应的奖励反馈和下一轮状态传回机器。这种交互是迭代进行的，机器的目标是最大化在多轮交互过程中获得的累积奖励的期望。
在Deep GSP机制下，强化学习涉及的状态、动作和奖励定义如下：

- 状态，广告$i$的状态$s_i$包括以下三部分信息：广告信息（出价、点击率、转化率等），广告主信息（预算、营销目标等），用户信息（性别、年龄、收入、购物偏好等），即$(b_i,x_i)$；
- 动作，广告$i$的动作$a_i$即由状态$s_i$输入排序分模型后输出的排序分$r_i$；
- 奖励，获取排序分后，按照Deep GSP机制进行广告位拍卖，胜出的广告被展现后收集反馈数据、评估多个广告效果指标，奖励包括两部分：一是多个广告效果指标的线性组合，越大越好，二是平滑切换约束中广告主在新拍卖机制下的效用负向波动，越小越好，因此奖励可表示为：

$re_i=F(b;\mathcal{M})-\eta\times\max(0,(1-\epsilon)\times\bar{u}_i(\mathcal{M}_0)-u_i(\mathcal{M}))$
其中$F(b;\mathcal{M})=\sum_j{w_j\times f_j}$。
强化学习的优化目标就是通过训练一个策略函数$R_\theta^*$，使得其获取的奖励最大，从而实现Deep GSP机制下的广告效果指标多目标优化，$R_\theta^*$可表示为：
$R_\theta^\ast=\argmax_{R_\theta}{\mathbb{E}_{b\sim\mathcal{D}}[re_i|R_\theta]}$
如何实现强化学习的优化目标，论文这里使用了基于Actor-Critic算法的DDPG（Deep Deterministic Policy Gradient，深度确定性策略梯度）算法。论文用图29比较形象地描述了DDPG算法在Deep GSP广告拍卖机制下的流程，其中，红色箭头表示前向传播，蓝色箭头表示反向传播。前向传播链路中，用户侧发起广告请求，广告平台召回候选广告集，对于每个候选广告，使用Actor网络，输入广告特征，输出$\pi_\theta(b_i,x_i)$，和出价$b_i$相乘得到排序分$r_i$，基于排序分$r_i$进行分配和计价，最后胜出的广告向用户展示，用户可能点击广告并发生进一步的转化行为，另外，前向传播链路中，对于每个候选广告，还使用Critic网络，输入广告特征和Actor网络输出的$\pi_\theta(b_i,x_i)$，输出$Q_i$。反向传播链路中，收集用户行为反馈，计算多个广告效果指标，并进而计算奖励$re_i$，基于$Q_i$和$re_i$之间的差值，计算Critic网络的损失函数值和梯度，并更新Critic网络的参数，再进而更新Actor网络的参数。
![图29 DDPG算法在Deep GSP广告拍卖机制下的流程](https://cdn.nlark.com/yuque/0/2023/png/1332969/1694523032508-3af76fe0-1c3e-4fe3-b3eb-d921a3a3f552.png#averageHue=%23f3f0ef&clientId=u4114c901-6606-4&from=paste&height=248&id=Df3lz&originHeight=496&originWidth=1714&originalType=binary&ratio=2&rotation=0&showTitle=true&size=119641&status=done&style=none&taskId=ue955d402-f058-4cf6-8e2f-6f4934680c1&title=%E5%9B%BE29%20DDPG%E7%AE%97%E6%B3%95%E5%9C%A8Deep%20GSP%E5%B9%BF%E5%91%8A%E6%8B%8D%E5%8D%96%E6%9C%BA%E5%88%B6%E4%B8%8B%E7%9A%84%E6%B5%81%E7%A8%8B&width=857 "图29 DDPG算法在Deep GSP广告拍卖机制下的流程")
#### DNA-端到端学习的多目标优化的拍卖机制
《Neural Auction: End-to-End Learning of Auction Mechanisms for E-Commerce Advertising》是阿里妈妈算法团队于2021年发表的另一篇论文，其中提出了另一种基于深度学习的多目标优化的广告拍卖机制——DNA（Deep Neural Auction）。
DNA需要解决的问题是：在满足激励兼容（每个广告主都如实报价）和个体独立（每个广告主都获得非负的效用）的前提下，最大化多个广告效果指标（RPM、CTR、GMV等）的线性组合。
而激励兼容的充要条件是：分配规则$\mathcal{R}$单调分配（Monotone Allocation），即胜出的竞拍者如果提高出价仍能够赢得拍卖，且定价规则$\mathcal{P}$基于最低出价（Critical Bid based Pricing），即定价是维持当前胜出状态的最低出价。另外，可以论证若机制满足上述条件，则其也满足个体独立。
因此，DNA需要解决的问题可以进一步转化为：和Deep GSP类似，设计一个包含分配规则和定价规则的广告拍卖机制，分配规则中，排序分随出价单调变化，定价规则中，取排在后一位的广告的排序分逆向计算当前广告的定价。
DNA的整体架构如图30下方所示，如标题所描述，其整体是一个端到端的解决方案，推理时，输入是候选广告集合，整体链路前向传播后，输出是胜出广告集合，训练时，整体链路反向传播更新参数。整体链路主要包括以下3个模块：

- 集合编码器（Set Encoder），将候选广告集合作为特征进行Embedding，因为候选广告集合是无序的，所以使用满足置换不变性的集合编码器；
- 上下文感知的排序分网络（Context-Aware Rank Score Network），所有候选广告共享一个排序分网络，排序分网络的输入包括候选广告本身的特征（如出价）、和作为上下文的候选广告集合Embedding特征，排序分网络输出排序分；排序分网络满足单调性——即排序分随候选广告的出价单调变化，同时，排序分网络支持逆向操作，取排在后一位的广告的排序分逆向计算当前广告的定价；
- 可导的排序操作（Differentiable Sorting Operator），通常排序操作是离散且不可导的，为了实现端到端的解决方案，需要整体链路可导从而支持反向传播更新参数，因此论文实现了可导的排序操作。

基于上述3个模块，对候选广告集合进行排序，选取靠前的若干个广告胜出，并基于排序分网络进行逆向操作计算定价，从而实现广告拍卖的分配和定价。
![图30 DNA整体架构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1694582973556-2b1caf58-65f9-4926-a3d7-111025124558.png#averageHue=%23656564&clientId=u4114c901-6606-4&from=paste&height=353&id=bpXgd&originHeight=522&originWidth=942&originalType=binary&ratio=2&rotation=0&showTitle=true&size=56444&status=done&style=none&taskId=ua9612ef7-9771-41d6-8126-9ec7b3c6ef3&title=%E5%9B%BE30%20DNA%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84&width=637 "图30 DNA整体架构")
## 出价
广告主在投放效果广告时，会对广告流量设置一定的出价，而广告平台会基于多个广告主的出价进行广告流量的竞价，对竞价胜出的广告主，在广告流量上展示其广告，并进行计费。随着效果广告业务和技术的不断发展，广告主的出价功能也不断迭代，其迭代可分为三个阶段：在第一个阶段，广告主在广告的展现或点击上进行手动出价，广告平台在广告的展现上进行竞价，广告主在点击上的出价会通过乘以预估点击率转化为eCPM，从而在展现上进行竞价，最终在展现或点击上进行计费（即CPM和CPC），这一阶段广告的投放效果、特别是后链路的转化效果（如点击后的转化成本）依赖广告主根据效果数据和投放经验手动调整出价，人力成本较高、且效果难以保障；在第二个阶段，广告主在广告的转化上进行手动出价（即设置转化的目标成本），广告平台在广告的展现上进行竞价，广告主在转化上的出价会通过乘以预估点击率和预估转化率转化为eCPM，从而在展现上进行竞价，最终在展现或点击上进行计费（即oCPM和oCPC），这一阶段广告的转化效果（如点击后的转化成本）由广告平台的投放系统根据广告主设置的目标成本和预估点击率、预估转化率自动调整在展现或点击上的出价进行保障，从而降低人力成本、提升投放效果；在第三个阶段，广告主无需在展现、点击和转化上进行手动出价，仅需设置一定的约束条件，如每日广告投放预算，而由广告平台的投放系统根据广告流量和投放效果自动调整出价，为广告主实现广告流量价值的最大化。
### 基于强化学习的自动出价
第三个阶段中广告平台投放系统为广告位进行自动出价的问题即预算约束下的出价问题（Budget Constrained Bidding，BCB），其目标可用以下公式表示：
$\begin{align}
\max{\sum_{i=1\dots N}{x_iv_i}}\\
\text{s.t.}\space\sum_{i=1}^N{x_ic_i\le B}
\end{align}$
其中，每天共有$N$次广告流量，对于$i$次广告流量，$v_i$表示广告流量对于广告主的价值，$b_i$表示根据$v_i$计算得到的广告主对于广告流量的出价，即：
$b_i=v_i/\lambda$
$x_i$表示根据广告主的出价和其他广告主进行竞价后的结果（1表示竞胜，0表示未竞胜），$c_i$表示广告主在竞胜后需要支付的费用（一般由广告平台投放系统按照GSP机制使用竞价排序中下一位广告主的出价计算所得），则自动出价问题即需要为广告主提供一个策略，该策略可根据广告流量价值计算广告出价，使得最终广告投放费用在不超过预算的前提下，最大化广告流量价值。
由于广告流量和竞价环境的动态性，因此业界有较多工作基于强化学习进行自动出价，使用强化学习与环境交互、收集轨迹数据、迭代更新价值和策略函数的特点，解决广告流量和竞价环境动态性的问题。
#### DRLB-基于无模型强化学习的自动出价算法
2018年发表的论文《Budget Constrained Bidding by Model-free Reinforcement Learning in Display Advertising》提出了基于无模型强化学习的自动出价。
这里的无模型是指不对环境进行建模、不直接计算状态转移概率，智能体只能和环境进行交互，通过采样得到的数据进行学习，这类学习方法统称为无模型的强化学习（Model-free Reinforcement Learning）。无模型的强化学习又可以分为基于价值和基于策略的算法，基于价值的算法主要是学习价值函数，然后根据价值函数导出一个策略，学习过程中并不存在一个显式的策略，而基于策略的算法则是直接显式地学习一个策略函数，另外，基于策略的算法中还有一类Actor-Critic算法，其会同时学习价值函数和策略函数。
论文具体使用DQN算法，该算法将强化学习中基于价值的算法和深度学习相结合。另外，论文在该算法的基础上，针对自动出价场景的特点进行优化。
论文使用带约束的马尔科夫决策过程（Constrained Markov Decision Process，CMDP）对自动出价问题进行建模，如图31所示，其分为离线训练环境和在线预测环境。
离线训练环境包含仿真竞价系统，以模拟线上真实竞价系统。自动出价智能体在离线训练环境进行$K$轮迭代，每轮迭代内，自动出价智能体模拟一天内的广告投放，与仿真竞价系统进行$T$步交互（将每天平均分成$T$个时间段，每个时间段为15分钟，即$T$为96），在第$t$步，智能体使用策略根据状态$s_t$选择动作$a_t$，将出价调控系数由$\lambda_{t-1}$调整至$\lambda_t$，则随后智能体对第$t$步和第$t+1$步之间的每个广告流量的出价使用公式$v_i/\lambda_t$计算得到，并将第$t$步和第$t+1$步之间竞得的广告流量价值之和作为第$t$步的动作$a_t$所获得的奖励$r_t$，第$t$步和第$t+1$步之间竞得的广告流量费用之和为$c_t$。智能体在$K$轮迭代中，通过与仿真竞价系统的交互，获取反馈数据，不断学习、更新策略，调整出价调控系数，在满足预算约束（即$\sum_{t=1}^T{c_t}\le B$）的前提，最大化累积奖励$\sum_{t=1}^T{r_t}$。
在线预测环境即线上真实竞价环境。与离线训练环境类似，在线预测环境中的自动出价智能体在一天内的广告投放中，与真实竞价系统进行$T$步交互，在第$t$步，智能体使用策略根据状态$s_t$选择动作$a_t$，将出价调控系数由$\lambda_{t-1}$调整至$\lambda_t$，但与离线训练环境不同，在线预测环境中的自动出价智能体不再学习、更新策略。
![图31 问题建模](https://cdn.nlark.com/yuque/0/2024/png/1332969/1707297004895-3410dcc2-f9ca-4fc2-b717-677bc5de4362.png#averageHue=%23efefef&clientId=u90620007-bd18-4&from=paste&height=283&id=xqh04&originHeight=510&originWidth=1190&originalType=binary&ratio=1&rotation=0&showTitle=true&size=311080&status=done&style=none&taskId=u4d5f6f11-08b6-433f-bad6-4450db91023&title=%E5%9B%BE31%20%E9%97%AE%E9%A2%98%E5%BB%BA%E6%A8%A1&width=660 "图31 问题建模")
论文在原DQN算法的基础上进行了两处改进：一是设计新的奖励函数，解决原有奖励函数只考虑最近一个时间段内的广告流量价值之和，从而导致策略忽视预算约束、倾向于尽快消耗预算、算法得到次优解的问题，二是设计新的探索策略，解决原$\epsilon$-贪婪策略在动作价值随动作的变化为多峰分布时、探索不充分的问题。
#### USCB-带约束自动出价统一方案
2021年阿里妈妈发表的论文《A Unified Solution to Constrained Bidding in Online Display Advertising》针对带约束自动出价除预算约束以外、还存在其他多种类型约束的问题，提出了带约束自动出价统一方案——USCB（Unified Solution to Constrained Bidding）。该方案对各种类型约束下的自动出价进行统一建模，推导出通用的出价计算公式，并使用无模型强化学习中同时学习价值函数和策略函数的Actor-Critic算法之一——DDPG算法，先离线与仿真竞价系统交互进行训练、再在线与真实竞价系统交互进行预测，输出出价计算公式中各参数最优值，从而计算出价并竞价，最终实现各种类型约束下广告流量价值的最大化。
自动出价目标可由下式表示：
$\begin{align}
\max{\sum_{i=1\dots N}{x_iv_i}}\\
\text{s.t.}\space\sum_{i=1}^N{x_ic_i\le B}
\end{align}$
其中，每天共有$N$次广告流量，对于$i$次广告流量，$v_i$表示广告流量对于广告主的价值，$b_i$表示广告主对于广告流量的出价，$x_i$表示根据广告主的出价和其他广告主进行竞价后的结果（出价高者竞胜，1表示竞胜，0表示未竞胜），$c_i$表示广告主在竞胜后需要支付的费用（一般由广告平台投放系统按照GSP机制使用竞价排序中下一位广告主的出价计算所得），则自动出价问题即需要为广告主提供一个策略，该策略可根据广告流量价值计算广告出价，使得最终广告投放费用在不超过预算的前提下，最大化广告流量价值。
另外，不同的广告主在投放广告时，除了引入预算约束外，还会引入其他类型的效果指标作为约束条件，这些约束条件可以分为两大类：一类是成本相关的约束条件（Cost-Related，CR），包括单次点击成本（Cost Per Click，CPC）和单次转化成本（Cost Per Action，CPA）等；另一类是非成本相关的约束条件（Non-Cost-Related，NCR），包括点击率（Click Through Rate，CTR）和转化率（Conversion Per Impression，CPI）等。上述两类约束条件中的每一个约束条件可由下式统一表示：
$\frac{\sum_i{\mathbf{c}_{ij}x_i}}{\sum_i{\mathbf{p}_{ij}x_i}}\le \mathbf{k}_j$
其中，$\mathbf{k}_j$表示约束条件$j$的上界，$\mathbf{p}_{ij}$表示约束条件$j$下、展现$i$的某个效果指标或常量，$\mathbf{c}_{ij}$可由下式计算：
$\mathbf{c}_{ij}=c_i\mathbf{1}_{CR_j}+\mathbf{q}_{ij}(1-\mathbf{1}_{CR_{j}})$
其中，$\mathbf{1}_{CR_j}$表示是否为成本相关的约束条件（1表示是，0表示否），即对于成本相关的约束条件，$\mathbf{c}_{ij}$为广告主在竞胜展现$i$后需要支付的费用成本$c_i$，对于非成本相关的约束条件，$\mathbf{c}_{ij}$为约束条件$j$下、展现$i$的某个效果指标或常量$\mathbf{q}_{ij}$。
论文进一步列出多个具体约束条件中、上述统一表示公式各项的取值，如图32所示。例如，对于单次点击成本（CPC）约束，$\mathbf{c}_{ij}$为$c_i$，$\mathbf{p}_{ij}$为$\text{CTR}_i$，则统一表示公式的分子为费用成本和，分母为点击次数，相除后为单次点击成本，$\mathbf{k}_j$为单次点击成本的上界；对于点击率（CTR）约束，$\mathbf{c}_{ij}$为1，$\mathbf{p}_{ij}$为$\text{CTR}_i$，则统一表示公式的分子为展现次数，分母为点击次数，相除后为点击率的倒数，$\mathbf{k}_j$为点击率下界的倒数。
![图32 多个具体约束条件中、上述统一表示公式各项的取值](https://cdn.nlark.com/yuque/0/2024/png/1332969/1705846451361-ebae83a1-d1a9-4614-aa4f-d6435998b9bc.png#averageHue=%23d9d9d8&clientId=u9e68f905-a682-4&from=paste&height=270&id=EiJaC&originHeight=328&originWidth=810&originalType=binary&ratio=1&rotation=0&showTitle=true&size=174945&status=done&style=none&taskId=ud416d101-f432-4f24-a41e-f33323e6f1a&title=%E5%9B%BE32%20%E5%A4%9A%E4%B8%AA%E5%85%B7%E4%BD%93%E7%BA%A6%E6%9D%9F%E6%9D%A1%E4%BB%B6%E4%B8%AD%E3%80%81%E4%B8%8A%E8%BF%B0%E7%BB%9F%E4%B8%80%E8%A1%A8%E7%A4%BA%E5%85%AC%E5%BC%8F%E5%90%84%E9%A1%B9%E7%9A%84%E5%8F%96%E5%80%BC&width=666 "图32 多个具体约束条件中、上述统一表示公式各项的取值")
基于上述约束条件的统一表示，论文给出带约束自动出价问题的统一表示：
$\begin{align}
\max_{x_i}\space\sum_i{v_ix_i} \\
\text{s.t.}\space\sum_i{c_ix_i}\le B\\
\frac{\sum_i{\mathbf{c}_{ij}x_i}}{\sum_i{\mathbf{p}_{ij}x_i}}\le \mathbf{k}_j,\space\forall j\\
x_i\le 1,\space\forall i\\
x_i\ge 0,\space\forall i\\
\end{align} \tag{LP1}$
论文进一步推导出上述线性规划问题取得最优解时，每次展现的出价可由下式表示：
$b_i^*=w_0^*v_i-\sum_j{w_j^*(\mathbf{q}_{ij}(1-\mathbf{1}_{CR_j})-\mathbf{k}_j\mathbf{p}_{ij})}$
从上式可以看出，每次展现的出价依赖参数$\{w_k^*\}_{k=0}^M$，即将自动出价问题转化为该$M+1$个参数的求解。
USCB也使用马尔科夫决策过程（Markov Decision Process，MDP）对自动出价问题进行建模，将一天内的广告投放划分为96个时间段，每个时间段为15分钟，在第$t$步，自动出价智能体使用策略$\pi:\mathcal{S}\mapsto\mathcal{A}$，根据状态$s_t\in\mathcal{S}$，选择动作$a_{0t},a_{1t},\dots,a_{Mt}\in\mathcal{A}$（$\mathcal{A}=\mathcal{A}_0\times\mathcal{A}_1\times\dots\times\mathcal{A}_M\subseteq\reals^{M+1}$），修改出价公式相应各参数，然后自动出价智能体基于修改后的参数和竞价系统进行交互，计算出价并竞价，将第$t$步和第$t+1$步之间竞得的广告流量价值之和作为第$t$步所获得的奖励$r_t:\mathcal{S}\times\mathcal{A}\mapsto\mathcal{R}\subseteq\reals$，自动出价智能体的优化目标即最大化一天内的奖励之和$R=\sum_{t=1}^T{\gamma^{t-1}r_t}$。
上一节已提到无模型的强化学习分为基于价值和基于策略的算法，基于价值的算法主要是学习价值函数，然后根据价值函数导出一个策略，学习过程中并不存在一个显式的策略，而基于策略的算法则是直接显式地学习一个策略函数，另外，基于策略的算法中还有一类Actor-Critic算法，其会同时学习价值函数和策略函数。USCB具体使用了Actor-Critic算法中的DDPG算法。DDPG算法使用Actor网络和Critic网络分别拟合策略函数和价值函数。DDPG算法在和环境的交互中，先使用Actor网络根据状态得到动作，再使用Critic网络根据状态和动作得到动作价值。USCB中DDPG算法的实现与原始DDPG算法基本一致，其中几处细节是：

- USCB算法的每轮迭代，随机选择一个广告计划，并使用模拟竞价系统作为环境，以模拟该广告计划在一天内的广告投放，投放中，自动出价智能体与环境进行多步交互，每步交互根据状态选择动作调整出价参数，并使用调整后的出价参数对后续广告流量计算出价、参与竞价。
- 自动出价智能体与环境的每步交互中，首先使用策略网络根据状态$s_t$预测动作，并在预测结果上增加随机噪声，得到最终的动作$\overrightarrow{a}_t=\pi_\theta(s_t)+\mathcal{E}$，并根据动作$\overrightarrow{a}_t$调整出价参数：$\overrightarrow{w}_{t}=\overrightarrow{w}_{t-1}\cdot(1+\overrightarrow{a}_t)$。
#### PerBid-个性化自动出价方案
2023年阿里妈妈发表的论文《A Personalized Automated Bidding Framework for Fairness-aware Online Advertising》指出不同的广告主其广告投放目标和过程并不相同，基于统一方案构建一个自动出价智能体并应用于多个广告主，可能会导致该智能体为各广告主自动出价所带来的投放效果参差不齐，影响广告投放的整体公平性，因此该论文在USCB算法的基础上，提出了个性化自动出价方案——PerBid，该方案首先通过广告计划画像网络输出广告计划画像表征，能够表征广告计划的静态属性特征和动态环境特征，在此基础上，将广告计划划分为多个类簇，为每个类簇构建一个自动出价智能体，并且每个自动出价智能体将广告计划画像表征作为状态输入之一、从而感知不同广告计划动态环境的上下文，论文通过实验论证，该方案在保障投放效果的同时，也能提升广告投放的公平性。
和USCB类似，PerBid首先对带约束自动出价问题进行建模：
$\begin{align}
\max_{\mathbf{x}}{\sum_{i=1}^{n}{x_i\times v_i}}\\
\text{s.t.}\space\sum_{i=1}^{n}{x_i\times cost_i}\le Budget\\
\frac{\sum_{i=1}^{n}{x_i\times cost_i}}{\sum_{i=1}^{n}{x_i\times ctr_i}}\le PPC\\
x_i\in\{0,1\},\forall i\in[1,n]
\end{align}$
即在预算和单次点击成本约束下，最大化竞得的广告流量价值之和。
和USCB类似，PerBid也推导出最优出价公式：
$bid_i^*=\left(\alpha^*\times\frac{\frac{v_i}{ctr_i}}{average(\frac{v_i}{ctr_i})}+\beta^*\right)\times PPC$
其中，$\alpha^*$和$\beta^*$是需要求解的最优出价参数。
和USCB类似，PerBid也将自动出价问题转化为马尔科夫决策过程，自动出价智能体在第$t$步，根据状态$s_t$，通过策略$\pi$，输出动作$a_t$，调节出价参数，随后根据出价参数计算出价并竞价，获得奖励$r_t$，并在第$t+1$步，状态转移至$s_{t+1}$。令根据状态$s_t$选择动作$a_t$所获得的长期价值期望为$G(s_t,a_t)$，则策略$\pi$的优化目标即最大化上述长期价值期望，即$\pi(s)=\arg\max_a{G(s,a)}$。
为解决公平性问题，论文提出了个性化自动出价方案——PerBid（Personalized Automated Bidding Framework），其整体框架如图33所示。该方案首先通过广告计划画像网络输出广告计划画像表征，能够表征广告计划的静态属性特征和动态环境特征，在此基础上，将广告计划划分为多个类簇，为每个类簇构建一个自动出价智能体，并且每个自动出价智能体将广告计划画像表征作为状态输入之一、从而感知不同广告计划动态环境的上下文。
![图33 PerBid整体框架](https://cdn.nlark.com/yuque/0/2024/png/1332969/1705980341300-af44642a-893c-4099-a226-1b4650c0399c.png#averageHue=%23e5e6d4&clientId=uf1ce3805-c8ba-4&from=paste&height=288&id=hOLBa&originHeight=576&originWidth=828&originalType=binary&ratio=2&rotation=0&showTitle=true&size=358287&status=done&style=none&taskId=u40652718-8647-4259-8c13-740ceee756a&title=%E5%9B%BE33%20PerBid%E6%95%B4%E4%BD%93%E6%A1%86%E6%9E%B6&width=414 "图33 PerBid整体框架")
#### SOLA-离线强化学习和在线安全探索相结合的方案
之前所介绍的各自动出价方案，均先离线与仿真竞价系统交互进行训练、再在线与真实竞价系统交互进行预测，因此存在一个共性问题是如何保持仿真竞价系统和真实竞价系统的一致性，而真实竞价系统存在复杂的拍卖机制、激励的出价竞争，仿真竞价系统难以精确模拟真实竞价系统，而如果不能保持两个系统的一致性，则可能导致仿真竞价系统下所训练的自动出价方案在真实竞价系统中非最优。
为解决上述问题，一种方案是用真实竞价系统替代仿真竞价系统，让智能体通过与真实竞价系统的交互，进行策略学习，但直接替代并不可行，若直接替代，初始未训练的智能体与真实竞价系统交互会生成不安全的出价，影响广告投放效果。而另一种方案是使用离线强化学习，这里需要注意的是，离线强化学习中的“离线”和离线与仿真竞价系统交互进行训练的“离线”并不是一个概念，离线强化学习中的“离线”是指不与环境交互，使用预先已收集的轨迹数据进行训练，而在线强化学习则会与环境交互，收集交互后产生的轨迹数据进行训练，因此之前所介绍的各自动出价方案均属于在线强化学习，只是与智能体发生交互的是仿真竞价系统、非真实竞价系统，而离线强化学习因为不与环境交互，所以训练时也不需要仿真竞价系统，但同时也引入了另一个问题——分布偏移，因此，目前离线强化学习领域有较多的工作探索如何解决分布偏移问题，如CQL、IQL等。
阿里妈妈在2023年发表的论文《Sustainable Online Reinforcement Learning for Auto-bidding》，其提出的SOLA框架，将离线强化学习和在线安全探索相结合，同时解决了训练依赖仿真竞价系统和在线探索出价安全性的问题。
![图34 SOLA的整体框架](https://cdn.nlark.com/yuque/0/2024/png/1332969/1708331663725-1416eeea-d9b9-4be8-938b-1b45ec7952c0.png#averageHue=%23fbfcfb&clientId=u1030c07e-d302-4&from=paste&height=206&id=gkIYU&originHeight=412&originWidth=1458&originalType=binary&ratio=2&rotation=0&showTitle=true&size=630294&status=done&style=none&taskId=u3cb31ead-efe7-44a7-9f8c-52022e26765&title=%E5%9B%BE34%20SOLA%E7%9A%84%E6%95%B4%E4%BD%93%E6%A1%86%E6%9E%B6&width=729 "图34 SOLA的整体框架")
SOLA框架如图34所示，其整体分为两个阶段，在启动阶段（Warm Booting），SOLA使用由USCB算法训练所得的出价策略$\mu_s$在真实竞价系统中进行出价、获取轨迹数据集$\mathcal{D}_s=\{(s_k,a_k,r_k,s'_k)\}_k$，并基于$\mathcal{D}_s$使用V-CQL算法训练得到初始出价策略$\mu_0$；在迭代阶段（Iteration Process），SOLA会迭代多次，令第$\tau$次迭代的出价策略为$\mu_\tau$，在线安全探索策略为$\pi_{e,\tau}$，该次迭代在真实竞价系统中使用在线安全探索策略进行出价、获取的轨迹数据集为$\mathcal{D}_{on,\tau}$，然后基于已有累积的数据集使用V-CQL算法训练出价策略$\mu_\tau$，如次迭代，直至最终算法收敛。
# 粗排
粗排和精排均可以认为是排序（Learning to Rank，LTR）问题，而排序问题的求解一般有3种方式：

- Pointwise，逐个求解序列中各元素的优先级，从而得到整个序列的排序结果；
- Pairwise，序列中各元素两两配对，逐对求解两元素之间的优先级，从而得到整个序列的排序结果；
- Listwise，直接求解序列的排序结果。

这三种方法从上到下，从只考虑序列中元素自身、到考虑序列中两两元素相互关系再到考虑序列中所有元素相互关系，考虑的信息更加全面，但问题求解的样本空间也逐渐增大，从所有元素构成的样本空间、到所有元素两两配对构成的样本空间、再到所有元素组合序列构成的样本空间。
广告投放流程中提到的先预估点击率、再计算eCPM、最后根据eCPM进行排序的方法，则属于Pointwise类型的排序方法。
另一方面，粗排和精排虽然都是排序问题，但结合各自所处的广告投放环节，又有着不同的条件约束和演进方向。
精排需处理的广告数相对粗排较少，仅约数百个，因此可以在保证在线广告系统低延时、高吞吐的前提下，设计复杂的模型结构，精准地预估pCTR，从而计算eCPM，这方面的模型工作有DIN、DIEN等。而随着广告主愈加关注后链路转化效果，其偏向采用oCPC方式进行广告投放，即在转化上进行出价，因此，需要精排同时能预估转化率，并根据以下公式计算eCPM：eCPM=pCTR×pCVR×tCPA，转化率预估方面的模型工作有ESMM等。另外，由于精排和后续的定价属于广告位拍卖场景，涉及广告主、媒体、广告平台三方的博弈，因此精排和定价需要满足拍卖机制中的激励兼容（每个广告主都如实报价）和个体独立（每个广告主都获得非负的效用），同时，原先按照特定公式计算eCPM并排序的方式并不满足多目标优化（同时优化媒体用户体验、广告主GMV和平台收益），因此，需结合广告特征（包含pCTR和pCVR）、上下文特征，针对多目标优化，对精排和定价进行端到端的学习和推理，这方面的模型工作有Deep GSP、DNA等。
而粗排需处理的广告数相对精排较多，约数万个，另外，粗排的排序逻辑与精排近似，主要承担承上启下、初步筛选的作用，减少精排的计算量，因此，粗排一般使用相对简单的模型进行点击率、转化率的预估，在模型准确度和计算复杂度上达到均衡。基于这个思路，粗排模型不断发展，从简单的后验统计、到浅层模型、再到深层模型。虽然模型结构不断变化，但粗排均是Pointwise类型的排序方法，受限计算复杂度的约束，其预估的点击率、转化率难以对齐精排，而随着广告出价方式和调控机制的发展，eCPM的计算已不是简单的预估点击率和出价的相乘，这样导致粗排更难以通过预估点击率和出价相乘对齐精排的排序结果。因此，粗排模型的演进从Pointwise类型的排序方法——先预估每个广告的点击率、再计算eCPM进行排序，发展至Pairwise类型的排序方法——直接学习并推理精排的排序结果。
## 双塔模型-向量内积后再通过Sigmoid函数输出预估点击率
阿里妈妈在粗排中所使用的双塔模型如图35所示。令$e_u$为用户特征经过Embedding层后的Embedding向量，$e_a$为广告特征经过Embedding层后的Embedding向量。这两个Embedding向量分别经过各自的全连接网络后，得到用户向量$v_u$和广告向量$v_a$，对这两个向量求内积后再通过Sigmoid函数得到预估点击率：
$\begin{align}
y&=\sigma(v_u^Tv_a) \\
v_u&=\text{FC}(e_u) \\
v_a&=\text{FC}(e_a)

\end{align}$
![图35 双塔模型](https://cdn.nlark.com/yuque/0/2023/png/1332969/1698486829450-ed70c359-8cd2-4cf0-8c90-bf1ed4cee8ad.png#averageHue=%23eeeeed&clientId=u21ef724f-2cb5-4&from=paste&height=402&id=VNrXw&originHeight=804&originWidth=546&originalType=binary&ratio=2&rotation=0&showTitle=true&size=58872&status=done&style=none&taskId=u4aa26e35-6792-42a2-9386-2e4e86f5c6f&title=%E5%9B%BE35%20%E5%8F%8C%E5%A1%94%E6%A8%A1%E5%9E%8B&width=273 "图35 双塔模型")
从模型结构可以看出，虽然分别使用各自的子网络对用户特征和广告特征进行深度挖掘，但是用户特征和广告特征直至最终的Sigmoid函数才会交叉，导致模型的表达能力有所欠缺。
使用双塔模型进行粗排的工程实现如图36所示。每天使用历史的展现、点击日志进行模型训练、参数更新，然后离线对用户和广告集合中的所有用户和广告使用其特征计算其向量，并保存至索引中。在线推理时，对于每次广告请求，从索引中获取当前发起广告请求的用户的用户向量和召回返回的粗排候选广告集合中所有广告的广告向量，对于每个广告向量，计算其与用户向量的内积，并进而通过Sigmoid函数计算广告预估点击率，最后再使用eCPM公式得到粗排的广告排序分进行排序。
![图36 双塔模型进行粗排的工程实现](https://cdn.nlark.com/yuque/0/2023/png/1332969/1698231686936-b812299b-e7e0-45d5-bc5a-07b5875c9fe2.png#averageHue=%23efeadd&clientId=u1a74dbcc-64ff-4&from=paste&height=225&id=KHFj0&originHeight=450&originWidth=774&originalType=binary&ratio=2&rotation=0&showTitle=true&size=76949&status=done&style=none&taskId=u5657ca17-5001-4d4b-95a1-f37fdd34428&title=%E5%9B%BE36%20%E5%8F%8C%E5%A1%94%E6%A8%A1%E5%9E%8B%E8%BF%9B%E8%A1%8C%E7%B2%97%E6%8E%92%E7%9A%84%E5%B7%A5%E7%A8%8B%E5%AE%9E%E7%8E%B0&width=387 "图36 双塔模型进行粗排的工程实现")
从中可以看出，计算复杂度较高的用户向量和广告向量计算均可离线完成，在线推理时只需从索引中按主键获取相应的向量进行简单计算即可，大大减少了在线计算量，但是该工程实现也存在以下不足：所有用户和广告均需离线计算其向量，导致离线计算量较大，但实际有大量非活跃用户并不会发起广告请求、大量受众较少的广告也并不会被召回；同时，在离线计算后产生的新用户和新广告在索引中并没有其向量，导致无法对其进行粗排；另外，离线计算的用户和广告向量在全量更新至各自的索引时，会由于更新的时间差导致索引中用户和广告向量版本在一段时间内不一致，从而影响在线推理的准确度。
## COLD-算法优化和工程优化结合的轻量级粗排模型
《COLD: Towards the Next Generation of Pre-Ranking System》由阿里妈妈于2020年发表，介绍了其粗排模型基于Pointwise类型的排序方法，从简单的后验统计、到浅层模型、再到深层模型的演进历程，并主要介绍了其深层模型COLD在模型结构和工程实现上的优化思路，从而在保证在线系统性能要求的前提下，在模型准确度上取得较好的提升。
COLD的网络结构复用GwEN，但进行了轻量化的改造。轻量化改造的方案一般包括模型剪枝、特征筛选等。论文采用特征筛选方案，如图37所示，在原GwEN的Embedding层之后增加SE（Squeeze-and-Excitation）模块。Squeeze-and-Excitation Networks（SENet）是由自动驾驶公司Momenta于2017年公布的一种图像识别模型，其和2017年Google提出的Transformer模型一样，均采用了注意力机制，通过对特征通道间的相关性进行建模，把重要的特征进行强化来提升准确率，取得了2017年ILSVR竞赛的冠军。
![图37 COLD网络结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1698499706130-d8644c8a-cebf-4bde-bb0f-62b73f5e4e0b.png#averageHue=%23ededed&clientId=u21ef724f-2cb5-4&from=paste&height=363&id=DNJSu&originHeight=816&originWidth=566&originalType=binary&ratio=2&rotation=0&showTitle=true&size=64103&status=done&style=none&taskId=u34aa5dcc-cad0-4045-8dc2-5c53a1b9442&title=%E5%9B%BE37%20COLD%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=252 "图37 COLD网络结构")
论文使用SE模块计算各特征组Embedding向量的权重。令模型输入包含$M$个特征组，$e_i$表示第$i$个特征组的Embedding向量，向量的维度为$k$，则特征组Embedding向量的权重可通过以下方式计算：
$s=\sigma(W[e_1,\dots,e_m]+b)$
其中，$s\in\mathbb{R}^M$，$W\in\mathbb{R}^{k\times M}$，$b\in\mathbb{R}^M$，$s$的第$i$个元素$s_i$表示$e_i$的权重标量，使用$s_i$与$e_i$的每个元素进行相乘，得到$e_i$加权后的Embedding向量$v_i$，再将加权后的Embedding向量拼接在一起输入后续的全连接网络。
论文在计算Embedding向量的权重后，按权重排序，选择靠前的$K$个特征组进行模型训练和在线推理。而$K$值的设置，论文进行了多轮离线实验，评估不同$K$值下的模型表达能力（通过GAUC指标）和系统性能指标（通过QPS和RT），通过权衡，确认最终的$K$值。
COLD的工程实现如图38所示，其在线推理包含两步：第一步，对于每次广告请求，从用户特征表获取当前发起广告请求的用户的用户特征，从广告特征表获取召回返回的粗排候选广告集合中所有广告的广告特征，并进而计算用户和广告的交叉特征；第二步，将特征输入网络，先转化为Embedding向量，再拼接在一起输入全连接网络，前向传播后输出预估点击率。工程实现的优化包括并行化、列式计算和降低精度等。
![图38 COLD工程实现](https://cdn.nlark.com/yuque/0/2023/png/1332969/1698560227245-a81ec6c6-c689-45be-b617-0d3910d09a20.png#averageHue=%23e4e4e4&clientId=u6b870e71-e99a-4&from=paste&height=297&id=u98bf49b8&originHeight=650&originWidth=962&originalType=binary&ratio=2&rotation=0&showTitle=true&size=86216&status=done&style=none&taskId=u1d62ac00-9499-4721-b423-8087789f610&title=%E5%9B%BE38%20COLD%E5%B7%A5%E7%A8%8B%E5%AE%9E%E7%8E%B0&width=439 "图38 COLD工程实现")
并行化。由于需对多个候选广告预估点击率，而每个广告的计算相互独立，因此，可以按广告维度进行并行计算。
列式计算。对于多个候选广告的特征计算，论文不采用按广告维度的行式计算，而采用按特征组维度的列式计算，如图39所示，这样可以借助SIMD（Single Instruction Multiple Data）实现同时对多个数据执行指令以加速特征计算。
![图39 特征的行式计算和列式计算](https://cdn.nlark.com/yuque/0/2023/png/1332969/1698562470848-fae51ddf-ac1f-49e3-8e7a-d724a2144a91.png#averageHue=%23f5f4f1&clientId=u6b870e71-e99a-4&from=paste&height=291&id=u26af87b7&originHeight=808&originWidth=1550&originalType=binary&ratio=2&rotation=0&showTitle=true&size=170819&status=done&style=none&taskId=u36a0e077-f641-42d8-9d75-b3a8cdfe479&title=%E5%9B%BE39%20%E7%89%B9%E5%BE%81%E7%9A%84%E8%A1%8C%E5%BC%8F%E8%AE%A1%E7%AE%97%E5%92%8C%E5%88%97%E5%BC%8F%E8%AE%A1%E7%AE%97&width=559 "图39 特征的行式计算和列式计算")
降低精度。网络前向传播中的运算主要是矩阵相乘，而Float16相对Float32在矩阵相乘上有更高的性能，因此，可以使用Float16替代Float32。
## COPR-引入对齐模块，以Pairwise方式学习精排排序结果
虽然以前模型结构不断变化，但粗排均是Pointwise类型的排序方法，其存在以下不足：粗排模型训练样本使用的是精排进一步筛选出的广告的展现、点击日志，存在样本偏差问题；粗排模型虽然不断演进，但受限计算复杂度的约束，其预估的点击率、转化率难以对齐精排。因此，粗排模型的演进从Pointwise类型的排序方法——先预估每个广告的点击率、再计算$\text{eCPM}$进行排序，发展至Pairwise类型的排序方法——直接学习并推理精排的排序结果。[《COPR: Consistency-Oriented Pre-Ranking for Online Advertising》](https://arxiv.org/abs/2306.03516)由阿里妈妈于2023年发表，介绍了其在粗排阶段直接学习精排排序结果并进行推理的相关工作——COPR（Consistency-Oriented Pre-Ranking）。
首先介绍广告请求中的两种日志。每次广告请求，精排会对粗排返回的$M$个广告预估点击率、计算$\text{eCPM}$、并按$\text{eCPM}$进行排序，将排序结果写入排序日志（Ranking Log）。一次广告请求的排序日志如下所示：
$\mathbf{R}=[(ad_1,pCTR_1,bid_1),\dots,(ad_M,pCTR_M,bid_M)]$
其中，每个广告包含其广告特征、精排预估点击率和出价。
精排从排序结果中选取靠前的$N$个广告展示给用户。系统将广告展现和用户是否点击写入展现日志。一次广告请求的展现日志（Impression Log）如下所示：
$\mathbf{I}=[(ad_1,y_1),\dots,(ad_N,y_N)]$
其中，每个广告包含其广告特征和是否被点击的标记$y$。
![图40 COPR网络结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1698140505782-0eb8172a-379b-4636-a5e2-6fd1ff615a1a.png#averageHue=%23eeedeb&clientId=u3bc052c6-74df-4&from=paste&height=199&id=FWFRo&originHeight=446&originWidth=1384&originalType=binary&ratio=2&rotation=0&showTitle=true&size=92815&status=done&style=none&taskId=uc0a1ccee-865c-48c9-ad22-bc79862bb3a&title=%E5%9B%BE40%20COPR%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=618 "图40 COPR网络结构")
COPR网络结构如图40所示，其将CLOD作为基线模型，并在COLD的基础上进行升级，增加排序对齐模块。
对于排序对齐，COPR并没有直接将粗排和精排排序结果完全对齐，而是设计了基于块的采样（Chunk-Based Sampling），如图41所示。论文将每条排序日志平均切分为$D$个块（Chunk），每个块中广告个数为$K=\frac{M}{D}$，对每个块，从其$K$个广告中抽样一个广告，如此共抽样$D$个广告，构成新的精排排序结果：
$\mathbf{R}_{chunk}=[(ad_{s_d},pCTR_{s_d},bid_{s_d},D-d)]_{d=1}^D$
其中，$D-d$表示排序优先级，值越大，优先级越高。COPR即基于上述新的排序结果进行学习，也就是说在块的粒度实现粗排和精排排序结果对齐。$K$值越小，粗排和精排排序结果一致性越高，相应要求粗排模型表达能力越高，但其计算复杂度也越高。若$K=1$，则粗排和精排排序结果完全一致。论文将$K$设置为10。
![图41 基于块的采样](https://cdn.nlark.com/yuque/0/2023/png/1332969/1698213888044-a1d17399-aea3-4d40-8c7c-c1269f510cb9.png#averageHue=%23ecebe9&clientId=u1a74dbcc-64ff-4&from=ui&height=205&id=dAqiX&originHeight=308&originWidth=736&originalType=binary&ratio=2&rotation=0&showTitle=true&size=27709&status=done&style=none&taskId=ua6a77bdf-5688-4061-8a11-a0235c09e30&title=%E5%9B%BE41%20%E5%9F%BA%E4%BA%8E%E5%9D%97%E7%9A%84%E9%87%87%E6%A0%B7&width=489 "图41 基于块的采样")
# 反馈延迟
反馈延迟建模是效果广告领域的一个常见问题。在效果广告领域中，随着广告主愈加关注后链路转化效果，其偏向采用oCPC方式进行广告投放，即在转化（如下载、安装、激活、付费、加购、下单等）上进行出价，由广告平台在广告请求到来后，用模型实时预估点击率pCTR和转化率pCVR，并根据转化出价，计算eCPM：
$\text{eCPM}=\text{pCTR}\times\text{pCVR}\times\text{tCPA}$
最后使用eCPM进行排序和计价，选取靠前的广告进行展示。
而模型预估的准确性依赖于是否准确收集用户在广告展现后的反馈（是否点击和转化）作为样本标记进行模型训练，特别是实时模型更新更依赖样本的时效性，而转化相对于展现、点击一般都会延迟发生，比如用户点击游戏广告下载后，可能会延迟数日才会在游戏中进行付费。假设将付费作为转化目标由模型预估转化率，若在模型训练时，付费还未发生，则由展现、点击日志构成的样本会被标记为负样本，若最终付费在广告平台归因最长窗口期内发生并回传，则样本又会被标记为正样本，这就会导致样本在前后模型训练时的标记不同，并且训练时样本标记的分布和真实分布相比是有偏的——错误地将正样本标记为负样本（Fake Negative，FN），如果模型直接基于有偏的样本集进行训练，最终模型预估的转化率也是有偏的（偏低）。
令真实的训练样本集，IP（Immediate Positive）表示立即发生转化的正样本，DP（Delay Positive）表示延迟发生转化的正样本，其等价于FN（Fake Negative），即假负样本，RN（Real Negative）表示未发生转化的真负样本。
## 重要性采样
对于上述反馈延迟问题，业界已经有比较多的理论研究和业务实践，其中一个思路是保持模型不变，但针对反馈延迟导致的样本标记分布偏差，通过在损失函数中进行正负样本的加权来纠偏。
令$x$表示样本，$y\in\{0,1\}$表示样本最终是否发生转化，$f_\theta(y|x)$表示预估转化率的模型，则训练模型常用的交叉熵损失函数可表示为：
$\mathcal{L}(\theta)=-\mathbb{E}_p[\log{f_\theta(y|x)}]=-\sum_{x,y}{p(x,y)\log{f_\theta(y|x)}}$
因为模型训练样本集存在FN问题，所以训练样本集并不满足真实概率分布$p(x,y)$。令样本集的概率分布为$b(x,y)$，与真实概率分布存在偏差，因此，调整损失函数计算方式，由计算真实概率分布下的期望等价修改为计算偏差概率分布下的期望：
$\mathcal{L(\theta)}=-\mathbb{E}_p[\log{f_\theta(y|x)}]=-\mathbb{E}_b[\frac{p(x,y)}{b(x,y)}\log{f_\theta(y|x)}]$
令$w(x,y)=\frac{p(x,y)}{b(x,y)}$表示重要性权重，上述期望基于训练样本集计算的公式如下：
$\mathcal{L}(\theta)=-\frac{1}{N}\sum_n{w(x_n,y_n)\log{f_\theta(y_n|x_n)}}$
通过上述公式即可以基于有偏训练样本集计算损失函数，并通过梯度下降更新模型参数最小化损失函数。这种基于有偏概率分布预估真实概率分布的方法被称为重要性采样（Importance Sampling）。重要性采样并不需要对原有点击率预估模型结构进行修改，仅需要调节损失函数中正负样本权重，其核心问题即如何计算权重$w(x,y)$。
![图42 重要性采样的几种方案](https://cdn.nlark.com/yuque/0/2024/png/1332969/1715754256459-b8152d4c-f11a-45a1-812f-2f6f0438a72c.png#averageHue=%23858281&clientId=ud8dbd77c-e869-4&from=paste&height=221&id=u00d2b4e4&originHeight=442&originWidth=1042&originalType=binary&ratio=2&rotation=0&showTitle=true&size=33040&status=done&style=none&taskId=u8773a6bb-e9d4-4bd2-b6c1-f0d64a2d857&title=%E5%9B%BE42%20%E9%87%8D%E8%A6%81%E6%80%A7%E9%87%87%E6%A0%B7%E7%9A%84%E5%87%A0%E7%A7%8D%E6%96%B9%E6%A1%88&width=521 "图42 重要性采样的几种方案")
### FNW/FNC
较早采用重要性采样的比较经典的一篇论文是Twitter于2019年发表的《Addressing Delayed Feedback for Continuous Training with Neural Networks in CTR prediction》。Twitter在这篇论文中主要解决其视频广告的延迟点击问题，因此，在本节内容中，样本为展现日志，样本标记为是否点击。
对于点击延迟，论文在处理样本时并不会等待其相应的点击发生，而是直接将样本标记为负样本，待点击发生后，再将原样本复制成一条新样本，并将新样本标记为正样本。
论文通过推导得出，在损失函数中，当训练样本集有偏时，对于正样本，用$(1+p(y=1|x))$进行加权，对于负样本，用$(1-p(y=1|x))\cdot(1+p(y=1|x))$进行加权。由于$p(y=1|x)$无法提前得知，所以用模型预估值$f_\theta(y=1|x)$直接替换真实概率分布$p(y=1|x)$。
### ES-DFM
针对反馈延迟问题，Twitter提出的FNW先将所有样本直接标记为负样本用于训练，等点击发生后，再复制原样本、构造新的正样本用于训练，并基于重要性采样，通过调节损失函数中正负样本权重，解决反馈延迟问题。这种方法虽然样本时效性较高，但会导致所有正样本初始时均会被错误标记为负样本。一种权衡的方法是等待一段时间（Elapsed Time），若点击发生，则将样本标记为正样本，否则仍标记为负样本，等待虽不能解决FN问题（点击可能在等待后发生），但在降低样本时效性的代价下，能够带来样本准确性的提升。基于这个思路，阿里妈妈于2021年发表了论文《Capturing Delayed Feedback in Conversion Rate Prediction via Elapsed-Time Sampling》，提出了ES-DFM（Elapsed-Time Sampling Delayed Feedback Model）算法。由于后续内容主要解决转化延迟问题，因此样本为展现、点击日志，样本标记为是否转化。
论文通过推导，损失函数最终可写成：
$\mathcal{L}_{iw}^n=-\sum_{(x_i,y_i)\in\tilde{D}}^n{y_i[1+p_{dp}(x_i)]\log(f_\theta(x_i))+(1-y_i)[1+p_{dp}(x_i)]p_{rn}(x_i)\log(1-f_\theta(x_i))}$
其中，$\tilde{D}$表示训练样本集，其满足概率分布$q(x,y)$。至此，论文将正负样本权重的计算转化为计算$p_{dp}$和$p_{rp}$。论文分别使用分类器$f_{dp}$和$f_{rn}$预估$p_{dp}$和$p_{rn}$，这两个分类器共用一个模型进行联合训练，模型结构和转化率预估模型相同。
对于如何构造$f_{dp}$和$f_{rn}$的训练数据，论文使用30天前的历史样本数据，这些样本数据已无反馈延迟，可以对其准确标记正负样本以及当时是否延迟。对于$f_{dp}$，其正样本为转化延迟正样本，其负样本为历史样本数据中除转化延迟正样本以外的其他样本，对于$f_{rn}$，其正样本为转化负样本，其负样本为转化延迟正样本。
论文通过等待转化发生并归因以提升样本准确性。对于等待时间的长短，论文指出电商广告场景下，不同类型商品的转化延迟时间不同（比如价格高的商品转化延迟大），因此相应的等待时间也应不同，但论文对此进行了简化，将等待时间固定为常量，并通过实验验证其取值为1小时较合适。
### DEFER
FNW和ES-DFM在处理样本时，均只对Fake Negative样本进行处理，在转化发生并归因后，复制原样本，构造新的正样本，如图43左侧所示。阿里妈妈于2021年发表了论文《Real Negatives Matter: Continuous Training with Real Negatives for Delayed Feedback Modeling》，提出了DEFER（DElayed FEedback modeling with Real negatives）算法，如图43右侧所示，除复制Fake Negative样本外，也复制Real Negative样本和Positive样本，从而保证训练样本集特征概率分布$q(x)$和真实样本集特征概率分布$p(x)$一致。
![图43 DEFER和之前算法样本复制策略的不同](https://cdn.nlark.com/yuque/0/2023/png/1332969/1697456953540-880c65e7-5b57-48b1-a6e7-1b7195644cc9.png#averageHue=%23f1f0f0&clientId=uad3bb9e5-dbb2-4&from=paste&height=201&id=ZshSZ&originHeight=402&originWidth=1616&originalType=binary&ratio=2&rotation=0&showTitle=true&size=127088&status=done&style=none&taskId=u0b0fc535-6316-40ac-9d77-d03336b7e50&title=%E5%9B%BE43%20DEFER%E5%92%8C%E4%B9%8B%E5%89%8D%E7%AE%97%E6%B3%95%E6%A0%B7%E6%9C%AC%E5%A4%8D%E5%88%B6%E7%AD%96%E7%95%A5%E7%9A%84%E4%B8%8D%E5%90%8C&width=808 "图43 DEFER和之前算法样本复制策略的不同")
通过复制，训练样本集除包含真实样本集中的Positive、Fake Negative和Real Negative外，还包含复制样本集中的真正样本和真负样本，真实样本集和复制样本集除部分样本的标记不同外，其余均一致。
### DEFUSE
前述论文均是基于重要性采样，并不断优化权重计算方式，但均存在一个问题，即Fake Negative样本在训练时被错误地标记。阿里妈妈于2022年发表了论文《Asymptotically Unbiased Estimation for Delayed Feedback Modeling via Label Correction》，提出了DEFUSE（DElayed Feedback modeling with UnbiaSed Estimation）算法，采用两阶段优化来解决上述问题，首先预测Fake Negative样本的概率，然后再进行重要性采样。
# 多触点归因
广告主一般会在多个渠道（媒体或广告平台）分配预算，进行多渠道的广告投放，比如某个游戏行业广告主为推广新游戏，可能会在搜索、信息流、开屏、直播等多种形式的渠道流量上投放游戏广告，吸引用户下载、安装、激活并付费，而用户可能会在上述渠道流量上先后浏览到该游戏广告，直至最后发生转化行为。广告对用户的曝光被称为触点，某个广告对某个用户在多个渠道流量的多次曝光构成触点序列，被称为转化路径。广告主一般会收集其在多个渠道上投放广告的曝光、点击和转化明细数据，从而从全局角度构建全渠道下的转化路径集合，并借助算法分析转化路径各个触点对最终转化的归因权重，这种分析被称为多触点归因（Multi Touch Attribution），通过这种分析，广告主可以量化评估各个渠道对转化的重要性，从而调整在各个渠道上的预算分配，实现全局角度的投放策略优化。
多触点归因算法主要分为基于规则的算法和基于数据的算法两大类。
## 基于规则的归因算法
基于规则的归因算法包括但不限于：

- 末次触点归因（Last-Touch），将转化仅归因到最后的一次触点上；
- 首次触点归因（First-Touch），将转化仅归因到最早的一次触点上；
- 线性归因（Linear），将转化归因到每次触点上，且每次触点的权重相等；
- 时间衰减归因（Time Decay），将转化归因到每次触点上，且每次触点的权重随时间进行衰减，越早的触点时间衰减越多，权重越小。
## 基于数据并基于深度学习的归因算法
基于数据的算法最早于2011年在论文《Data-driven Multi-touch Attribution Models》中被提出，其中使用Logistic回归模型进行各触点归因权重分析，而随着深度学习的发展，近几年来不少论文探索基于深度学习的多触点归因算法。
### DNAMTA-转化为转化率预估问题，通过RNN对触点序列建模，并由各触点隐状态计算其归因权重
2018年发表的论文《Deep Neural Net with Attention for Multi-channel Multi-touch Attribution》提出了DNAMTA算法。论文指出其首次在业界将深度学习应用于多触点归因中。
令$P_i=\{x_1,x_2,\dots,x_T\}$表示某个用户的某个转化路径，其中$x_t$表示转化路径中的某个触点，$x_t\in\mathbb{M}$，$\mathbb{M}$表示所有触点类型的集合。使用序列$\{U_1,U_2,\dots,U_T\}$表示$P_i$中每个触点的发生时间，其中$U_t$表示$P_i$中触点$x_t$的发生时间。转化路径除了触点序列这一信息外，还包括一些不随触点序列变化的静态信息，比如用户属性（年龄、性别、职业等），使用$C_i$表示$P_i$的这些静态信息。使用$Y_i$表示$P_i$最终是否发生转化（1表示发生转化，0表示未发生转化）。使用$a_t$表示$P_i$中触点$x_t$的归因权重，且$P_i$中所有触点的归因权重之和为1，即$\sum_{t=1}^T{a_t}=1$。基于上述问题描述，将深度学习应用于多触点归因的目标即对于发生转化的转化路径$P_i$，求解其中每个触点的归因权重$a_t$。
DNAMTA算法将求解$P_i$其中每个触点的归因权重$a_t$，转化为求解$P_i$的转化率$P(Y_i|P_i,C_i)$，并通过模型预测$P_i$的转化率$f(C_i,\{x_t\}_{1:T})$，每个触点的归因权重$a_t$作为模型中间层的输出。模型首先使用LSTM挖掘触点之间的关系，将每个触点转化为相应的隐状态，然后基于注意力机制计算每个触点隐状态的权重。
![图44 DNAMTA网络结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1698844097178-2e180d44-443e-4723-a5c1-a36787c4f6cd.png#averageHue=%23f8f7f7&clientId=u247dd7fb-e5a7-4&from=paste&height=302&id=yCwdv&originHeight=604&originWidth=1140&originalType=binary&ratio=2&rotation=0&showTitle=true&size=78104&status=done&style=none&taskId=uc391e93a-8f3a-4701-9536-8d8fc8c23cd&title=%E5%9B%BE44%20DNAMTA%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=570 "图44 DNAMTA网络结构")
### DARNN-在DNAMTA基础上，增加点击序列隐状态
2018年发表的论文《Learning Multi-touch Conversion Attribution with Dual-attention Mechanisms for Online Advertising》提出了DARNN算法。和DNAMTA相同的是，DARNN也将目标转化为转化率预测，使用LSTM挖掘触点序列的信息，并引入注意力机制，学习各触点的归因权重。但和DNAMTA不同的是，DARNN的触点特征还包含用户、广告和渠道特征，还通过序列到序列框架，挖掘展现和点击之间的关系，并使用LSTM掘点击序列的信息，从而考虑用户转化前反馈对最终转化的影响，注意力机制不仅作用在触点序列的隐状态，还作用在点击序列的隐状态上，将两者组合用于归因权重计算和转化率预测，从而实现用户个性化的多触点归因分析，这也是论文标题中“Dual-attention”的含义。
![图45 DARNN网络结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1698982172223-7470ce91-facb-41e0-bbe3-d02c2633ed60.png#averageHue=%23f8f4f3&clientId=u307f8546-672a-4&from=paste&height=498&id=n1ECr&originHeight=996&originWidth=1138&originalType=binary&ratio=2&rotation=0&showTitle=true&size=112213&status=done&style=none&taskId=ubabb128e-31f1-4e18-83bb-53721f7df4e&title=%E5%9B%BE45%20DARNN%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=569 "图45 DARNN网络结构")
除了设计DARNN算法，论文还设计了触点归因权重的评估方法。广告主会在多个渠道上投放广告，比如信息流推荐广告、搜索广告、应用开屏广告等。广告主会对每个渠道设置各自的广告预算，并统计每个渠道上因转化带来的收入，进一步计算每个渠道的ROI，衡量各个渠道的投放效果。论文使用下述公式计算某个渠道$c_k$的ROI：
$\text{ROI}_{c_k}=\frac{\sum_{\forall y_i=1}\text{Attr}(c_k|y_i)V(y_i)}{\text{Money spend on channel }c_k}$
其中，分母即广告主在渠道$c_k$上的广告投放费用，分子是对每个发生转化的转化路径，计算渠道$c_k$的归因权重，基于这个权重，将这个转化带来的广告主收入的一部分分配给该渠道，并求和所有转化分配给该渠道的收入作为该渠道带来的收入，而渠道的归因权重可由下式计算：
$\text{Attr}(c_k|y_i)=\sum_{j=1}^{m_i}{\text{Attr}_j\cdot 1(c_j=c_k)}$
即将转化路径中属于该渠道的触点的归因权重求和。
将各个渠道的ROI作为其权重，从总预算中分配该渠道的预算：
$b_k=\frac{\text{ROI}_{c_k}}{\sum_{v=1}^K{\text{ROI}_{c_v}}}\times B$
ROI越高的渠道，预算越多。然后，按时间序回放广告主在全部渠道上的广告投放历史数据。广告投放历史数据为转化路径集合，而每个转化路径为触点序列，每个触点包含广告渠道、广告费用、是否发生转化等信息。初始化转化路径黑名单为空。回放广告投放历史数据时，若当前触点所对应的转化路径不在黑名单中，则判断触点所对应的渠道是否超预算（渠道预算余额是否足够支付触点广告费用），若超出预算，则将该转化路径加入黑名单，若未超出预算，则按广告费用扣减渠道预算，并累加广告投放总费用$O$和转化总数$Y$（当前触点发生转化才累加转化总数）。回放完成后，根据广告投放总费用$O$和转化总数$Y$衡量触点归因权重是否合理。
### JDMTA-转化为转化率预估问题，并通过反事实分析计算各触点的夏普利值作为其归因权重
2019年京东发表的论文 《Causally Driven Incremental Multi Touch Attribution Using a Recurrent Neural Network》提出了JDMTA算法。在京东电商场景下，不同品牌会在多个广告位持续进行商品广告投放，而用户持续多天在多个广告位上浏览某品牌的商品广告后，可能会对该品牌的商品下单，形成转化。JDMTA算法的目标是量化分析某品牌某天在某广告位上对用户的广告曝光，对最终下单的归因权重。JDMTA算法架构如图46所示，其整体分为两部分：

- 第一部分（图4中蓝色框）和DNAMTA算法类似，JDMTA算法也将目标转化为下单率预测，使用LSTM挖掘广告曝光序列的信息，并另外通过一个全连接网络对用户特征进行建模，最后结合广告曝光序列隐状态和用户特征隐向量预测下单率；
- 第二部分（图4中绿色框）量化分析归因权重，和上述两个算法通过注意力机制、使用模型中间层输出作为归因权重不同，JDMTA算法引入合作博弈论（Cooperative Game Theory），基于广告曝光应会提升下单率这一假设，通过反事实（Counterfactual）分析，使用第一部分完成训练的下单率预测模型，预测某品牌某天在某广告位上对用户的广告曝光对下单率的边际期望增益（Marginal Expected Incremental Benefit），也就是夏普利值（Shapley Value），作为其归因权重。

![图46 JDMTA算法架构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1698844832318-e650a163-28a2-4c37-b807-aa2137555f45.png#averageHue=%23f7f7f2&clientId=u247dd7fb-e5a7-4&from=paste&height=361&id=uc16664c2&originHeight=1080&originWidth=1158&originalType=binary&ratio=2&rotation=0&showTitle=true&size=198213&status=done&style=none&taskId=u121491b0-f349-49c0-bc28-bf93ebc97aa&title=%E5%9B%BE46%20JDMTA%E7%AE%97%E6%B3%95%E6%9E%B6%E6%9E%84&width=387 "图46 JDMTA算法架构")
### CAMTA-通过因果循环网络训练消除用户动态历史偏差，解决Confounding问题
2020年的论文《CAMTA: Causal Attention Model for Multi-touch Attribution》提出了CAMTA，其基本思路仍是将多触点归因问题变换为转化率预估问题，采用循环神经网络对触点序列进行序列建模，通过模型预估转化率，在得到转化率预估模型的基础上，再进一步进行归因权重的计算，从而实现用户个性化的多触点归因，其创新之处是借鉴了CRN的思想，将常规循环神经网络升级为因果循环神经网络（Causal Recurrent Network），解决广告场景中多触点归因的混杂问题。
论文使用图47说明广告场景中多触点归因的混杂问题，$\mathbf{T}_1$和$\mathbf{T}_2$时刻用户的上下文$\mathbf{x}_1$、$\mathbf{x}_2$作为混杂因子，既影响这两个时刻用户是否发生转化，也影响这两个时刻的渠道选择，从而导致渠道选择并不是随机的，即选择偏差（Selection Bias）问题。而理想的无偏情况下，每个触点、每个渠道的用户分布应该是随机。
![图47 广告场景中多触点归因的混杂问题](https://cdn.nlark.com/yuque/0/2024/png/1332969/1704188868051-13d464d4-573f-4012-a808-5ea882980f11.png#averageHue=%23f7f6f6&clientId=uce2b9787-47ae-4&from=paste&height=455&id=u7786760d&originHeight=1432&originWidth=1158&originalType=binary&ratio=2&rotation=0&showTitle=true&size=894111&status=done&style=none&taskId=u7ff3fdc8-b135-4765-91b5-f834080a8e5&title=%E5%9B%BE47%20%E5%B9%BF%E5%91%8A%E5%9C%BA%E6%99%AF%E4%B8%AD%E5%A4%9A%E8%A7%A6%E7%82%B9%E5%BD%92%E5%9B%A0%E7%9A%84%E6%B7%B7%E6%9D%82%E9%97%AE%E9%A2%98&width=368 "图47 广告场景中多触点归因的混杂问题")
![图48 CAMTA整体网络结构](https://cdn.nlark.com/yuque/0/2023/png/1332969/1698844502449-75d2bee9-a8b7-442f-be0e-357e9389f407.png#averageHue=%23a2a4a2&clientId=u247dd7fb-e5a7-4&from=paste&height=415&id=tIWdl&originHeight=1226&originWidth=1524&originalType=binary&ratio=2&rotation=0&showTitle=true&size=476390&status=done&style=none&taskId=ua1005344-9143-46b2-9e00-a4d9fcf566e&title=%E5%9B%BE48%20CAMTA%E6%95%B4%E4%BD%93%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=516 "图48 CAMTA整体网络结构")
CAMTA的整体网络结构如图48所示，其包含三部分。
第一部分是因果循环神经网络，由粉色虚线标出，其使用循环神经网络对用户历史的触点序列进行建模，得到每个触点的隐状态表征$\mathbf{s}_t^n$，进而通过域对抗训练，得到每个触点的无偏隐状态表征$\mathbf{r}_t^n$，最后再基于每个触点的无偏隐状态表征通过点击预估分类器预估每个触点的点击率。
第二部分是注意力机制，由蓝色虚线标出，其基于每个触点的预估点击率学习每个触点的点击隐状态表征$\mathbf{v}_t^n$，再计算$\mathbf{v}_t^n$和一个可训练的上下文向量$\mathbf{u}$的点积，并通过Softmax函数对点积进行归一化，作为各触点的归因权重。
第三部分是转化率预估，由灰色虚线标出，其对各触点的点击隐状态表征基于归因权重进行加权求和，得到用户历史的隐状态表征$\mathbf{h}^n$，最后再通过一层MLP和Sigmoid函数得到最终的转化率预估值。
### CausalMTA-进一步通过访问路径重加权消除用户静态属性偏差，解决Confounding问题
2021年的论文《CausalMTA: Eliminating the User Confounding Bias for Causal Multi-touch Attribution》提出了CausalMTA，其和CAMTA相比，将用户偏好这一混杂因子，进一步区分为不变的静态属性和变化的动态特征，对于静态属性，其使用变分循环自编码器作为渠道序列生成模型获取其无偏分布，然后基于无偏分布和逆概率加权方法对每个转化路径重加权，从而消除静态属性引起的选择偏差，而对于动态特征，其和CAMTA类似，也是借鉴CRN，通过循环神经网络和域对抗训练，生成用户历史的无偏表征，从而消除动态特征引起的选择偏差，得到无偏的转化率预估模型。最后，基于转化率预估模型，采用反事实分析计算各渠道的夏普利值作为归因权重，即对各渠道，使用转化率预估模型分别预估有无该渠道时的转化率，因引入该渠道带来的转化率提升即该渠道对转化的边际期望增益，也就是该渠道的夏普利值，被作为该渠道的归因权重。
用户偏好中的静态属性和动态特征如图49所示，静态属性包括用户年龄、性别、职业等不随触点序列变化的属性，动态特征即用户已浏览广告构成的触点序列。
![图49 多类别混杂因子导致的混杂问题](https://cdn.nlark.com/yuque/0/2024/png/1332969/1705156240785-d4df656e-fc02-46ed-90b8-7dda82c0d2ef.png#averageHue=%23f6f6f5&clientId=u3ab4bf98-2763-4&from=paste&height=451&id=u5c82a5e3&originHeight=1066&originWidth=1052&originalType=binary&ratio=2&rotation=0&showTitle=true&size=668436&status=done&style=none&taskId=u872d616b-aed3-4478-88d3-887ea9487e6&title=%E5%9B%BE49%20%E5%A4%9A%E7%B1%BB%E5%88%AB%E6%B7%B7%E6%9D%82%E5%9B%A0%E5%AD%90%E5%AF%BC%E8%87%B4%E7%9A%84%E6%B7%B7%E6%9D%82%E9%97%AE%E9%A2%98&width=445 "图49 多类别混杂因子导致的混杂问题")
![图50 CausalMTA整体解决方案](https://cdn.nlark.com/yuque/0/2023/png/1332969/1698813087880-0625f053-1c93-4b15-b51a-9f295d16e386.png#averageHue=%23edeceb&clientId=u3c564273-31f1-4&from=paste&height=294&id=LfAT1&originHeight=588&originWidth=1410&originalType=binary&ratio=2&rotation=0&showTitle=true&size=168589&status=done&style=none&taskId=uc8acd8b2-051d-4941-b30e-08cfc08e18a&title=%E5%9B%BE50%20CausalMTA%E6%95%B4%E4%BD%93%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88&width=705 "图50 CausalMTA整体解决方案")
CausalMTA整体解决方案模型如图50所，其包含三部分。第一部为转化路径重加权（Journey Reweighting），其对于静态属性，使用变分循环自编码器作为渠道序列生成模型获取其无偏分布，然后基于无偏分布和逆概率加权方法对每个转化路径重加权，从而消除静态属性引起的选择偏差。第二部分为因果转化率预估（Causal Conversion Prediction），其对于动态特征，借鉴CRN，通过循环神经网络和域对抗训练，生成用户历史的无偏表征，从而消除动态特征引起的选择偏差，得到无偏的转化率预估模型。第三部分为归因权重计算（Attribution），其基于转化率预估模型，采用反事实分析计算各渠道的夏普利值作为归因权重，即对各渠道，使用转化率预估模型分别预估有无该渠道时的转化率，因引入该渠道带来的转化率提升即该渠道对转化的边际期望增益，也就是该渠道的夏普利值，被作为该渠道的归因权重。
# 多任务学习
多任务学习（Multi-Task Learning，MTL）是一种机器学习方法，其对多个不同类型的任务进行联合学习，相比于对多个任务分别单独建模进行学习，多任务学习可以通过共享模型参数减少整体参数量，并将模型挖掘的知识在多个任务之间进行迁移，提升整体性能。在计算广告领域，多任务学习的应用场景包括对广告展现样本点击率和转化率预估的联合学习等。
## MMoE-多个门控的混合专家网络
之前介绍的ESMM算法对点击率和转化率预估进行联合学习，其多任务学习的核心思想是共享Embedding层，而在Embedding层之上构建两个网络分别用于点击率和转化率预估。这类共享底层网络层的多任务学习算法被统称为“Share Bottom”算法，即多个任务共享底层的网络层，而在共享的网络层之上每个任务有各自的网络塔和输出，由于共享底层的网络层，因此每个任务挖掘的信息可以在底层网络层实现相互之间的迁移，而每个任务各自的网络塔则保留各自特有的信息。基于“Shared Bottom”的ESMM算法在点击率和转化率预估这两个相关性较强的任务上进行联合学习并取得较好的效果，而对于相关性较弱的多个任务如何进行联合学习。
Google于2018年在KDD上发表了论文《Modeling Task Relationships in Multi-task Learning with Multi-gate Mixture-of-Experts》，其中提出了MMoE算法用于多任务学习。和“Shared Bottom”算法相同的是，MMoE算法中每个任务也有各自的网络塔和输出，而和“Shared Bottom”算法不同的是，MMoE并没有简单采用共享的底层网络层，而是采用多个专家网络对各任务的知识进行挖掘和共享，并对每个任务设计其专有的门控网络，每个任务的门控网络对各专家网络的输出进行加权求和作为该任务专有网络塔的输入。论文在实验中构造了不同相关性的多任务，论证了对于不同相关性的多任务，MMoE相比“Shared Bottom”算法均取得更好的效果。
![图51 多任务学习数种实现方式的网络结构（Shared Bottom、OMoE、MMoE）](https://cdn.nlark.com/yuque/0/2024/png/1332969/1705568182430-128e6b0d-8e74-4320-b2ae-dc4c6aaa512a.png#averageHue=%23e2e8dc&clientId=uf2d28aa8-ad4c-4&from=paste&height=250&id=cWWdV&originHeight=562&originWidth=1504&originalType=binary&ratio=2&rotation=0&showTitle=true&size=445148&status=done&style=none&taskId=u56d65e28-c2df-40af-a413-48971db33fb&title=%E5%9B%BE51%20%E5%A4%9A%E4%BB%BB%E5%8A%A1%E5%AD%A6%E4%B9%A0%E6%95%B0%E7%A7%8D%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F%E7%9A%84%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84%EF%BC%88Shared%20Bottom%E3%80%81OMoE%E3%80%81MMoE%EF%BC%89&width=670 "图51 多任务学习数种实现方式的网络结构（Shared Bottom、OMoE、MMoE）")
## CGC & PLE-解决多任务学习的跷跷板问题
上一节介绍的MMoE采用多个专家网络对各任务的知识进行挖掘和共享，并对每个任务设计其专有的门控网络，每个任务的门控网络对各专家网络的输出进行加权求和作为该任务专有网络塔的输入。
腾讯于2020年在RecSys上发表了论文《Progressive Layered Extraction (PLE): A Novel Multi-Task Learning (MTL) Model for Personalized Recommendations》，其中提出了PLE算法用于多任务学习。论文提出多任务学习在推荐系统中已有较多应用，但受限于推荐系统不同任务之间复杂的相关性，原有的多任务学习方案存在负迁移和跷跷板问题，负迁移问题指多任务学习相比对各任务分别单独建模进行学习，所有任务的效果未提升或更差，跷跷板问题指多任务学习相比对各任务分别单独建模进行学习，有的任务的效果提升，有的任务的效果下降。为了解决上述问题，论文首先提出了CGC网络，其针对MMoE中仅通过门控网络隐式地控制各专家网络在任务专有和共享中比重的问题，显式地将专家网络划分为各任务专有的专家网络和所有任务共享的专家网络，并对于每个任务，使用和MMoE类似的门控网络将该任务专有的专家网络和共享的专家网络的输出表征进行加权求和，进而得到任务塔的输入。论文接着提出了PLE网络，其在CGC的基础上进行扩展，包含多层抽取网络，且每层抽取网络的结构和CGC网络类似，从而实现各任务专有信息和所有任务共享信息的逐层抽取和深层挖掘。论文最终通过在腾讯视频推荐系统和公开数据集上的实验，证明了PLE能够有效解决多任务学习中的负迁移和跷跷板问题，并在效果上取得SOTA。
### CGC-进一步将专家网络划分为不同类型的组
![图52 CGC的网络结构](https://cdn.nlark.com/yuque/0/2024/png/1332969/1709798144597-5b636e13-6c0c-4fe6-8ece-cadf3471aaef.png#averageHue=%23f3f1f1&clientId=u134fc435-1062-4&from=paste&height=382&id=plu5M&originHeight=764&originWidth=1006&originalType=binary&ratio=2&rotation=0&showTitle=true&size=448701&status=done&style=none&taskId=u5ca3e982-06b4-478a-ae11-f0cd6be13a2&title=%E5%9B%BE52%20CGC%E7%9A%84%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=503 "图52 CGC的网络结构")
CGC的网络结构如图52所示，其上层是每个任务各自专有的网络塔，下层是多个专家网络。CGC将专家网络划分为多组，每个任务专有一组专家网络负责挖掘该任务专有的知识，同时所有任务共享一组专家网络负责挖掘所有任务共享的知识。对于任务$k$，其通过该任务专有的门控网络合并该任务专有的专家网络组的输出和所有任务共享的专家网络组的输出。
### PLE-进一步构建多层抽取网络，对专有和共享信息逐层抽取和挖掘
![图53 PLE的网络结构](https://cdn.nlark.com/yuque/0/2024/png/1332969/1709798175195-5242fa6a-9375-468a-96c1-021a6c3f56d7.png#averageHue=%23f6f4f4&clientId=u134fc435-1062-4&from=paste&height=534&id=H2JEO&originHeight=1068&originWidth=1218&originalType=binary&ratio=2&rotation=0&showTitle=true&size=719498&status=done&style=none&taskId=u1210db20-1a96-47c2-9bfc-ce4f14a0b44&title=%E5%9B%BE53%20PLE%E7%9A%84%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84&width=609 "图53 PLE的网络结构")
PLE的网络结构如图53所示，其在CGC的基础上构建$N$层抽取网络（Extraction Network），除最后一层抽取网络的结构和CGC相同外，其他各层抽取网络的结构在CGC的基础上稍作改动。
其他各层抽取网络的结构和CGC相同的是，每个任务专有一组专家网络负责挖掘该任务专有的知识，同时所有任务共享一组专家网络负责挖掘所有任务共享的知识。对于第$j$层抽取网络中的任务$k$，其通过该任务专有的门控网络合并该任务专有的专家网络组的输出和所有任务共享的专家网络组的输出，得到当前层任务$k$的输出，并作为下一层任务$k$专有的专家网络组的输入。
其他各层抽取网络的结构和CGC不同的是，每层抽取网络还有一个从该层所有专家网络进一步挖掘共享知识的共享门控网络。对于第$j$层抽取网络中的共享门控网络，其合并该层所有专家网络（包括各任务专有的专家网络组和所有任务共享的专家网络组）的输出，得到当前层共享知识的输出，并作为下一层所有任务共享的专家网络组的输入。
最后一层抽取网络的结构和CGC相同，输出各任务的表征，作为各任务专有塔的输入，最后由各任务专有塔输出预估结果。
综上，PLE构建多层抽取网络，在CGC网络区分任务专有专家网络和共享专家网络、分别挖掘专有和共享知识的基础上，进一步通过共享门控网络逐层从所有专家网络中抽取共享知识，从而实现各任务专有信息和所有任务共享信息的逐层抽取和深层挖掘。
# 更多应用场景
深度学习在计算广告中还有很多应用场景和研究方向。例如，广告投放平台拥有广告展现和点击数据，而广告主拥有后链路转化数据，另外还拥有用户的其他特征数据，或者其在其他广告投放平台的投放数据，如果广告主能将上述这些数据也提供给当前的广告投放平台用于模型训练，则可以让模型挖掘到更多知识，提升预估的准确性，但是随着用户隐私和数据安全越来越被重视，应安全合规、隐私保护要求，越来越多的数据不能出域，为解决数据不能出域的问题，联邦学习方案被提出，其能够实现数据不出域前提下的模型联合训练，联邦学习的方式主要包括两种，一是横向联邦，即特征空间相同，样本空间不同，通过横向联邦实现增加样本的联合训练，二是纵向联邦，即特征空间不同，样本空间相同，通过纵向联邦实现增加特征的联合训练。
更多应用场景还包括但不局限于：

- 标签挖掘；
- 内容识别；
- 风险控制；
- 样式优选。

后续笔者会持续更新增加上述场景的介绍。
# 更多介绍
关于深度学习在计算广告中应用的详细介绍，可以浏览笔者梳理的相关论文阅读笔记：

- [《广告召回论文阅读笔记（1）-双塔模型》](https://zhuanlan.zhihu.com/p/673883972)
- [《广告召回论文阅读笔记（2）-从TDM到二向箔》](https://zhuanlan.zhihu.com/p/675418752)
- [《点击率预估论文阅读笔记》](https://zhuanlan.zhihu.com/p/667661892)
- [《多任务学习、多场景学习相关论文阅读笔记（1）》](https://zhuanlan.zhihu.com/p/695613940)
- [《多任务学习、多场景学习相关论文阅读笔记（2）》](https://zhuanlan.zhihu.com/p/696644357)
- [《基于深度学习的广告拍卖机制论文阅读笔记（1）》](https://zhuanlan.zhihu.com/p/659320621)
- [《基于深度学习的广告拍卖机制论文阅读笔记（2）》](https://zhuanlan.zhihu.com/p/659547565)
- [《基于强化学习的自动出价论文阅读笔记（1）》](https://zhuanlan.zhihu.com/p/682009207)
- [《基于强化学习的自动出价论文阅读笔记（2）》](https://zhuanlan.zhihu.com/p/682415692)
- [《基于强化学习的自动出价论文阅读笔记（3）》](https://zhuanlan.zhihu.com/p/684052241)
- [《广告粗排相关论文阅读笔记》](https://zhuanlan.zhihu.com/p/663955557)
- [《反馈延迟建模论文阅读笔记》](https://zhuanlan.zhihu.com/p/662621218)
- [《基于深度学习的多触点归因论文阅读笔记》](https://zhuanlan.zhihu.com/p/665186606)
- [《基于深度学习的多触点归因论文阅读笔记（2）》](https://zhuanlan.zhihu.com/p/677754958)

# 相关论文
笔者也将所阅读论文原文汇总至GitHub，地址是：
