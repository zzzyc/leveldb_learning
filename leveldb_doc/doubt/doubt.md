# 一些疑惑
Q: 默认的 keys 比较函数是 lexicographic byte-wise ordering ，这是指什么


A：指的是按字节大小去比较，以 `ABC` 和 `ABD` 为例，`ABC` 就是小于 `ABD` 的，因为第一个不同的字节值 `C < D`
底层是用 `memcmp` 实现的（我也是第一次遇到 `memcmp`），与 `strcmp` 的区别在于，`memcmp` 要指定比较前 `k` 个字节，但是不用去判断字符是否为 `'\0'` ，相对来说效率较高。

`leveldb/utils/comparator.cc/` 中的 `BytewiseComparatorImpl.Compare` 调用了 `leveldb/include/leveldb/slice.h` 中的 `Slice.compare` ，这个 `Slice.compare` 调用的即为 `memcmp`
****
Q: 什么是 MAINFEST

A: 

leveldb中的 MANIFEST 是一个记录 leveldb 数据库状态信息的文件，它记录了当前 leveldb 数据库中所有 SSTable 的元数据信息，包括文件名、文件大小、文件起始键、文件结束键等信息，是管理 leveldb 数据库中数据的核心数据结构之一。

在 leveldb 中，每个 SSTable 代表一个数据文件，SSTable 中存储了一段连续的键值对，并按照键的字典序排序。当往 leveldb 数据库中添加或删除数据时，会生成新的 SSTable 文件或删除旧的 SSTable 文件，这样就需要有一种机制来记录数据库中所有 SSTable 的元信息。这个机制就是 MANIFEST 文件。

MANIFEST 文件是以二进制格式存储的，由一系列记录组成，每个记录都包含了一个特定的操作类型和相应的元数据信息，可以分为以下几类：

- Version Edit：用于记录数据库的版本信息，包括 SSTable 的增、删操作等，它是 MANIFEST 文件中的核心记录类型，由多个操作记录组成；
- Put/Delete：用于记录某个键值对的插入或删除操作；
- DeleteFile：用于记录 SSTable 文件的删除操作。
当 leveldb 数据库启动时，会通过 MANIFEST 文件的记录信息恢复数据库的状态，并构建出一个新的 Version，然后根据这个 Version 进行读写操作。

需要注意的是，MANIFEST 文件是在 leveldb 数据库运行过程中不断变化的，因此需要定期进行 MANIFEST 文件的压缩和删除。在 leveldb 中，定期合并多个 SSTable 并删除旧的 SSTable 的过程称为“压实（Compaction）”，而对 MANIFEST 文件进行压缩和删除的过程称为“清理（Cleanup）”。 MANIFEST 文件中不再被引用的 SSTable 文件也需要从磁盘中删除，以释放磁盘空间。

****
Q: leveldb 中的 key 和 value 是如何存储的？

A: 

leveldb 中的 key 和 value 是分开存储的。
key 和 value 分别存储在 key block 和 data block 中。
每个 data block 会存储一些元数据，记录该 block 中的 key-value 对数量，该 block 中所有 key 的范围信息等，所以会有 key 的一些信息。

在查找数据时，首先在 key block 中对 key 进行二分查找，找到对应的 key 后，获取该 key 在 data block 中的偏移量，然后根据偏移量在 data block 中查找对应的 value 返回即可。

****