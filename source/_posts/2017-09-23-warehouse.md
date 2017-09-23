---
title: 数据仓库读书笔记
date: 2017-09-23 11:38:13
description: 很多经典的结论，基本也和工业界数据仓库的脉络相似，但是也有部分内容过时了，尤其不适应互联网背景和”国情“。但是作为启发式读物还是不错的，刺激读者思考和发散。
category: 读书笔记
tags: [数据仓库]
---

前不久读的一本经典读物《数据仓库》，整理了一些要点。

# 思维导图

![warehouse](/images/warehouse/warehouse.png)

# 文字版

    决策支持系统的发展
      演化
        蜘蛛图
            抽取程序是决策系统的基础，最受欢迎，起初只是抽取，随后是抽取之上的抽取，接着再次的抽取，形成“蜘蛛图”
      自然演化式体系结构的问题
        数据缺乏可信
        体系结构化环境
            部门环境=数据集市层=在线分析处理层OLAP=多维DBMS层
            在线分析处理OLAP vs 在线事务处理OLTP
        体系结构化环境中的数据集成
            抽取/转换/装载（ETL）是数据从操作型环境进入数据仓库的一个必须步骤，这个集成过程秩序进行一次，却是必须的
      开发生命周期
        操作型环境通常是面向应用的，非集成的；数据仓库则必须是集成的
        操作型环境中使用的是传统的系统开发生命周期SDLC，瀑布是开发方法，一个活动结束后下一个才触发开始；数据仓库的开发则以CLDS的开发周期进行，与SDLC顺序相反。传统的SDLC由需求驱动，CLDS由数据开始，得到数据后将数据集成，然后检查数据偏差，最后理解系统需求，不断调整系统，螺旋式开发方法。
      监控数据仓库环境
        确定发生了什么增长，发生在什么地方，以什么速率发生
        确定那些数据正在被使用
        估算最终用户得到的响应时间
        确定谁在实际使用数据仓库
        说明最终用户正在使用数据仓库种的多少数据
        精确指出数据仓库何时被使用
        确定数据仓库中有多少数据被使用
        监测数据仓库使用率水平
    数据仓库环境
      面向主题
        每一个主要主题域都是以一组相关的表来具体实现的
      粒度
        粒度问题是设计数据仓库最重要的方面
        细节程度越高，粒度级就越低；相反，细节程度越低，粒度级就越高。
        数据粒度级别太高，意味着开发人员必须话费大量设计和开发资源对于数据进行拆分
        网络日志数据是一个粒度级别太低的例子，点击数据流要适应数据仓库环境，需要进行编辑、过滤和汇总
        数据仓库低级别粒度的一个好处是灵活性，另一个好处是可以包含整个企业活动和事件的历史
        双重粒度：当数据仓库拥有大量数据时，在细节部分考虑使用双重（多重）粒度级别是很有意义的。例如：轻度综合数据+真实细节数据。
      分区设计方法
        在数据仓库中的细节数据，应该如何分区?
        数据分区的目的是划分成小的可管理的物理单元，具有更大灵活性。重构，索引，顺序扫描，重组，恢复，监控等
        分区标准：时间，业务范围，地理位置，组织单位等
      数据仓库中的数据组织
        数据堆积模式 => 轮转综合数据存储
        日数据=>周槽=>月槽=>年槽
            不同的轮转级别数据，不同的数据存储方式，不同的粒度级别（集成程度），不同的读写效率
      数据的同构/异构
        数据仓库中的数据清理
            数据并非永久的注入数据仓库，它有自己的生命周期
            加入到失去原有细节的轮转综合文件
            从高性能的介质转移到大容量介质上
            数据真正的清除
            数据从体系结构的一个层次转到另一个层次，比如ODS到数据仓库
      数据仓库中的错误数据
        如何处理错误数据?
            找到错误条目，更新
                    需要修改更多条目，报表失去一致性
            加入修正条目
                    需要修改很多条目，修改公式可能非常复杂
            重置最后的结果
                    需要重置操作和过程进行约定，不能对于过去的错误进行解释
        没有最好的错误处理方法，根据条件取舍
    设计数据仓库
      数据仓库和数据模型
      数据模型和迭代式开发
      元数据
      数据周期-时间间隔
      数据仓库记录的触发
        事件
        快照的构成
      概要记录
      管理大量数据
      从数据仓库环境到操作型环境
    数据仓库的粒度
      粗略估算
      规划过程的输入
      确定粒度级别
        在第一次设计中，如果有50%是正确的，那么整个设计就是成功的
      一些反馈循环技巧
        在数据仓库的建造中，如果已经知道了至少一般需求，还不开始建造，同样也是不明智的
        在少量数据时，可能不需要双重粒度
      填充数据集市
        为了合适地填充数据集市，数据仓库种的数据必须在一个所有数据集市所需要的最低的粒度级别上
    数据仓库和技术
      管理大量数据
      管理多种介质
      索引和监控数据
      多种技术的接口
      数据存放位置的控制
      数据的并行存储和管理
      语言接口
        SQL
      数据压缩
      复合主键
      变长数据
        性能问题
      加锁技术
      快速恢复
    分布式数据仓库
    主管信息系统和数据仓库
      EIS
        趋势分析和发现
        关键比例指标度量和跟踪
        向下钻取分析
        问题监控
        关键性能指标监控
      向下钻取分析
        向下钻取数据是指从一个汇总数据开始，将该汇总数据分解成一组更细致的汇总数据
        通过获取汇总数据下的细节数据，管理者能够知道究竟正在发生什么，特别是哪里出现异常
      到哪里取数据
        操作型环境 => 数据仓库 => 部门(数据集市) => 个体
        EIS功能使用（从数据仓库）
            用数据仓库提供汇总数据
            用数据仓库结构支持向下钻取处理
            用数据仓库的元数据为DSS分析员规划建造EIS系统
            用数据仓库的历史内容支持管理人员所需的趋势分析
            用数据仓库的集成数据观察整个公司的运行概况
      事件映射
        趋势曲线上的事件标记：从另一个角度观察收益数据和事件之间的关系
        对某些类型的事件，事件映射是度量事件结果的唯一方式
    外部数据与数据仓库
    迁移到体系结构化环境
    数据仓库和Web
      数据从Web转移到数据仓库
        Web数据收集到日志
        日志数据在通过粒度管理器时进行处理
        粒度管理器将提炼后的数据传递给数据仓库
      对Web的支持
        容纳巨量数据的能力
            数量 vs 可用性 vs性能
        存取集成数据的能力
            Web数据和其他企业数据的结合？
            企业数据可以同一个或多个Web站点数据汇合和集成，形成一个单一的共同数据源
        提供优良性能的能力
      ODS是数据和Web环境之间的唯一联系
        能够确保联机事务始终能迅速、一致的处理
    非结构化数据和数据仓库
      两层数据仓库
        （非结构化数据）数据迁移到结构化环境
        结构、非结构两层数据仓库
            非结构化数据仓库
                    数据以低粒度存在
                    存在一个隶属于数据的时间要素
                    数据在一定主题范围或“主题"下规范组织起来
    大型数据仓库
      快速增长的原因
      庞大数据量的影响
      数据在不同介质的存储
      环境间数据转移
      总费用
      最大容量
    关系模型和多维模型数据库
      管理模型 vs 多维模型
      简历独立数据集市
    数据仓库高级话题
      追踪数据仓库中的数据流
        推和拉
      ODS：概要记录和性能
      公司信息工厂(CIF）
        分析
            最有前景的数据分析之一就是面向未来的分析
            数据仓库中的数据是历史数据
            利用数据仓库中数据为基础，进行未来规划（预测）
        政府信息工厂（GIF）
      数据进入企业后是有生命周期的，会变老，通过不同的方式使用
        数据生命周期应该与数据仓库相吻合，与支持数据仓库的部件相吻合
    数据仓库的成本论证和投资回报
      宏观角度上，影组织结构的因素很多，阐释数据仓库的意义是不容易的。微观角度相对容易
      没有数据仓库的公司信息的成本相对于有数据仓库的公司，要高很多
      数据仓库增加信息的时间价值，提供集成数据平台，为历史数据提供方便的存储池
    数据仓库和ODS
      互补的结构
        ODS支持高性能的处理；但当事务需要大量数据时，ODS处理时间长
        ODS种的一些数据是为了高性能的事务处理过程设计的
        小的事务，消耗少量的资源
        ODS中的数据是实时值，数据仓库是历史值
        ODS记录 = 概要记录。是对客户的数据多次观察，分析概括得出的
        ODS比数据仓库小得多
        多个ODS
    企业信息依从准则和数据仓库
    最终用户社区
      四种DSS最终用户
        农民
            主导地位，可预测的日常规律查询
        探险者
            不可预测，启发模式地发现问题
        矿工
            挖掘数据，分析数据，解决问题，和探险者一前一后工作
        旅行者
            发散地解释问题，具备更大的数据广度
      不同的最终用户表现不同的地方
        频繁访问数据 vs 不频繁
        ROI（投资回报率）分析和成本
    数据仓库设计的复查要目