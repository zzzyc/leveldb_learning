# 一些疑惑
Q: 默认的 keys 比较函数是 lexicographic byte-wise ordering ，这是指什么


A：指的是按字节大小去比较，以 `ABC` 和 `ABD` 为例，`ABC` 就是小于 `ABD` 的，因为第一个不同的字节值 `C < D`
底层是用 `memcmp` 实现的（我也是第一次遇到 `memcmp`），与 `strcmp` 的区别在于，`memcmp` 要指定比较前 `k` 个字节，但是不用去判断字符是否为 `'\0'` ，相对来说效率较高。

`leveldb/utils/comparator.cc/` 中的 `BytewiseComparatorImpl.Compare` 调用了 `leveldb/include/leveldb/slice.h` 中的 `Slice.compare` ，这个 `Slice.compare` 调用的即为 `memcmp`
****