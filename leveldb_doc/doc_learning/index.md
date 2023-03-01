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

****
... 待续