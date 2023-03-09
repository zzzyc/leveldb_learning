leveldb
=======
Jeff Dean, Sanjay Ghemawat

leveldb 库提供持久化的 KV 存储。keys 和 values 可以是任意的字节数组。 keys 会根据用户指定的比较函数进行排序。（补充：默认的比较函数是 [lexicographic byte-wise ordering](../doubt/doubt.md) 的，即按照字节的值大小去比较）

## 打开 Databse
一个 leveldb 数据库的 name 绑定到文件系统中的一个目录。数据库的所有内容都存储在这个文件目录下。下面是示范如何打开一个数据库，如果打开的不存在的话，先创建。
```cpp
#include <cassert>
#include "leveldb/db.h"

leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
assert(status.ok());
...
```

如果数据库已经存在，你打开时想要抛出一个错误，在 `leveldb::DB::Open` 前添加如下语句（大致意思是使用时默认是不存在的，默认是重新创建，如果存在的话，不符合预期，故设置报错）：
```cpp
options.error_if_exists = true;
```

## Status
上述的 `leveldb::Status` 类型，这个类型的值被作为大多数 leveldb 中函数的返回值，即用来判定当前操作是否产生了错误。
如下是一个例子，`s.ok() == false` 表示当前操作产生错误，`s.Tostring()` 表示错误信息：
```cpp
leveldb::Status s = ...;
if (!s.ok()) cerr << s.ToString() << endl;
```

## 关闭 Database
关闭数据库，只要 `delete` 数据库对象即可，如下是示例：
```cpp
... open the db as described above ...
... do something with db ...
delete db;
```

## 读写
leveldb 提供了 Put、Delete 和 Get 方法去修改或查询数据。下面是一个例子，这个例子将 `key1` 在数据库中存储的数据移动到了 `key2` 。
```cpp
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) s = db->Put(leveldb::WriteOptions(), key2, value);
if (s.ok()) s = db->Delete(leveldb::WriteOptions(), key1);
```

## 原子更新
注意，如果进程在 `Put key2` 之后，且 `Delete key1` 之前挂了（数据库宕机了），那么就会有一个值对应多个键，这个问题可以用 `WriteBatch` 来进行原子操作，保证原子性：
```cpp
#include "leveldb/write_batch.h"
...
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) {
  leveldb::WriteBatch batch;
  batch.Delete(key1);     // 先 Delete key1，后 Put key2 ，是保证当 key1==key2 时，数据也不会丢失
  batch.Put(key2, value);
  s = db->Write(leveldb::WriteOptions(), &batch);
}
```

`WriteBatch` 维护一个操作序列，以保证数据库按顺序去执行操作。我们在 Put 前调用了 Delete，这样保证当 key1 和 key2 相同时，我们不会错误地删除该值。除了保证原子性外，通过 `WriteBatch` 进行一系列读写操作，会比单次执行更快。

## 同步写
默认情况下，每个读写操作都是异步的：数据库将操作交给操作系统后就返回了。从操作系统内存到底层的持久化存储是异步进行的，当然你也可以使得每次操作是同步的，即使得每次操作数据库返回时，该次操作的结果已经持久化存储了。`WriteOptions.sync = true` 即可开启同步（在 Posix 系统上，这个同步是调用 `fsync(...)` 或者 `fdatasync(...)` 或者 `msync(..., MS_SYNC)` 这些系统调用来实现的）。
```cpp
leveldb::WriteOptions write_options;
write_options.sync = true;
db->Put(write_options, ...);
```

异步写通常比同步写快一千多倍。异步写的缺点是如果机器崩溃可能会导致最后几个更新丢失。

注意，这仅仅是写进程的崩溃（即，不是宕机）不会造成任何损失，因为即使 sync 为 false，更新也会在被认为完成之前从进程内存中推入操作系统。

<br/>

异步写可以被更安全地使用。例如，当加载大量数据到数据库中，你可以在崩溃后重启批量加载来处理丢失的更新。

混合方案也可以实现，其中每 n 次写入都是同步的，在发生崩溃时，大容量负载将在上次运行完成最后一次同步写入后重启。（同步写入可以更新一个标记，该标记描述崩溃时在哪里重新启动）


`Writebatch` 提供一个可替代的异步写方式。在一个 `WriteBatch` 中执行多次操作，通过同步写一起执行。（即，`write_opions.sync` 设置为 `true`）。同步写的额外开销会被均摊到 `WriteBatch` 中的每个操作中


## 并发
一个数据库在同一时间只能被一个进程打开。leveldb 从操作系统获得一个锁，以防误用（一个数据库同一时间误被多个进程打开）。

在一个进程的多个并发线程中，`leveldb::DB` 对象可以被安全地共享使用（线程安全）。即，同一进程中的不同线程可能在同一个数据库对象上执行写入或者获取迭代器或者调用Get操作，但也不需要额外的同步操作。（leveldb的实现已经做好了需要的同步操作）然而其他对象，像迭代器和 `WriteBatch` 可能也需要额外的同步操作。如果两个线程共享一个对象，他们必须使用自己的锁来实现安全的访问（即自己加锁来实现线程安全）。在头文件中有更多的细节。


## 迭代器
下面是一个如何打印数据库中所有 key 和 value 对的例子。


```cpp
leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  cout << it->key().ToString() << ": "  << it->value().ToString() << endl;
}
assert(it->status().ok());  // Check for any errors found during the scan
delete it;
```

下面是从 `start` 开始，遍历直到 `limit - 1` ，即 `[start, limit)` 。
注意这里是 `std::string` 的比较。
```cpp
for (it->Seek(start);
   it->Valid() && it->key().ToString() < limit;
   it->Next()) {
  ...
}
```

下面是反向迭代
```cpp
for (it->SeekToLast(); it->Valid(); it->Prev()) {
  ...
}
```

## 快照

快照为 KV 存储的整个状态提供了一致的只读视图。`ReadOptions::snapshot` 如果不是 `NULL` ，表示在进行读操作时应该操作数据库的指定版本（即快照）。如果 ReadOptions::snapshot` 如果是 `NULL` ，读操作将在当前状态的数据库上直接读取。

快照由 `DB::GetSnapshot()` 方法创建。

```cpp
leveldb::ReadOptions options;
options.snapshot = db->GetSnapshot();
... apply some updates to db ...
leveldb::Iterator* iter = db->NewIterator(options);
... read using iter to view the state when the snapshot was created ...
delete iter;
db->ReleaseSnapshot(options.snapshot);
```

注意快照不再使用时，需要使用 `DB::ReleaseSnapshot` 释放，以使得之后读取的数据是当前数据库的最新状态下的数据。

## Slice
上述 `it->key()` 和 `it->value()` 调用的返回值类型都是 `leveldb::Slice` 。`Slice` 是一个简单的数据结构，包含 `size_` 表示长度，`data_` 是一个指针，指向堆内存区域的字节数组。 

因为不需要复制大的 key 和 value ，故采取返回一个 `Slice` 来代替返回一个 `std::string` 的策略。此外，leveldb 的方法不会返回以非空结束的 C 风格的字符串，这是因为 leveldb 的 keys 和 values 允许包括 `\0` 字符。 

C++ 字符串和以非空结束的 C 风格字符串可以被轻易地转换称为一个 `Slice`：
```cpp
leveldb::Slice s1 = "hello";
std::cout << s1.ToString() << std::endl;

std::string str("world");
leveldb::Slice s2 = str;
std::cout << s2.ToString() << std::endl;
```

`Slice` 也可以轻易地被转换回 C++ 字符串
```cpp
std::string str = s1.ToString();
assert(str == std::string("hello"));
```

使用 `Slice` 时要小心，因为调用者要确保在使用 `Slice` 时， `Slice.data_` 指向的外部的字节数组内存还未被释放。下面的例子就存在内存问题：
```cpp
leveldb::Slice slice = "hello";
if (...) {
  std::string str = ...;
  slice = str;
}
std::cout << slice.ToString() << std::endl;
```
当 `if` 语句出了作用域，`str` 的内存将被释放，`Slice.data_` 对应的内存也是无效的了。

## 比较器

上述例子用的是 leveldb 的默认比较器，可以理解为根据 ascii 码的大小去进行比较。你可以在打开数据库时提供一个自定义比较器。
例如，假设每个数据库的 key 包括两个数字，我们要对第一个数字排序，如果第一个数字相同，再对第二个数字排序（即第一关键字和第二关键字）。定义一个合适的派生类 `leveldb::Comparator` 来实现自定义比较器：

```cpp
class TwoPartComparator : public leveldb::Comparator {
 public:
  // Three-way comparison function:
  //   if a < b: negative result
  //   if a > b: positive result
  //   else: zero result
  // 如果 a 小于 b，返回负数
  // 如果 a 大于 b，返回正数
  // 如果 a 等于 b，返回 0
  int Compare(const leveldb::Slice& a, const leveldb::Slice& b) const {
    int a1, a2, b1, b2;
    ParseKey(a, &a1, &a2); // ParseKey 是一个解析 key 的方法，将 key 拆分成第一关键字和第二关键字
    ParseKey(b, &b1, &b2);
    if (a1 < b1) return -1;
    if (a1 > b1) return +1;
    if (a2 < b2) return -1;
    if (a2 > b2) return +1;
    return 0;
  }

  // 先忽视下面的方法
  const char* Name() const { return "TwoPartComparator"; }
  void FindShortestSeparator(std::string*, const leveldb::Slice&) const {}
  void FindShortSuccessor(std::string*) const {}
};
```

用上述自定义的比较器创建数据库：

```cpp
TwoPartComparator cmp;
leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
options.comparator = &cmp;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
```

## 向后兼容
在打开一个已经存在的数据库时，比较器需要和创建时的比较器进行对比，如果两次不兼容，那么第二次排序就是不对的了。Leveldb新建数据库时，会向[MAINFEST](../doubt/doubt.md)写入一个比较器的名称，下次打开时会检查名称是否相同，来判断兼容性。如果当前打开的数据库的数据可以丢弃，那么你可以采用相同 `Name()` 的比较器，但是为新键增加版本号以及修改比较器函数，使得比较器会找到键的版本号，根据版本号来决定如何排序。注意：每次数据库被打开时，都会按照最新的比较器函数来对数据进行排序。

## 性能指标
通过更改 `include/options.h` 中定义的类型的默认值来调整性能。`WriteOptions`、`ReadOptions`、`Options` 和 `Comparator`。
通过更改这些选项中的参数，可以更改 LevelDB 的行为以实现更好的性能。

## 数据块 (Block)
leveldb 将相邻的 key 分组放在同一 block 中，这样一个 block 就是持久化存储中传输的单位。默认的 block 大小是 4096 个未压缩的字节。恰当的 block 大小会优化数据库的性能。
每个 block 有一个对应的 block header，用于存储该 block 的元信息，如 block 大小，最大和最小键值等。此外，每个 block 还可以被压缩以节省压缩空间。

在 leveldb 中，block 是从磁盘读取的最小单位，当读取一个键值对时，首先会定位该键值对所在的 block ，并从磁盘中读取整个 block 。一旦读取到 block，leveldb 会在内存中解压缩该 block 并将其存储在内存中。

对于写入操作，leveldb 将待写入的键值对存储在一个内存的数据结构中，称为 memtable，当 memtable 中的数据达到一定的阈值时，leveldb 会将 memtable 转换为一个不可变的 SSTable(Sorted String Table) ，即有序字符串表，所有的键值对按照键的字典序排序，并且被分割成多个 block 存储在磁盘上。

注意在 leveldb 中，key 和 value 是分开存储的。

## 压缩
每个 block 在写入磁盘前都会被单独压缩。由于默认压缩方法非常快速而且自动禁用无法压缩的数据，默认情况下开启压缩。极少数情况下，应用程序希望完全禁用压缩，但是只有在 benchmarks 显示性能提高才会完全禁用，下面是一个示例：
```cpp
leveldb::Options options;
options.compression = leveldb::kNoCompression;
... 
leveldb::DB::Open(options, name, ...) 
....
```

leveldb 使用多种压缩方法，包括 Snappy、Zlib、Bzip2 等。默认情况下使用 Snappy，因为其速度很快。
数据写入时，leveldb 会把每个 block 单独压缩后写入磁盘，如此可以显著减少磁盘空间占用，提高数据读取效率，对于大规模数据集，也可以降低 CPU 和 内存的使用了。

极少数情况下，一些应用程序会遇到无法压缩的数据，此时开启压缩反而会影响性能，此时可以选择关闭压缩，但是需要通过 benchmarks 来验证是否真的会提高性能。

****
... 待续