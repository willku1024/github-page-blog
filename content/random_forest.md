+++
title = "Random_Forest"
template = "page.html"
date = 2025-10-03T15:00:00Z
[taxonomies]
tags = ["machine learning"]
[extra]
+++

### 特点
- 集成了bagging优点
- 在随机选择样本上, 加入随机选择特征
- 假设p是总共的特征数: 则随机选择k个特征，k=sqrt(p)
- 随机性降低了相关性 (bagging的相关性太强)



### 图示

<img src="/images/random_forest02.png" style="width: 50%">

---

<img src="/images/random_forest01.png" style="width: 50%">

