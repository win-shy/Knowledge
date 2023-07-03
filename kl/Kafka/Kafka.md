[toc]

# 如果副本在`ISR`中停留了很长时间表明什么

`ISR` 是一组与 `leader`完全同步的消息副本，也就是说`ISR`中包含了所有提交的消息。`ISR`应该总是包含所有副本，直到出现真正的故障。如果一个副本从`leader`中脱离，那将会被`ISR`删除。

如果一个副本在`ISR`中保留很长的事件，说明该副本可以向`leader`收集数据那样快速地获取数据。

如果首选副本不在`ISR`中，`controller` 将无法从`leadership`转移到首选副本。

#  如果首选副本不在`ISR`中会发生什么

如果首选副本不在`ISR`中，那么`controller`无法将`leadership`转移到首选副本。

# 有可能在生产后收到消息偏移

作为生产消息的用户，可以通过返回的`RecordMetadata `来获取到偏移量，分区等信息

# 分区

生产者分区器根据参数来区分数据要发往的分区：

1. 有 Partition：直接将数据存入对应的分区
2. 没有 Partition 有 Key：将通过 Key的Hash值 % 主题的分区数 来得到一个 Partition 值
3. 没有 Partition 没有 Key：采用 Sticky Partition（粘性分区器），会随机选择一个分区，并尽可能的一直使用这个分区，待该分区的 `ProducerBatch` 满了或者已完成，再随机选择其他的分区（不会重复使用上一次的分区）