---
layout: post
title: 数据分析和挖掘方法学习笔记
description: 调研和总结了数据分析和挖掘方法的一些基本概念
category: 百科
tags: [数据挖掘,数据分析]
---

## 数据挖掘基本介绍

数据挖掘是从大量的、不完全的、有噪声的、模糊的、随机的数据集中识别有效的、新颖的、潜在有用的，以及最终可理解的模式的过程。它是一门涉及面很广的交叉学科，包括机器学习、数理统计、神经网络、数据库、模式识别、粗糙集、模糊数学等相关技术。

<!--more-->

### 数据挖掘背景

上世纪九十年代．随着数据库系统的广泛应用和网络技术的高速发展，数据库技术也进入一个全新的阶段，即从过去仅管理一些简单数据发展到管理由各种计算机所产生的图形、图像、音频、视频、电子档案、Web页面等多种类型的复 杂数据，并且数据量也越来越大。在给我们提供丰富信息的同时，也体现出明显的海量信息特征。信息爆炸时代．海量信息给人们带来许多负面影响，最主要的就是有效信息难以提炼。过多无用的信息必然会产生信息距离(the Distance of Information-state Transition，信息状态转移距离，是对一个事物信息状态转移所遇到障碍的测度。简称DIST或DIT)和有用知识的丢失。这也就是约翰•内斯伯特(John Nalsbert)称为的“信息丰富而知识贫乏”窘境。因此，人们迫切希望能对海量数据进行深入分析，发现并提取隐藏在其中的信息．以更好地利用这些数据。但仅以数据库系统的录入、查询、统计等功能，无法发现数据中存在的关系和规则，无法根据现有的数据预测未来的发展趋势。更缺乏挖掘数据背后隐藏知识的手段。正是在这样的条件下，数据挖掘技术应运而生。

数据清理过程有两种，一种是有监督，有监督过程是在领域专家的指导下，分析收集的数据，去除明显错误的噪音数据和重复记录，填补缺值数据，另一种是无监督，无监督过程是用样本数据训练算法，使其获得一定的经验，并在以后的处理过程中自动采用这些经验完成数据清理工作。

### 数据挖掘步骤

如果计划实施数据挖掘，它需要多个步骤，主要包括：定义商业问题、建立数据挖掘模型、分析数据、准备数据、建立模型、评价模型、实施。

1. 定义商业问题。在开始知识发现之前最先的同时也是最重要的要求就是了解数据和业务问题。必须要对目标有一个清晰明确的定义，即决定到底想干什么。比如想提高电子信箱的利用率时，想做的可能是“提高用户使用率”，也可能是“提高一次用户使用的价值”，要解决这两个问题而建立的模型几乎是完全不同的，必须做出决定。
2. 建立数据挖掘库。建立数据挖掘库包括以下几个步骤：数据收集、数据描述和选择、数据质量评估和数据清理、合并与整合、构建元数据、加载数据挖掘库、维护数据挖掘库。 
3. 分析数据。分析的目的是找到对预测输出影响最大的数据字段，和决定是否需要定义导出字段。如果数据集包含成百上千的字段，那么浏览分析这些数据将是一件非常耗时和累人的事情，这时需要选择一个具有好的界面和功能强大的工具软件来协助你完成这些事情。 
4. 准备数据。这是建立模型之前的最后一步数据准备工作。可以把此步骤分为4个部分：选择变量、选择记录、创建新变量、转换变量。
5. 建立模型。建立模型是一个反复的过程。需要仔细考察不同的模型以判断哪个模型对商业问题最有用。先用一部分数据建立模型，然后再用剩下的数据来测试和验证得到的模型。有时还有第三个数据集，称为验证集，因为测试集可能受模型的特性的影响，这时需要一个独立的数据集来验证模型的准确性。训练和测试数据挖掘模型需要把数据至少分成两个部分：一个用于模型训练，另一个用于模型测试。 
6. 评价和解释。模型建立好之后，必须评价得到结果、解释模型的价值。从测试集中得到的准确率只对用于建立模型的数据有意义。在实际应用中，需要进一步了解错误的类型和由此带来的相关费用的多少。经验证有效的模型并不一定是正确的模型。造成这一点的直接原因就是模型建立中隐含的各种假定。因此直接在现实世界中测试模型很重要。先在小范围内应用，取得测试数据，觉得满意之后再向大范围推广。 
7. 实施。模型建立并经验证之后，可以有两种主要的使用方法。第一种是提供给分析人员做参考；另一种是把此模型应用到不同的数据集上。 

因为事物在不断发展变化，很可能过一段时间之后，模型就不再起作用。销售人员都知道，人们的购买方式随着社会的发展而变化。因此随着使用时间的增加，要不断的对模型做重新测试，有时甚者需要重新建立模型。

数据挖掘可以在任何类型的信息存储上进行，包括关系数据库、数据仓库、事务数据库、先进的数据库系统、展平的文件、WWW（万维网）。数据挖掘的主要任务是描述性的数据挖掘和预测性数据挖掘，下面我们介绍几个在数据挖掘任务中用到的关键技术。

## 数据预处理

现实世界中数据大体上都是不完整，不一致的脏数据，比如无法直接进行数据挖掘，挖掘结果差强人意。为了提高数据挖掘的质量产生了数据预处理技术。 数据预处理有多种方法：数据清理，数据集成，数据变换，数据归约等。这些数据处理技术在数据挖掘之前使用，大大提高了数据挖掘模式的质量，降低实际挖掘所需要的时间。预处理在数据挖掘中占有很重要的位置，如图1所示。

![preprocesing](/images/DataMining/preprocesing.jpg)

<center>图1 预处理在知识发现中所占分量</center>

### 数据清理

数据清理要去除源数据集中的噪声和无关数据，处理遗漏数据和清洗脏数据，去除空白数据域和知识背景上的白噪声，考虑时间顺序和数据变化等（主要包括重复数据处理和缺值数据处理），此外，还要完成一些数据类型的转换。

数据清理有两类基本的方法：
1. 缺失数据处理： 目前最常用的方法是使用最可能的值填充缺失值，比如可以用回归、贝叶斯形式化方法工具或判定树归纳等确定缺失值。这类方法依靠现有的数据信息来推测缺失值，使缺失值有更大的机会保持与其他属性之间的联系。还有其他一些方法来处理缺失值，如用一个全局常量替换缺失值、使用属性的平均值填充缺失值或将所有元组按某些属性分类，然后用同一类中属性的平均值填充缺失值。如果缺失值很多，这些方法可能误导挖掘结果。如果缺失值很少，可以忽略缺失数据。
2. 噪声数据处理： 噪声是一个测量变量中的随机错误或偏差，包括错误的值或偏离期望的孤立点值。目前最广泛的是应用数据平滑技术处理，具体包括：①分箱技术，将存储的值分布到一些箱中，用箱中的数据值来局部平滑存储数据的值。具体可以采用按箱平均值平滑、按箱中值平滑和按箱边界平滑；②回归方法，可以找到恰当的回归函数来平滑数据。线性回归要找出适合两个变量的“最佳”直线，使得一个变量能预测另一个。多线性回归涉及多个变量，数据要适合一个多维面；③计算机检查和人工检查结合方法，可以通过计算机将被判定数据与已知的正常值比较，将差异程度大于某个阈值的模式输出到一个表中，然后人工审核表中的模式，识别出孤立点；④聚类技术，将类似的值组织成群或“聚类”，落在聚类集合之外的值被视为孤立点。孤立点可能是垃圾数据，也可能为我们提供重要信息。对于确认的孤立点垃圾数据将从数据库中予以清除。

### 数据集成

数据集成就是将多个数据源中的数据合并存放在一个同一的数据存储（如数据仓库、数据库等）的一种技术和过程，数据源可以是多个数据库、数据立方体或一般的数据文件。数据集成涉及3个问题：

1. 模式集成: 涉及实体识别，即如何将不同信息源中的实体匹配来进行模式集成。通常借助于数据库或数据仓库的元数据进行模式识别；
2. 冗余数据集成: 在数据集成中往往导致数据冗余，如同一属性多次出现、同一属性命名不一致等。对于属性间冗余，可以先采用相关性分析检测，然后删除；
3. 数据值冲突的检测与处理: 由于表示、比例、编码等的不同，现实世界中的同一实体，在不同数据源的属性值可能不同。这种数据语义上的歧义性是数据集成的最大难点，目前没有很好的办法解决。

### 数据变换

数据变换是采用线性或非线性的数学变换方法将多维数据压缩成较少维数的数据，消除它们在时间、空间、属性及精度等特征表现方面的差异。这方法虽然对原始数据都有一定的损害，但其结果往往具有更大的实用性。常见数据变换方法如下：

1. 数据平滑：去除数据中的噪声数据，将连续数据离散化，增加粒度。通常采用分箱、聚类和回归技术。
2. 数据聚集：对数据进行汇总和聚集。
3. 数据概化：减少数据复杂度，用高层概念替换。
4. 数据规范化：使属性数据按比例缩放，使之落入一个小的特定区域；常用的规范化方法有最小—最大规范化、z—score规范化、按小数定标规范化等。
5. 属性构造：构造出新的属性并添加到属性集中，以帮助挖掘过程。应用实例表明，通过数据变换可用相当少的变量来捕获原始数据的最大变化。具体采用哪种变换方法应根据涉及的相关数据的属性特点而定，根据研究目的可把定性问题定量化，也可把定量问题定性化。
	
	
### 数据规约

数据挖掘时往往数据量非常大，在少量数据上进行挖掘分析需要很长的时间，数据归约技术可以用来得到数据集的归约表示，它小得多，但仍然接近于保持原数据的完整性，并结果与归约前结果相同或几乎相同。

数据归约技术可以用来得到数据集的归约表示，它接近于保持原数据的完整性，但数据量比原数据小得多。与非归约数据相比，在归约的数据上进行挖掘，所需的时间和内存资源更少，挖掘将更有效，并产生相同或几乎相同的分析结果。几种数据归约的方法：

1. 维归约: 通过删除不相关的属性（或维）减少数据量。不仅压缩了数据集，还减少了出现在发现模式上的属性数目。通常采用属性子集选择方法找出最小属性集，使得数据类的概率分布尽可能地接近使用所有属性的原分布。属性子集选择的启发式方法技术有：①逐步向前选择，由空属性集开始，将原属性集中“最好的”属性逐步填加到该集合中；②逐步向后删除，由整个属性集开始，每一步删除当前属性集中的“最坏”属性；③向前选择和向后删除的结合，每一步选择“最好的”属性，删除“最坏的”属性；④判定树归纳，使用信息增益度量建立分类判定树，树中的属性形成归约后的属性子集。
2. 数据压缩: 应用数据编码或变换，得到原数据的归约或压缩表示。数据压缩分为无损压缩和有损压缩。比较流行和有效的有损数据压缩方法是小波变换和主要成分分析。小波变换对于稀疏或倾斜数据以及具有有序属性的数据有很好的压缩结果。主要成分分析计算花费低，可以用于有序或无序的属性，并且可以处理稀疏或倾斜数据。
3. 数值归约: 数值归约通过选择替代的、较小的数据表示形式来减少数据量。数值归约技术可以是有参的，也可以是无参的。有参方法是使用一个模型来评估数据，只需存放参数，而不需要存放实际数据。有参的数值归约技术有以下2种：①回归：线性回归和多元回归；②对数线性模型：近似离散属性集中的多维概率分布。无参的数值归约技术有3种：①直方图：采用分箱技术来近似数据分布，是一种流行的数值归约形式。其中V-最优和MaxDiff直方图是最精确和最实用的；②聚类：聚类是将数据元组视为对象，它将对象划分为群或聚类，使得在一个聚类中的对象“类似”，而与其他聚类中的对象“不类似”，在数据归约时用数据的聚类代替实际数据；③选样：用数据的较小随机样本表示大的数据集，如简单选样、聚类选样和分层选样等。
4. 概念分层: 概念分层通过收集并用较高层的概念替换较低层的概念来定义数值属性的一个离散化。概念分层可以用来归约数据，通过这种概化尽管细节丢失了，但概化后的数据更有意义、更容易理解，并且所需的空间比原数据少。
	
## 关联规则

从事务数据库，关系数据库和其他信息存储中的大量数据的项集之间发现有趣的、频繁出现的模式、关联和相关性。关联规则的形式如图2-左，属性-值集如图2-右。比如，在同一个交易中， 一个顾客买某品牌的啤酒后，往往也会买另一个品牌的薯条。因为挖掘关联规则的时候也许会需要在大规模的事务数据库中重复的扫描，对处理能力要求较高。下面是关联规则不同分类方式。

![guanlian](/images/DataMining/guanlian.jpg)

<center>图2 关联规则的形式和属性-值集</center>

1. 按关联规则中处理变量的类别，可将关联规则分为布尔型和数值型布尔型关联规则中对应变量都是离散变量或类别变量,它显示的是离散型变量间的关系,比如“买啤酒→买婴儿尿布”;数值型关联规则处理则可以与多维关联或多层关联规则相结合,处理数值型变量,如“月收入5000元→每月交通费约800元”。
2. 按关联规则中数据的抽象层次，可以分为单层关联规则和多层关联规则单层关联规则中,所有变量都没有考虑到现实的数据具有多个不同的层次；而多层关联规则中,对数据的多层性已经进行了充分的考虑。比如“买夹克→买慢跑鞋”是一个细节数据上的单层关联规则,而“买外套→慢跑鞋”是一个较高层次和细节层次间的多层关联规则。
3. 按关联规则中涉及到的数据维数可以分为单维关联规则和多维关联规则单维关联规则只涉及数据的一个维度（或一个变量），如用户购买的物品;而多维关联规则则要处理多维数据,涉及多个变量,也就是说,单维关联规则处理单一属性中的关系,而多维关联规则则处理多个属性间的某些关系。比如“买啤酒→买婴儿尿布”只涉及用户购买的商品,属于单维关联规则,而“喜欢野外活动→购买慢跑鞋”涉及到两个变量的信息,属于二维关联规则。
	
## 分类和预测
数据库内容丰富，蕴藏大量信息，可以用来做出智能的商务决策。分类和预测是两种数据分析形式，可以用于提取描述重要数据类的模型或预测未来的数据趋势。然而，分类是预测分类标号（或离散值），而预测建立连续值函数模型。例如，可以建立一个分类模型，对银行贷款的安全或风险进行分类；而可以建立预测模型，给定潜在顾客的收入和职业，预测他们在计算机设备上的花费。许多分类和预测方法已被机器学习、专家系统、统计和神经生物学方面的研究者提出。大部分算法是内存算法，通常假定数据量很小。最近的数据挖掘研究建立在这些工作之上，开发了可规模化的分类和预测技术，能够处理大的、驻留磁盘的数据。这些技术通常考虑并行和分布处理。

数据分类是一个两步过程（图3）。第一步，建立一个模型，描述预定的数据类或概念集。通过分析由属性描述的数据库元组来构造模型。假定每个元组属于一个预定义的类，由一个称作类标号属性的属性确定。对于分类，数据元组也称作样本、实例或对象。为建立模型而被分析的数据元组形成训练数据集。训练数据集中的单个元组称作训练样本，并随机地由样本群选取。由于提供了每个训练样本的类标号，该步也称作有指导的学习（即，模型的学习在被告知每个训练样本属于哪个类的“指导”下进行）。

第二步（图3(b)），使用模型进行分类。首先评估模型（分类法）的预测准确率。保持（holdout）方法是一种使用类标号样本测试集的简单方法。这些样本随机选取，并独立于训练样本。模型在给定测试集上的准确率是正确被模型分类的测试样本的百分比。对于每个测试样本，将已知的类标号与该样本的学习模型类预测比较。注意，如果模型的准确率根据训练数据集评估，评估可能是乐观的，因为学习模型倾向于过分适合数据（即，它可能并入训练数据中某些异常，这些异常不出现在总体样本群中）。因此，使用测试集。

如果认为模型的准确率可以接受，就可以用它对类标号未知的数据元组或对象进行分类。（这种数据在机器学习也称为“未知的”或“先前未见到的”数据）。

分类和预测具有广泛的应用，包括信誉证实、医疗诊断、性能预测和选择购物。

![Classification](/images/DataMining/Classification.jpg)

<center>图3  数据分类过程：(a) 学习：用分类算法分析训练数据。这里，类标号属性是credit_rating，学习模型或分类法以分类规则形式提供。(b) 分类：测试数据用于评估分类规则的准确率。如果准确率是可以接受的，则规则用于新的数据元组分类</center>

## 聚类分析

设想要求对一个数据对象的集合进行分析，但与分类不同的是，它要划分的类是未知的。聚类(clustering)就是将数据对象分组成为多个类或簇(cluster)，在同一个簇中的对象之间具有较高的相似度，而不同簇中的对象差别较大。相异度是基于描述对象的属性值来计算的。距离是经常采用的度量方式。聚类分析源于许多研究领域，包括数据挖掘，统计学，生物学，以及机器学习。

作为统计学的一个分支，聚类分析已经被广泛地研究了许多年，主要集中在基于距离的聚类分析。基于k-means(k-平均值)，k-medoids(k-中心)和其他一些方法的聚类分析工具已经被加入到许多统计分析软件包或系统中，例如S-Plus，SPSS，以及SAS。在机器学习领域，聚类是无指导学习(unsupervised learning)的一个例子。与分类不同，聚类和无指导学习不依赖预先定义的类和训练样本。由于这个原因，聚类是通过观察学习，而不是通过例子学习。在概念聚类（conceptual clustering）中，一组对象只有当它们可以被一个概念描述时才形成一个簇。这不同于基于几何距离来度量相似度的传统聚类。概念聚类由两个部分组成：（1）发现合适的簇；（2）形成对每个簇的描述。

聚类应用范围也比较广。在商业上，聚类能帮助市场分析人员从客户基本库中发现不同的客户群，并且用购买模式来刻画不同的客户群的特征。在生物学上，聚类能用于推导植物和动物的分类，对基因进行分类，获得对种群中固有结构的认识。聚类在地球观测数据库中相似地区的确定，汽车保险持有者的分组，及根据房子的类型，价值，和地理位置对一个城市中房屋的分组上也可以发挥作用。聚类也能用于对Web上的文档进行分类，以发现信息。作为一个数据挖掘的功能，聚类分析能作为一个独立的工具来获得数据分布的情况，观察每个簇的特点，集中对特定的某些簇作进一步的分析。此外，聚类分析可以作为其他算法（如分类等）的预处理步骤，这些算法再在生成的簇上进行处理。

## 复杂类型数据的挖掘

### 空间数据库挖掘
与关系数据库不同，空间数据描述涉及许多特征。它包含拓扑和（或）距离信息；它们按照复杂多维空间索引结构进行组织；并通过空间数据存取方法进行存取。它常常还需要进行空间推理、地理计算和知识表示技术。空间数据挖掘需要将数据挖掘与空间数据库技术结合起来。空间数据挖掘的一项重要工作就是探索有效空间数据挖掘技术，通常是将传统的空间分析方法加以扩展，重点解决高效性和可伸缩性，与数据库系统紧密结合，改进与用户的交互，以及新的知识的发现。空间分析包括空间聚类、空间分类和空间趋势分析。

1. 空间关联分析：与在事务数据库和关系数据库中挖掘关联规则类似，也可以从空间数据库中挖掘空间关联规则，然而由于空间关联挖掘需要对大量空间对象中的多种关系进行评估。这一过程可能开销很大。这里可利用一个被称为主动提炼有价值的挖掘优化方法，来帮助进行空间关联分析。该方法首先利用快速算法粗略挖掘大的数据集；然后利用更费时的算法对粗略挖掘所获数据集进行更一步处理以改善其挖掘质量。
2. 空间聚类：在一个大规模多维数据集中，借助特定的距离计算方法，空间数据聚类分析就应该识别出聚类或密集区域，聚类分析通常是将空间聚类作为示例和应用领域。
3. 空间分类和趋势分析

空间分类就是分析空间对象和推导出分类模式，如决策树，以及与一定空间性质相关的，如高速公路、河流，或地区的邻居（情况）。

空间趋势分析是完成另一项任务。也就是：发现空间维上的变化和趋势。通常趋势分析是检测时间（维）的变化，如时序数据中的时模式变化，空间趋势检测则将时间换成了空间，即研究空间中非空间和空间数据变化的趋势。例如：在从城市中心向外推移空间中的经济情势的变化趋势，或从海洋向内陆移动空间中植物或气候的变化趋势等。这样的分析中，常常用到利用空间数据结构和存取方法的回归和相关分析方法。

目前虽然在空间分类和空间趋势分析方面有一些研究，但在时空数据挖掘方面却少有人去研究。未来还需要在时空数据挖掘方法与应用做必要的研究。

空间数据挖掘可以帮助理解空间数据、发现空间关系和空间与非空间数据间关系、构造空间知识库、重组空间数据库，以及优化空间查询等。此外也可以广泛应用于地理信息系统、地理市场、遥感、图像数据库探索、医疗成像、导航、交通控制、环保和许多其它利用空间数据的领域。

### 多媒体数据库挖掘
随着视频-音频设备、C-ROMs和互联网应用的普及，许多数据库中存有大量的多媒体对象，其中包括：视频数据、音频数据、图像数据、序列数据和超文本数据（其中包含文本、链接和标记）。存储和管理大量多媒体对象的数据库系统就称为是多媒体数据库系统。典型的多媒体数据库就是美国航空航天局的A󰀂.（地球观测系统）多媒体数据库。还有其它图像数据库、视频/音频数据库、人类基因数据库和互联网数据库，它们均是多媒体数据库。在多媒体上进行的操作有下面三类：

1. 多媒体数据的相似搜索：一种是基于描述的检索系统，主要是在图象描述之上建立标引和执行对象检索，如关键字，标题，尺寸，创建时间等；另一种是基于内容的检索系统，它支持基于图象内容的检索，如颜色构成，质地，形状，对象，和小波变换等。
2. 多媒体数据的分类和预测分析: 在所报道的图像数据挖掘应用中，决策树分类是其中一种主要的数据挖掘方法。如：天空图像可由天文学家分类后作为训练样本；然后根据亮度、面积、密度、动量、方位等性质，构造识别星系、恒星和星体的模型；最后大量通过天文望远镜或太空观测站获得的图像，可以通过该模型进行测试以便识别出新的空间物体。类似的研究已使得成功地发现金星上的火山。
挖掘这类图像数据，一个重要工作就是数据的预处理，如消除噪声、数据选择和特征的抽取；因为这类图像中包含了噪声，或图片是从不同角度采集的等。除了利用模式识别中的标准方法（如边缘检测、Hough转换）外，还可以将图像分解为共扼自适应概率模型以便处理其中的不确定性。由于图像数据常常非常大，因此就需要强有力处理能力，如并行处理和分布式处理就很有用。
3. 多媒体数据中的关联规则挖掘: 从图像和视频数据库中可以挖掘关联规则。鉴于关联规则包含多媒体对象，因此至少可以将这类关联规则分为以下三类：①图像内容与非图像内容特征之间的关联。一个规则“若图像上部至少有50%是蓝色的，那它就可能代表蓝天”；就属于这类规则。因为图像内容与关键字“蓝天”发生关联。②没有空间关系的图像内容间的关联。一个规则“若一个图像包含两个蓝方框，那它就可能包含一个红色园”；就属于这类规则。因为所描述的关联均是关于图像内容的。③有空间关系的图像内容间的关联。一个规则“若两个黄方框之间有一个红色三角形，那下面就可能有一个大的椭圆物体”；就属于这类规则。因为所描述的图像相关联的对象具有空间关系。

为挖掘多媒体对象间的关联（知识），就需要将一个图像当作一个事务来处理；首先发现不同图像中频繁出现的模式；但在多媒体数据库中挖掘关联与在事务数据库挖掘有一些细微的不同。

### 时序数据和序列数据的挖掘

一个时序数据库包含随时间变化而发生的数值或事件序列。时序数据库应用也较为普遍，如：对证券市场每日波动的研究、商业交易事务序列、动态生产过程踪迹、看病治疗过程、网页读取序列等等。本节将要介绍挖掘时序数据和序列数据的几个重要方面，其中包括：趋势分析、相似搜索和（从时间相关数据中）挖掘序列模式与周期模式。

1. 趋势分析：通过对趋势，循环，季节和非规则成分的运动的系统分析，人们可以在较合理的情况下，制定出长期或短期的预测（即预报时序）。它包含四中分析，①长期或趋势变化。这是有关一个时序在一个大的时间间隔内的总体变化方向。趋势变化是由趋势曲线来表示。确定这种趋势曲线的典型方法就是最小平方方法、加权移动平均方法等。②循环变化。这是指趋势曲线所表现出的一种长期振荡。这种循环可能也可能不是周期的，也就是说它们在相同时间间隔，可能有也可能没有相似的变化模式。③季节性变化。这是有关时序数据在连续年分的各相应月中所表现出相同或几乎相同的模式。这种变化是由于每年所重复发生事件而引起的。如在春节期间的销售额猛增。④无规律变化。这是有关时序数据所表现出的漫无规律的变化。它是由随机无规律事件所引起的。如：公司人事变动的宣布。
2. 相似搜索：找出与给定查询序列最接近的数据序列。子序列匹配（subsequence matching）是找出与给定序列相似的所有数据序列。整体序列匹配（whole sequence matching）是找出彼此间相似的序列。由于现实世界中大多数应用无法确保匹配的子序列能够沿时间轴完美地排放。这样各匹配子序列之间就存在不匹配的空隙，需要对它们进行缩放和转换。在一个改进的相似模型中，用户可以设置有关参数，如：滑动窗口的大小、匹配比例和最大空隙等，对于相似性分析，时序数据通过幅度缩放和偏差转换进行规格化。两个子序列如果一个落在另一个里面相差ε（ε是由用户指定的很小值），那就认为这两个子序列相似。忽略异常值，两个序列如果都有足够不相重叠（按时间顺序排放）相似子序列，那么它们就是相似的。
3. 时序模式挖掘：挖掘序列模式就是挖掘与时间或其它序列有关的频繁发生模式。例如：“一个九个月前购买赛洋的顾客可能会在一个月内购买新的CPU”，这就是一个序列模式。鉴于许多商业交易、电信记录、天气数据和生产过程都是时序数据，因此序列模式挖掘在对这类数据进行分析时肯定是很有用的。时序模式挖掘涉及到一些参数的设置，这些参数设置的好坏对序列模式挖掘结果影响很大。第一个参数数就是时间序列的时间长度；可以将数据库中的整个序列或用户所选择的序列（如2000年）作为时间序列的长度；第二个参数数是事件窗口w，一系列在一段时间内发生的事件在特定的分析中可以看成是一起发生的。如果一个事件窗口w被设置为同序一样长，那就会发现对时间不敏感的频繁模式，也就是基本关联模式。第三个参数数就是发现模式中事件发生的时间间隔int。若将int设为0，就意味着没有间隔，也就是发现严格的连续时间序列。关于时序模式挖掘方法，大多挖掘频繁序列模式的研究都是针对不同的参数设置，以及采用Apriori启发知识和与Apriori类似作了相应改动的挖掘方法。
4. 周期分析： 挖掘周期性模式，也就是在时序数据库中搜索重复出现的模式，它也是一种重要的数据挖掘问题；并具有广泛的应用，例如：季节、潮汐、每天的交通模式，以及每周电视节目等等，都包含了一定周期性模式。挖掘周期性模式问题可以分为以下几类：①挖掘所有周期性模式；②挖掘部分周期模式；③挖掘循环关联规则。完全周期就是指时间上的每一点都对时序中的周期性行为（模式）起作用。大多挖掘部分周期模式和循环关联规则的研究都利用了Apriori启发知识，以及对Apriori挖掘方法做了相应的调整。在挖掘序列模式和周期性模式方法还可以利用约束条件来帮助挖掘工作更好地进行。

### 文本数据库挖掘

在大多文本数据库所存放的数据都是半结构化的数据，即它们既不是完全结构化也不是完全无结构的。如：一个文档中包含一些结构化的字段，诸如标题、作者、出版时间、长度和类别等，但也包含大量无结构的文本内容，诸如摘要和内容。近来在数据库研究领域，对半结构化数据集如何进行建模和操作已有了许多研究成果。此外信息检索技术，如文本索引方法，也已应用到了对非结构化文档处理上。文本挖掘有以下两种方法：

1. 基于关键字关联分析:基于关键字关联分析就是首先收集频繁一起出现的项或关键字集合；然后发现其中所存在的关联或相关联系。 与大多文本数据库分析类似，关联分析首先对文本数据库进行语法分析、抽取词根、消去stop单词等预处理；然后调用关联挖掘算法。在一个文档数据库中，可以将每个文档视为一个事务，而文档中的一组关键字则作为事务中项集合。这样文档数据库就具有以下格式： {document_id,a_set_of_keywords}。这样在文档数据库中关键字关联挖掘问题就转换为在事务数据库中项关联挖掘问题。这类算法已经有了许多。
2. 文档分类分析: 自动文档分类是一个很重要的文本挖掘任务。由于存在巨大的在线文档，因此有必要对这些文档进行分类（尽管这是一项费力的工作），以方便文档检索和之后的分析。一般自动文档分类的操作步骤如下：首先将一组已分好类的文档作为训练样本集合；然后分析训练样本集合以获得一个分类模式（以决策树形式或决策规则形式表示）。这样分类模式还需要在测试过程中不断的完善。这样获得的分类模式也可以用于其它在线文档的分类。文档分类的一个有效方法就是：探索利用基于关联的分类方法，以便能够利用常常一起出现的关键字进行分类分析。

### Web挖掘
Web挖掘有多个难点：对数据仓库和数据挖掘而言，Web太庞大了；Web页面数据太复杂：没有结构，不标准；不断增长，不断变化；广泛的用户群体；仅有很小部分的Web数据是有用的或相关的，99%的Web 信息对99% 的Web用户是无用的。
在Web上的挖掘操作有三种：web内容挖掘，web结构挖掘和web使用记录的挖掘，如图4所示。
 
![Web](/images/DataMining/Web.jpg)

<center>图4 Web挖掘操作分类</center>

1. Web内容挖掘就是Web页面上文本内容的挖掘，是普通文本挖掘结合Web信息特征的一种特殊应用。目前应用较多的是页面内容特征提取，即提取页面上重要的名词、数字等等；另一方面是对页面进行聚类，即将大量Web页面进行各种方式的分类组合，如按站点的主题类别进行聚类、按页面的内容进行聚类等，可以发现其中可能存在的隐含模式等。
2. web结构挖掘属于信息结构（IA）方面的研究内容。对于一个站点而言，按结构层次高低可以分出以下三种结构：站点结构：指的是整个站点的框架结构；页面（框架）结构：较为简单，这是由于许多网页由框架（Frame）组成而产生的；页内结构：单个网页里面也存在一定层次结构，对页内文档结构的提取有助于分析页面内容，提取页面信息。
3. Web日志（使用）挖掘就是在服务端对用户访问网络的活动记录进行挖掘，目前这方面的实际应用最为广泛，大部分集中在银行业、证券业、电子商务等方面。 Web日志挖掘的主要目的包括网络广告分析、流量、用户分类、网络欺骗预防等等。




