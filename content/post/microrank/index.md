---
title: "WWW21 | MicroRank: 微服务系统中谱分析方法实现的根因定位"
description: MicroRank: End-to-End Latency Issue Localization with Extended Spectrum Analysis in Microservice Environments
date: 2022-02-08T15:04:55+08:00
tags: ["Paper", "RootCauseLocalization", "AIOps"]
categories: ["Technology"]
image: 
hidden: false
comments: true
draft: true
---

> 论文标题：MicroRank: End-to-End Latency Issue Localization with Extended Spectrum Analysis in Microservice Environments
>
> 论文来源：WWW21
>
> 论文链接：[MicroRank: End-to-End Latency Issue Localization with Extended Spectrum Analysis in Microservice Environments | Proceedings of the Web Conference 2021 (acm.org)](https://dl.acm.org/doi/10.1145/3442381.3449905)
>
> 源码链接：

## TL;DR

这篇文章提出了一个利用谱分析实现根因定位的方法：MicroRank，通过收集分析异常合正常的trace数据，来定位延迟问题的根因，当MicroRank的异常检测器检测到发生延迟问题时，就会触发根因定位的过程：MicroRank首先会利用一个PageRank的打分器不同trace进行评估，然后谱分析（spectrum analysis）利用打分器得出的权重来计算根因排序。通过在广泛使用的开源微服务系统和生产系统中，对比其他根因定位方法，如CauseInfer合Sieve，MicroRank可以得到更加有效的结果。

## Algorithm

### Spectrum-based fault localization

Spectrum-based fault localization (SBFL) ，谱分析在故障定位中是比较有效且简单的方法。简单来说，在一个程序$P$和其对应的测试集合中，对于程序中的每一个元素$O\in P$，定义一个四元组：

$$(O_{ef},O_{ep},O_{nf},O_{np})$$

$O_{ef}$: 覆盖O的测试case中失败的数量

$O_{ep}$: 覆盖O的测试case中通过的数量

$O_{nf}$: 没有覆盖O的测试case中失败的数量

 $O_{np}$: 没有覆盖O的测试case中通过的数量

类似软件测试中的思路，当经过一个服务执行的异常trace越多且执行正常trace的数量越少，那么这个服务越有可能是导致异常的根因。

### System

![ScreenShot_2022-02-12_at_23.49.58@2x](ScreenShot_2022-02-12_at_23.49.58@2x.png)

MicroRank主要分为四个部分：Anomaly Detector, Data Preparator, PageRank Scorer 和 Weighted Spectrum Ranker

系统通过一个时间窗口（文中是30s），如果实时的异常大于一个期望值，就认为出现了异常，并触发根因定位，由Data Preparator来区分时间窗口内的正常/异常trace，并且构建调用图，由一个基于PageRank的打分器为每一个operation（即服务接口，下文同）进行异常分数和正常分数的打分，输入到Spectrum Ranker不断的更新权重进行排序

Anomaly Detector

异常检测器主要计算每个operation在一段时间（比如一个小时）内的平均处理时间 $\mu_o$ 和标准差 $\delta_o$ ,文中特别说明了只有在出现预测偏差的时候才需要更新上述的两个值。

在一个时间窗口内，异常检测需要统计trace涉及的operation以及对应的数量 $count_o$ ，通过下面式子计算期望一条trace的期望响应时间 $L_{excepted}$ ：

$$L_{excepted} = \sum count_o \times (\mu_o + n \times \delta_o)$$

在文中$n=1.5$，如果实际trace的响应时间大于期望值，则会被认为是异常trace，同时触发根因定位并刷新时间窗口。

### PageRank Scorer



## Experiments



## Thoughts

