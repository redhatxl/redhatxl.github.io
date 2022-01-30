# 10.Client-go源码分析之Indexer

## 一 前言

从上文我们可以看到，在SharedInformer在HandleDeltas对pop出来的数据进行处理，一部分进行维护indexer，一部分进行事件的分发，那么在Informer为什么需要Indexer呢。





## 二 Indexer功能

Indexer字面上看是索引器，就是Informer的LocalStore。肯定有人会问**索引和存储**有啥关系，那数据库建索引不也是存储和索引建立了关系么？索引构建在存储之上，使得按照某些条件查询速度会非常快。



# 

## 参考链接

