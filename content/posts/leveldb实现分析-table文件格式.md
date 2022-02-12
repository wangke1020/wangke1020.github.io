---
title: leveldb实现分析-table文件格式
date: 2017-08-30 22:56:37
# tags: leveldb 
---

### 介绍

在leveldb的源码目录中，table文件的实现位于主目录下的table文件夹下，主要定义和实现了table文件和block的格式。
```
table
├── block_builder.cc
├── block_builder.h
├── block.cc
├── block.h
├── filter_block.cc
├── filter_block.h
├── filter_block_test.cc
├── format.cc
├── format.h
├── iterator.cc
├── iterator_wrapper.h
├── merger.cc
├── merger.h
├── table_builder.cc
├── table.cc
├── table_test.cc
├── two_level_iterator.cc
└── two_level_iterator.h

```


### table文件的格式

table文件格式如下：
```
<beginning_of_file>
[data block 1]
[data block 2]
...
[data block N]
[meta block 1]
...
[meta block K]
[metaindex block]
[index block]
[Footer]        (fixed size; starts at file_size - sizeof(Footer))
<end_of_file>
```
其中data block 存储的是Key Value数据，meta block 存储的是Filter信息和其他一些元数据。
metaindex block和index block分别记录了meta block和data block各个block的offset和length。

以上所有block的格式都在block_builder.cc类定义的，block中数据按照key排序存放。

Footer中存储了metaindex block和index的位置和长度。
 
### Footer格式
leveldb用BlockHandle来封装block的offset和length，在table/format.h中，BlockHandle组成如下：
```cpp
// BlockHandle is a pointer to the extent of a file that stores a data
// block or a meta block.
class BlockHandle {

 ...

 private:
  uint64_t offset_;
  uint64_t size_;
};
```
Footer中则存储了metaindex block和index的BlockHandle：
```cpp
// Footer encapsulates the fixed information stored at the tail
// end of every table file.
class Footer {
 
 ...

 private:
  BlockHandle metaindex_handle_;
  BlockHandle index_handle_;
};
```
### block格式

block中存储了key value对，每一个key value队是一个entry。每个entry的格式如下：
```
     shared_bytes: varint32
     unshared_bytes: varint32
     value_length: varint32
     key_delta: char[unshared_bytes]
     value: char[value_length]
```
为了减少磁盘占用，block中的key存储采用了前缀压缩的方式，key信息记录的是与上一个key相差的字节信息。假设新插入如下3个key value对：
```
abc:1
abcde:2
a:3
```
对应的两个entry为
```
|shared_bytes|unshared_bytes|value_length|key_delta|value
|0           |3             |1           |"abc"    |"1"
|3           |2             |1           |"de"     |"1"
|0           |1             |1           |"a"      |"1"
```

另外， 对于每N个key， 不适用前缀压缩。此时shared_bytes=0。N被记作restart point。restart point可以通过Options按需配置。

在调用BlockBuilder::Finish()完成创建Block时，需要restart point数组记录在Block中：
```cpp
Slice BlockBuilder::Finish() {
  // Append restart array
  for (size_t i = 0; i < restarts_.size(); i++) {
    PutFixed32(&buffer_, restarts_[i]);
  }
  PutFixed32(&buffer_, restarts_.size());
  finished_ = true;
  return Slice(buffer_);
}
```
<!-- more -->
