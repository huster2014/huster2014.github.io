---
layout: post
title: ceph osd中omap的实现
---

   麦子迈博客，“ObjectStore主要接口分为三部分，第一部分是Object的读写操作，类似于POSIX的部分接口，第二部分是Object的属性(xattr)读写操作，这类操作的特征是kv对并且与某一个Object关联。第三部分是关联Object的kv操作(在Ceph中称为omap)，这个其实与第二部分非常类似，但是在实现上可能会有所变化。
   
   目前ObjectStore的主要实现是FileStore，也就是利用文件系统的POSIX接口实现ObjectStore API。每个Object在FileStore层会被看成是一个文件，Object的属性(xattr)会利用文件的xattr属性存取，因为有些文件系统(如Ext4)对xattr的长度有限制，因此超出长度的Metadata会被存储在DBObjectMap里。而Object的omap则直接利用DBObjectMap实现。因此，可以看出xattr和omap操作是互通的，在用户角度来说，前者可以看作是受限的长度，后者更宽泛(API没有对这些做出硬性要求)。”

DBObjectMap向上提供了操作omap和xattr的接口，如set_，get_和rm_操作，以及clone和sync操作等。每个object的omap由一个header和多个kv对构成，xattr由多个kv对构成。DBObjectMap实现了omap克隆操作的0-copy，但是xattr则采用直接复制的方式实现克隆；omap还对外提供了一个迭代器，用来实现omap中kv的遍历操作，为了支持0-copy操作，迭代器的实现还是有点复杂的。
 
上图是取自麦子迈的博客，用来说明0-copy的实现原理。
图中的header是DBObjectMap内部使用的结构体(不是omap中的header)，每个object对应一个header，记录分配给object的序列号，在leveldb中，object拥有的kv对都以该序列号作为前缀，从而方便在leveldb中进行查找。
当克隆某个src object时就为src和dst都分配一个新的序列号，并创建相应的新的header结构体，而src原来的header将作为它们的parent header，当查找某object的kv值时，可以根据parent header的序列号在leveldb中查找相应的数据，这部分操作被封装在DBObjectMap的omap迭代器中。
## 一个全局的序列号分配器
```c++
struct State {
    __u8 v;
    uint64_t seq;
}
```

每个object都被分配一个全局唯一的序列号
header结构体记录序列号，以及clone操作所产生的父子关系
```c++
struct _Header {
    uint64_t seq;
    uint64_t parent;
    uint64_t num_children;
    coll_t c;
    ghobject_t oid;
    SequencerPosition spos;
}
```

// omap的迭代器，其中KeyValueDB::Iterator继承了DBObjectMapIteratorImpl，并实现了具体的key()和value()方法，DBObjectMapIteratorImpl的key()和value()方法直接调用KeyValueDB::Iterator中对应的方法来实现
// 迭代器中还有一个complete_iter，它指向一个区间表，每个表项为[begin,end)，标识该区间内的所有kv对都以当前header中的序列号为前缀，而不是以parent header的序列号为前缀。它用来实现kv的rm操作，当rm掉object的某对kv时，该kv可能在parent header中，而parent header中的kv是共享，不能直接删除，否则会影响到其它的object，于是先从parent header中位于被删除的kv对附近的多个连续的kv对直接复制的当前object下，并创建一个区间，标识该区间内的所有kv都被拷贝到当前object中，无需访问parent header，这样一来就可以避免操作parent header而完成删除工作了…当然这其中还包括了区间的合并，以及parent header的释放问题，还是有点复杂滴.
```c++
class DBObjectMapIteratorImpl : public ObjectMapIteratorImpl {
  public:
    DBObjectMap *map;
    MapHeaderLock hlock;
    Header header;

    ceph::shared_ptr<DBObjectMapIteratorImpl> parent_iter;
    KeyValueDB::Iterator key_iter;
    KeyValueDB::Iterator complete_iter;
    ceph::shared_ptr<ObjectMapIteratorImpl> cur_iter;
    int r;

    bool ready;
    bool invalid;
 }
```

先看一下克隆操作的实现，可以瞥见它的实现原理：
1)	获取parent header
2)	分别为src和dst构建两个新的header，它们的父header都指向parent
3)	将parent header存放到以sys_prefix(parent)为前缀，key= HEADER_KEY的记录中
4)	将src(dst) header存放到前缀为HOBJECT_TO_SEQ，key = map_header_key(src)或者key = map_header_key(dst)的记录中。
5)	以上步骤实现了omap的0-copy，但对于xattr则直接从parent复制到src和dst中，完成之后在从parent中删除xattr。
```c++
int DBObjectMap::clone(const ghobject_t &oid,
		       const ghobject_t &target,
		       const SequencerPosition *spos)
{
  … 
  // parent被扔到以sys_prefix(parent)为前缀的记录中，但是并没有从HOBJECT_TO_SEQ中删除，… 
  Header parent = lookup_map_header(*lsource, oid);
  if (!parent)
    return db->submit_transaction(t);

  Header source = generate_new_header(oid, parent);
  Header destination = generate_new_header(target, parent);
  if (spos)
    destination->spos = *spos;

  parent->num_children = 2;
  set_header(parent, t);
  set_map_header(*lsource, oid, *source, t);
  set_map_header(*ltarget, target, *destination, t);

  map<string, bufferlist> to_set;
  KeyValueDB::Iterator xattr_iter = db->get_iterator(xattr_prefix(parent));
  for (xattr_iter->seek_to_first();
       xattr_iter->valid();
       xattr_iter->next())
    to_set.insert(make_pair(xattr_iter->key(), xattr_iter->value()));
  t->set(xattr_prefix(source), to_set);
  t->set(xattr_prefix(destination), to_set);
  t->rmkeys_by_prefix(xattr_prefix(parent));
  return db->submit_transaction(t);
}
```
rm_key操作，这里展现了rm操作对于complete为前缀的区间记录表的依赖:
1)	先从以user_prefix(header)为前缀的表中删除to_clear[]中指定的kv对
2)	如果header拥有父header，则对to_clear[]中任意一个需要删除的kv对i：
a)	从parent中读取[i，i+20)区间内的连续并且没有被删除的kv对到to_write[]中
b)	相应的区间记录在new_complete[]表中，key为begin，value=end
3)	将to_write[]中记录的kv写入到user_prefix(header)为前缀的表中
4)	将新的区间和header现有的区间进行合并，并记录到以complete_prefix(header)为前缀的表中，这也就意味着header直接从parent中拷贝相应区间内的kv对，这些区间内的kv不再需要从parent中获取了
5)	如果header不再需要parent，则递减parent的引用计数，并进行清理
```c++
int DBObjectMap::rm_keys(const ghobject_t &oid,
			 const set<string> &to_clear,
			 const SequencerPosition *spos)
{
  MapHeaderLock hl(this, oid);
  Header header = lookup_map_header(hl, oid);
  KeyValueDB::Transaction t = db->get_transaction();
  if (check_spos(oid, header, spos))
    return 0;
  t->rmkeys(user_prefix(header), to_clear);
  if (!header->parent) {
    return db->submit_transaction(t);
  }

  // Copy up keys from parent around to_clear
  int keep_parent;
  {
    DBObjectMapIterator iter = _get_iterator(header);
    iter->seek_to_first();
    map<string, string> new_complete;
    map<string, bufferlist> to_write;
    for(set<string>::const_iterator i = to_clear.begin();
	i != to_clear.end();
      ) {
      unsigned copied = 0;
      iter->lower_bound(*i);
      ++i;
      if (!iter->valid())
		break;
      string begin = iter->key();
      if (!iter->on_parent())
		iter->next_parent();
      if (new_complete.size() && new_complete.rbegin()->second == begin) {
		begin = new_complete.rbegin()->first;
      }
      while (iter->valid() && copied < 20) {
		if (!to_clear.count(iter->key()))
	  		to_write[iter->key()].append(iter->value());
		if (i != to_clear.end() && *i <= iter->key()) {
		  ++i;
		  copied = 0;
		}
		iter->next_parent();
		copied++;
      }
      if (iter->valid()) {
		new_complete[begin] = iter->key();
      } else {
		new_complete[begin] = "";
		break;
      }
    }
    t->set(user_prefix(header), to_write);
    merge_new_complete(header, new_complete, iter, t);
    keep_parent = need_parent(iter);
    if (keep_parent < 0)
      return keep_parent;
  }
  if (!keep_parent) {
    copy_up_header(header, t);
    Header parent = lookup_parent(header);
    if (!parent)
      return -EINVAL;
    parent->num_children--;
    _clear(parent, t);
    header->parent = 0;
    set_map_header(hl, oid, *header, t);
    t->rmkeys_by_prefix(complete_prefix(header));
  }
  return db->submit_transaction(t);
}
```
对omap的遍历依赖于DBObjectStroe的迭代器ObjectMapIteratorImpl，而对xattr的遍历则依赖leveldb的迭代器IteratorImpl，虽然IteratorImpl继承自ObjectMapIteratorImpl，但除了接口相同外它们的作用和实现是完全不同的，IteratorImpl的用来在leveldb中特定的表中进行遍历，而ObjectMapIteratorImpl在IteratorImpl基础之上，实现了对分散在父子header中kv对的遍历。
几个重要的对象成员：
```c++
DBObjectMap *map;
ceph::shared_ptr<DBObjectMapIteratorImpl> parent_iter; // 指向父header的迭代器
KeyValueDB::Iterator key_iter;	// 指向当前header表的leveldb迭代器指针
KeyValueDB::Iterator complete_iter;	// 指向区间表的leveldb迭代器指针
ceph::shared_ptr<ObjectMapIteratorImpl> cur_iter; // 指向key_iter和parent_iter中的一个，标识正待访问的kv对

// 初始化方法，对各个迭代器进行初始化
int DBObjectMap::DBObjectMapIteratorImpl::init()
{
 … 
  if (header->parent) {
    Header parent = map->lookup_parent(header);
    parent_iter.reset(new DBObjectMapIteratorImpl(map, parent));
  }
  key_iter = map->db->get_iterator(map->user_prefix(header));
  complete_iter = map->db->get_iterator(map->complete_prefix(header));
  cur_iter = key_iter;
  ready = true;
  return 0;
}

// 将父子迭代器都指向第一个kv对，然后调用adjust()方法对cur_iter进行调整…
int DBObjectMap::DBObjectMapIteratorImpl::seek_to_first()
{
  init();
  if (parent_iter) {
    r = parent_iter->seek_to_first();
  }
  r = key_iter->seek_to_first();
  return adjust();
}

// 将cur_iter指向一下kv对，并使用adjust()进行对cur_iter进行调整
int DBObjectMap::DBObjectMapIteratorImpl::next()
{
  cur_iter->next();
  return adjust();
}

// 对cur_iter和parent_iter进行调整：
// 1) parent_iter如果落在了complete区间表的某个区间内[begin, end)，则将parent_iter设置为end，因为该区间内的kv对都存放在key_iter中
// 2) parent_iter指向key和key_iter指向的key相同，则以key_iter为主，因为修改操作将拥有最新值得kv对放在当前header的表中
// 接着调用valid_parent()方法，对比parent_iter拥有的kv对是否比key_iter指向的小，如果是，则cur_iter指向parent_iter，否则为key_iter
// 该成员方法表述了DBObjectMap的所有内涵…
int DBObjectMap::DBObjectMapIteratorImpl::adjust()
{
  string begin, end;
  while (parent_iter && parent_iter->valid()) {
    if (in_complete_region(parent_iter->key(), &begin, &end)) {
      if (end.size() == 0) {
		parent_iter->seek_to_last();
		if (parent_iter->valid())
		  parent_iter->next();
      } else
		parent_iter->lower_bound(end);
    } else if (key_iter->valid() && key_iter->key() == parent_iter->key()) {
      parent_iter->next();
    } else {
      break;
    }
  }
  if (valid_parent()) {
    cur_iter = parent_iter;
  } else if (key_iter->valid()) {
    cur_iter = key_iter;
  } else {
    invalid = true;
  }
  assert(invalid || cur_iter->valid());
  return 0;
}
```
