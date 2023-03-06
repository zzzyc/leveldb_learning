leveldb
=======
Jeff Dean, Sanjay Ghemawat

leveldb 库提供持久化的 KV 存储。keys 和 values 可以是任意的字节数组。 keys 会根据用户指定的比较函数进行排序。（补充：默认的比较函数是 lexicographic byte-wise ordering 的，即按照字节的值大小去比较）

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

****
... 待续