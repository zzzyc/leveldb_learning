# Slice 介绍
Slice 是一个很简单的数据结构，可以理解为字节型字符串。

`Slice` 只有两个成员变量：`const char* data_` 和 `size_t size_` 。

 `data_` 是一个常量指针，意思是不可通过 `data_` 修改 `data_` 指向的字节数组的内容。

`size_` 表示字节数组的长度。

注意：
- `Slice` 表示的是字节数组，所以是允许 `'\0'` 存在的。

- `Slice` 指向的字节数组在使用时，需要调用者保证在 `Slice.data_` 指向的内存未被销毁。

# Slice 的构造函数
```cpp
// 默认构造函数，默认为空，长度为 0，使用的是字符串字面量
Slice() : data_(""), size_(0) {}

// C 风格字符串为参数的构造函数，长度取决于 s 开始的第一个 '\0' 出现的位置
Slice(const char* s) : data_(s), size_(strlen(s)) {}

// 字符指针和长度 n，说明指定了 d[0,n-1]作为 Slice 指向的字节数组
Slice(const char* d, size_t n) : data_(d), size_(n) {}

// C++ 风格字符串为参数的构造函数，长度和内容均取决于 s
Slice(const std::string& s) : data_(s.data()), size_(s.size()) {}
```

# Slice 的拷贝构造函数和赋值函数都是 default 

# Slice 的成员变量相关函数
```cpp
// 返回一个指向的字节数组的起始位置的指针
const char* data() const { return data_; }

// 返回指向的字节数组的长度
size_t size() const { return size_; }

// 根据字节数组的长度判断是否为空
bool empty() const { return size_ == 0; }

// 返回指向的字节数组中的第 i 个字节，假设 i 小于数组长度
char operator[](size_t n) const {
    assert(n < size());
    return data_[n];
}

// 清空当前 Slice
void clear() {
    data_ = "";
    size_ = 0;
}

// 移除当前 Slice 的前 n 个字节，
// 等价于 data_ 指向数组的起始位置后移 n 个，且数组的大小减小 n
// 假设 n 小于等于移除前的字节数组的长度
void remove_prefix(size_t n) {
    assert(n <= size());
    data_ += n;
    size_ -= n;
}

// 返回 C++ std::string 风格的字符串，但是会重新开辟一块内存
std::string ToString() const { return std::string(data_, size_); }

// 判断 Slice x 是否是当前 Slice 的前缀
bool starts_with(const Slice& x) const {
    return ((size_ >= x.size_) && (memcmp(data_, x.data_, x.size_) == 0));
}


// 判断两个 Slice 是否相等，长度相等且每个位置的字节都相等
inline bool operator==(const Slice& x, const Slice& y) {
  return ((x.size() == y.size()) && (memcmp(x.data(), y.data(), x.size()) == 0));
}

// 判断两个 Slice 是否不相等
inline bool operator!=(const Slice& x, const Slice& y) {
    return !(x == y);
}
```

## Slice 的 compare 函数
```cpp
// 判断两个 Slice 的大小
// r > 0 表示 *this > b
// r == 0 表示 *this == b
// r < 0 表示 *this < b
inline int Slice::compare(const Slice& b) const {
    // 取到较小的 Slice 的长度
    const size_t min_len = (size_ < b.size_) ? size_ : b.size_;

    // 判断两个数据在 min_len 范围内是否每个字节都相等
    int r = memcmp(data_, b.data_, min_len);

    // 都相等就看哪个 Slice 更长，两个 Slice 一样长时则两个 Slice 相等
    if (r == 0) {
        if (size_ < b.size_)
            r = -1;
        else if (size_ > b.size_)
            r = +1;
        }
    }
    // 如果 min_len 内已经存在不等，则直接可以返回答案
    return r;
}
```
#