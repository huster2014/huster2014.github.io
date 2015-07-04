---
layout: post
title: ceph osd中omap的实现
---

FileStore作为ObjectStore最终的实现之一，封装了OSD底层所有的I/O操作。FileStore除了提供读写Object的接口外，还提供了访问Object的属性(xattr)和omap的接口。Xattr和omap都是key-value对的集合，它们的主要区别体现在实现上，xattr借助于文件的xattr属性实现存取，而omap则借助DBObjectMap将kv对存储在leveldb等数据库中。<br>
由于文件的xattr属性在长度上有限制，xattr“溢出”的部分也会被存放在DBObjectMap中，所以从用户的角度来看，xattr适合于存取数量较少的kv对，而omap适合于存取数量不限的kv对。下面重点探究DBObjectMap在实现上的技巧。<br>

## 接口
DBObjectMap继承自ObjectMap，ObjectMap则定义了对外的公共接口(由于ObjectMap主要是负责存储对象的omap的，因此下面的讨论将只限于omap)：<br>
1、	插入，更新和删除关联于某对象的kv对；<br>
2、	对关联于某对象的kv对集合进行克隆，使得其它对象拥有完全一样的kv对集合；<br>
3、	返回一个迭代器，用于遍历关联于某个对象的所有的kv对。<br>

## 实现
DBObjectMap为每个对象都分配了一个序列号，当向leveldb中插入与某对象关联的kv对时就会以该序列号生成相应的前缀，假设某个对象的序列号为seq，则与该对象关联的omap的kv对的前缀就为：<br>
Prefix = USER\_PREFIX + header\_key(seq) + USER\_PREFIX <br>
然后通过leveldb的set(prefix，key，value)接口就能将相应的记录插入到leveldb中了。你估计也猜到了，xattr和omap的区别也在于前缀的不同，xattr的kv对的前缀为：<br>
Prefix = USER\_PREFIX + header\_key(header->seq) + XATTR\_PREFIX <br>

### 克隆操作
DBObjectMap之所以复杂，主要在于它实现了kv对的0-copy克隆。前面说过omap适用于存储kv对数量较多的场景，因此实现omap的0-copy克隆就很有诱惑力。那具体是如何实现的呢？答案同样还是在kv对的前缀上。<br>
![](/images/omap/clone.png)<br>
上图是执行clone(src, dst)操作的示意图。为了实现0-copy克隆，每个对象都会对应一个header结构体，它记录了分配给对象的序列号以及克隆操作所产生的父子关系。在上图中，src对象原来的序列号为seq1，执行克隆操作时，DBObjectMap就会为src和dst对象都分配一个新的序列号和新的header结构体，原来的header结构体将作为它们的parent header。<br>
当访问与dst对象相关联的kv对时，除了通过前缀prefix(seq3)在leveldb中检索kv对外，还能通过前缀prefix(seq1)检索因为克隆操作而与src对象共享的kv对。<br>
你可能已经注意到了，以prefix(seq1)为前缀的kv对在克隆之后就成为只读的了，因为被多个对象共享，直接修改这些kv对就相当于修改了所有对象的kv映射，这不是DBObjectMap所希望的。所以，当修改某对象已有的kv对时，必须以它当前的序列号作为前缀向leveldb中插入新的kv对。如上图所示，克隆完成之后，修改关联于dst对象的键值为 key1的kv对，将在leveldb中插入以Prefix(seq3)为前缀的新的记录，此时在leveldb中就会出现两个归属于dst对象的且键值相同的kv记录，即\<prefix(seq1)，key1， value1\>和\<prefix(seq3)， key1，value1\>，在检索kv对时，直接忽略前者就行了。<br>

### 删除操
前面已经介绍了如何在克隆成功之后添加或者更新对象的kv对，那怎样删除对象的kv对呢？是否能够直接将相应的kv对删除？例如，假设要删除上图中dst对象键值为key1的记录，如果直接删除\<prefix(seq3)， key1，value1\>，那么在下次检索键值为key1的kv对的时候就会返\回<prefix(seq1)，key1， value1\>，但实际上应该要返回空值。考虑到kv对\<prefix(seq1)，key1， value1\>既不能删除又不能修改，那么可以采取如下可能的措施：<br>
* 将\<prefix(seq3)， key1，value1\>更新为\<prefix(seq3)， key1，None\>，借助空值None来表示该kv对已经被删除；<br>
* 添加一个数据结构，明确指出dst对象的键值为key1的kv对不存在于parent header中。<br>
第一种方法相当于为对应的kv对创建了一个DeadStone，将删除操作转化成了插入操作，这是一个很广泛采用的技巧。但是DBObjectMap采用了第二张方法，请不要问我为什么，它就是这么干的！当然，直接采用第二种方法必然很没有效率，那么DBObjectMap具体是怎么做的呢？<br>
DBObjectMap为每个header结构体都维护一个区间表，每个形表项都代表了一个区间，如[start，end)，它表示在该区间内的任何kv对都不存在于parent header中。当需要删除某个对象的键值为key的kv对时，DBObjectMap就会从parent header中读取键值为key~key+20的kv对，然后以前缀为prefix(seq3)重新插入到leveldb中，然后向区间表中插入一个区间项[key，key+20)，暗示用户该访问区间内的kv对时直接无视parent header就好了… 仔细分析，你会发现这个过程其实是一个变相的copy on write技巧，这也是一个很常用的技巧。<br>
![](/images/omap/rm.png)
上图就是删除dst对象键值为key1的kv对的过程。<br>

### 迭代器
DBObjectMap的迭代器很有意思，有点递归的味道。下图是一个是执行了clone(obj1，obj2)，…，clone(obj2，obj3)，…，clone(obj3，obj4)之后，header结构体所构成的父子关系。此时用户要求返回obj4一个的迭代器，用来遍历与obj4关联的所有的kv对，你该怎么办?<br>
![](/images/omap/iterator.png)<br>
迭代器不仅需要遍历当前header(序列号为7)对应的kv对，而且还要遍历它的父亲headers(即序列号为seq1，seq3和seq5的header) 的kv对。迭代器的实现伪代码如下(为简单起见，假设不存在kv删除操作)：

```cpp
	Class iterator {
		Virtual void next();
		Virtual void *value()
	};
	Class leveldb_iterator::iterator {  // 用来遍历具有prefix(seq)的kv对
		leveldb_iterator(seq);
		void next();
		void *value()
	}
	Class Header_Iterator::iterator // 用来遍历关联与某个对象的kv对
{
		Header_Iterator *parent; // 指向父迭代器
		Leveldb_iterator *key_iter; // 执行leveldb的迭代器
		Iterator *cur; // 等于parent或者key_iterator，该迭代器指向正在被访问的kv对

		Header_iterator(header) {
			If (header.parent) // 递归构造父亲header的迭代器
				Parent = new parent(header.parent)
			Key_iter = new Leveldb::iterator(header.seq)
			If (parent && parent->value() < key_iter->value())
				Cur = parent;
			Else
				Cur = key_iter;
		}
		next() {
			cur->next(); // 递归调用迭代器的next()方法
			if(parent && key_iter->value() > parent->value()) 
				cur = parent;
			else
				cur = key_iter;
		}
		void *Value() {
			If (cur->valid()) // 递归调用迭代器的value方法
				return cur->value(); 
		}
	}
```
你是已经看出了这其中隐含的递归思想呢！！
