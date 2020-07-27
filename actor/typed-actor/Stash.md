# Stash (缓存消息)

## Introduction 

缓存可以让actor缓冲一下暂时不能处理的消息，比如:
actor在接收第一个业务消息之前应该初始化状态或者加载一下初始化的资源

StashBuffer 是一个缓冲区，缓存的消息被保留在内存中，直到它们被取消缓存或者actor被停止并进行垃圾回收。
存储过多的消息会占用很多内存，甚至会有OutOfMemoryError的风险。
StashBuffer是有边界的，在创建的时候要设置它能容纳多少消息
尝试存储的消息超过了容量会抛出StashOverflowException异常，可以在存储消息之前调用StashBuffer.isFull来避免这种情况。

通过调用 `unstashAll`来取消缓存消息时，消息会按照添加的顺序依次处理。除非出现异常，否则所有的消息都会被处理。
在`unstashAll`完成之前，actor对其他消息是没有反应的

