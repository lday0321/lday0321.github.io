---
title: B-Tree数据结构总结
date: 2018-02-21 14:04:14
tags: 
	- ds_algorithm
	- b-tree
	- cpp
---

对B树的理解一直停留在概念阶段，一直也想了解一些细节。趁着过年这段时间，把《算法导论》关于B树的章节（第18章）整个过了一遍。结合书上的说明，自己实现了一把，留为总结。以下文字内容多摘自《算法导论》，伪代码部分根据我自己的理解添加了注释。

<!-- more -->

# B树的定义

一棵B树T是具有如下性质的有根树(根为T.root)

1. 每个结点x有如下属性
 * x.n，表示当前存储在结点x中的关键字个数。
 * x.n的各个关键字本身：x.key<sub>1</sub>, x.key<sub>2</sub>, ... 以非降序存放，使得 x.key<sub>1</sub> &le; x.key<sub>2</sub> &le; ...
 * x.leaf，是一个布尔值，如果x是叶子结点，则为TRUE, 如果x为内部结点，则为FALSE。

2. 每个内部结点x还包含x.n+1个指向其孩子的指针x.c<sub>1</sub>, x.c<sub>2</sub>, ... 。 叶子结点没有孩子结点，所以他的c<sub>i</sub>属性没有定义。

3. 关键字x.key<sub>i</sub>对存储在各子树中的关键字进行分割：如果k<sub>i</sub>为任意一个存储在以x.c<sub>i</sub>为根的子树中的关键字，那么：
	k<sub>1</sub> &le; x.key<sub>1</sub> &le; k<sub>2</sub> x.key<sub>2</sub> &le; ... &le; x.key<sub>x.n</sub> &le; k<sub>x.n+1</sub>

4. 每个叶子结点具有相同的深度，即树的高度h

5. 每个结点所包含的的关键字个数有上界和下界。用一个被称作**B树的最小度数(minimum degree)**的估计整数t(t &ge; 2)来表示这些界：
 * 除了根结点以外的每个结点必须<font color="#FF0000">**至少有t-1个关键字**</font>。因此，除了根节点以外的每个内部结点至少有t个孩子。如果树非空，根结点至少有一个关键字。
 * 每个结点<font color="#FF0000">**最多包含2t-1个关键字**</font>。因此，一个内部节点至多可有2t个孩子。当一个结点恰好有2t-1个关键字时，称该结点是**满的(full)**。

二分搜索树的扩张往往是叶子节点的向下延伸，收缩往往是从叶子节点向上收缩。但是B树不太一样，他的扩张是根结点的高度增加，相反的，收缩则是根结点的高度降低。

t=2的B树是最简单的。每个内部结点有2个、3个或者4个孩子，即一棵2-3-4树。然而在实际中，t的值越大，B树的高度就越小。

![](http://og43lpuu1.bkt.clouddn.com/b_tree_summary/img/01_B-Tree_sample.png)

BTreeNode结构定义，代码实现:
```cpp
class BTree;

// A BTree Node
class BTreeNode {
    int *keys;              // 当前结点的关键字列表
    int t;                  // B-Tree的最小度(minimum degree)
    BTreeNode **children;   // 子结点列表
    int n;                  // 当前关键字数量（我把他称作当前结点的度）
    bool leaf;              // 是否为叶子结点

public:
    BTreeNode(int _t, bool _leaf);

    // 基于当前结点完成子树遍历
    void traverse();

    // 基于当前结点搜索关键字k，如果找不到则返回nullptr
    BTreeNode *search(int k);

    // 辅助函数，用于将当前结点的第i个子结点分裂开来
    // y结点是一个满结点
    void splitChild(int i, BTreeNode *y);

    // 辅助函数，用于向当前结点中插入一个关键字
    // 调用该函数的前提假设是，当前结点是非满结点
    void insertNonFull(int k);

    // 辅助函数，在当前结点上找到第一个大于等于目标关键字k的索引位置
    int findKey(int k);

    /*********************************************/
    /* 处理删除关键字的场景1                         */
    /*********************************************/
    // 从当前结点（叶子结点）上移除第idx位的关键字(k),(1)
    void removeFromLeaf(int idx);


    /*********************************************/
    /* 处理删除关键字的场景2                         */
    /*********************************************/
    // 在当前结点上，获得第idx位关键字的前驱关键字(k'),(2.a)
    int getPred(int idx);

    // 在当前结点上，获取第idx位关键爱你字的后继关键字(k'),(2.b)
    int getSucc(int idx);

    // 把第idx个子结点和第idx+1个子结点，以及第idx个关键字合并
    // 这个合并的前提应该是第idx个关键字k的左右两个子结点都只包含t-1的度(2.c/3.b)
    void merge(int idx);

    // 从当前结点（非叶子结点）上移除第idx位的关键字(k), (2)
    void removeFromNonLeaf(int idx);


    /*********************************************/
    /* 处理删除关键字的场景3                         */
    /*********************************************/
    // 关键字在第idx个子结点上，且该子结点只有t-1的度
    // 但是该子结点的左兄弟有至少t的度，则从他的左兄弟(prev)
    // 借一个关键字（左兄弟上的最大关键字）提升到当前结点
    // 把当前结点上介于左兄弟和目标子结点之间的关键字下降到目标子结点(3.a)
    void borrowFromPrev(int idx);

    // 关键字在第idx个子结点上，且该子结点只有t-1的度
    // 但是该子结点的右兄弟有至少t的度，则从他的右兄弟(next)
    // 借一个关键字（右兄弟上的最小关键字）提升到当前结点
    // 把当前结点上介于目标子结点和右兄弟之间的关键字下降到目标子结点(3.a)
    void borrowFromNext(int idx);

    // 目标关键字不在当前结点上，idx是我们需要进一步第归的目标关键字
    // 可能在的第idx个子结点，通过fill保证children[idx]的度不低于t
    // 其处理了两中场景：3.a和3.b
    // 3.a: 虽然children[idx]只有t-1的度，但是他有兄弟是t及以上的
    //      通过旋转，完成子结点的度提升
    // 3.b: children和他兄弟都只有t-1的度，就通过merge来完成子结点度的提升
    // 返回fill之后的目标分支
    int fill(int idx);



    // 基于当前结点删除关键字k
    bool remove(int k);


    // 让BTree成为BTreeNode的友元
    friend class BTree;
};

BTreeNode::BTreeNode(int _t, bool _leaf) {
    t = _t;
    leaf = _leaf;

    // 根据t分配关键字的最大空间，关键字按升序偏序组织
    keys = new int[2*t-1];

    // 根据t分配子结点的最大空间
    children = new BTreeNode *[2*t];

    // 初始阶段当前结点的关键字数量为0
    n = 0;
}
```

BTree结构定义，代码实现：
```cpp
class BTreeNode;

// A BTree
class BTree {
    BTreeNode *root;    // 根结点
    int t;              // 最小度

public:
    BTree(int _t);

    // 遍历整个B-Tree
    void traverse();

    // 在B-Tree中查找某个关键字
    BTreeNode *search(int k);

    // 向B-Tree中插入关键字
    void insert(int k);

    bool remove(int k);
};

BTree::BTree(int _t) {
    root = nullptr;
    t = _t;
}
```



# B树的基本操作

对于磁盘读写的约定：
1. B树的根结点始终在主存中，这样无需对根做DISK-READ操作；然而，当根结点被修改后需要对根结点做一次DISK-WRITE操作。
2. 任何被当做参数的结点在被传递之前，都要对他们先做一次DISK-READ操作。

## 搜索B树

B-TREE-SEARCH是定义在二叉搜索树上的TREE-SEARCH过程的一个直接扩展。他的输入是一个指向某子树根结点x的指针，以及要在该子树中搜索的一个关键字k。因此顶层调用的形式为B-TREE-SEARCH(T.root, k)。如果k在B树中，那么B-TREE-SEARCH返回的是由节点y和使得y.key<sub>i</sub>=k的下标i(i表示的是结点y上的第i个slot)组成的有序对(y, i)；否则，过程返回NIL。

伪代码：
```
B-TREE-SEARCH(x, k):
i = 1
while i <= x.n and k > x.key_i  // 在结点x这一层，遍历所有排好序的slot，找到k应该在的恰当位置
    i = i + 1
if i <= x.n and k == x.key_i // 相当，则意味着正好找到了！
    return (x, i)
elsif x.leaf // 如果已经是叶子结点了， 说明已经到底了，找不到，返回NIL
    return NIL
else DISK-READ(x, c_i) // 此时的x.key_i是大于k的第一个结点，因此应该沿着x.key_i的左手边，也就是c_i这个子结点，继续递归下去
    return B-TREE-SEARCH(x.c_i, k)
```

代码实现：
```cpp
// 查找指定关键字
BTreeNode *BTreeNode::search(int k) {

    // 在当前结点上找到大于等于k的关键字(keys是按照升序偏序，<=... <=... 来组织)
    int i = 0;
    while( i < n && k > keys[i])
        i++;

    // 如果正好找到， 则返回
    if (i < n && keys[i] == k)
        return this;

    // 如果当前结点已经是叶子结点了， 但还是没找到，则返回nullptr
    if (leaf == true)
        return nullptr;

    // 其他情况下， 则基于找到的大于k的最小子结点进行第归
    return children[i]->search(k);
}

// 在B-Tree上查找
BTreeNode *BTree::search(int k) {
    if (root != nullptr) {
        return root->search(k);
    } else {
        return nullptr;
    }
}
```

遍历B树的代码实现：
```cpp
// 遍历结点
void BTreeNode::traverse() {
    // 一个结点包含了n个关键字，同时也有n+1个子结点
    // 遍历的时候， 我们首先输出关键字左侧的子结点，然后输出当前关键字，最后再输出关键字右侧的子结点
    // 这样的遍历，才会是有序的
    int i;
    for (i = 0; i < n; i++) {
        // 如果不是叶子结点，那么就存在子结点
        if (leaf == false) {
            children[i]->traverse();
        }

        // 在遍历完左侧子结点之后，再遍历当前关键字
        cout << " " << keys[i];
    }

    // 上面遍历是children[0]->keys[0]->children[1]->keys[1]->...->children[n-1]->keys[n-1]
    // 还少了最后一个child: children[n]
    if (leaf == false) {
        children[i]->traverse();
    }
}

// 遍历整个B-Tree
void BTree::traverse() {
    if (root != nullptr)
        root->traverse();
}
```

## 创建一棵空的B树

为创建一个B树T，先用B-TREE-CREATE来创建一个空的根结点，后续，则可以通过B-TREE-INSERT向T中添加新的关键字。创建树的过程中需要用到辅助过程ALLOCATE-NODE()，他在O(1)时间内为一个新结点分配一个磁盘页。我们可以假定由ALLOCATE-NODE所创建的节点并不需要DISK-READ，因为磁盘上还没有关于该结点的有用信息。

伪代码:
```
B-TREE-CREATE(T)
    x = ALLOCATE-NODE()
    x.leaf = TRUE
    x.n = 0
    DISK-WRITE(x)
    T.root = x
```

该部分代码实现，在`BTree::insert`接口中一并实现。

## 向B树中插入一个关键字

由于不能将关键字插入一个满的叶结点，故引入一个操作，将一个满的结点y(**2t-1**个关键字)按其**中间关键字**(median key): y.key<sub>t</sub>**分裂(split)**为两个各含t-1个关键字的结点。中间关键字被提升到y的父结点，以标识两棵树的划分点。但如果y的父结点也是满的，就必须在插入行的关键字之前将其分裂，最终满结点的分裂会沿着树向上传播。

与一棵二叉搜索树一样，可以在从树根到椰子这个单程乡下过程中将一个新的关键字插入B树中。为了做到这一点，我们并不是等到找出插入过程中实际要分裂的满结点时才做分裂。相反，当沿着树往下查找新的关键字所属位置时，就分裂沿途遇到的每个满结点（包括叶节点本身）因此，每当要分裂一个满结点y时，就能确保他的父结点不是满的

### 分裂B树中的结点

B-TREE-SPLIT-CHILD的输入是一个非满的内部结点x(假定在主存中)和一个使x.c<sub>i</sub>(也假定在主存中)为x的满子结点的下标i。（x.c<sub>i</sub>是x下面的一个子结点，现在的装填是：x.c<sub>i</sub>已经满了，得把他分裂掉）。该过程把这个子结点分裂成2个，并调整x,使x包好多出来的孩子。要分裂一个满的根，首先要让根成为一个新的空节点的孩子，这样才能使用B-TREE-SPLIT-CHILD，树的高度因此增加1；分裂是树长高的唯一途径。

分裂动作，还没有开始插入， 他仅仅是单纯的将一个满的(2t-1)节点，变成两个非满节点(t-1)，同时，将中间key，提升到父结点中。 2t-1 = 2*(t-1) + 1

下图显示了这个过程。分裂一个t=4的节点（最大：2t-1=7）。结点y=x.c<sub>i</sub>按照其中间关键字S进行分为两个结点y和z，y中的哪些大于中间关键字的关键字都置于新的节点z中，他成为x的一个新的孩子，y的中间关键字S被提升到y的父结点x中。

![](http://og43lpuu1.bkt.clouddn.com/b_tree_summary/img/02_B-Tree-split-child.png)

伪代码：
```
B-TREE-SPLIT-CHILD(x, i)
    z = ALLOCATE-NODE()  // 准备一个空的节点，用于存放分裂出来的，大于S的部分
    y = x.c_i // 取出当前已满的x子结点y
    z.leaf = y.leaf // 新结点z的leaf标签和原节点y保持一致
    z.n = t - 1  // 分裂后， z的度应该只到t-1
    
    // 接着搬key
    for j = 1 to t -1  //注意这里是到t-1
        z.key_j = y.key_(j+t)  // 循环遍历，将t右侧一半的key，从y搬到z
    
    // 接着搬child
    if not y.leaf // 如果y节点不是叶子结点，则将child部分，t右侧一半也搬到z
        for j = 1 to t //注意这里是到t
            z.c_j = y.c_(j+t)
            
    y.n = t - 1 // 分裂后，y的度由满时的2t-1缩小为t-1
    
    // 反着为新创建的结点z，流出一个在x上的child位置
    for j = x.n+1 downto i+1  // 这个是反着copy，将i+1空出来
        x.c_j+1 = x.c_j  // 顺位往后移一位，将i+1空出来
    x.c_i+1 = z // 把z放在i后面一个位置，也就是紧挨着y的那个
    
    // 同理，还需要将key也挪一个位置，用于放S(y.key_t)
    for j = x.n downto i 
        x.key_(j+1) = x.key_j
    x.key_i = y.key_t  // 把S放在x中的新的位置
    
    // 因为S被提到x中，x的度提升了1
    x.n = x.n + 1
    
    // 将x, y, z分别写回磁盘
    DISK-WRITE(y)
    DISK-WRITE(z)
    DISK-WRITE(x)
```

代码实现：
```cpp
// 将当前结点的第i个子结点y分裂开来
// y结点是一个满结点
void BTreeNode::splitChild(int i, BTreeNode *y) {

    // 分配一个新的结点，用于存放分裂出来的(t-1)个关键字
    // y中被提升的中间key，在t位置，假设为s
    BTreeNode *z = new BTreeNode(t, y->leaf);
    z->n = t - 1;

    // 循环遍历，将y右侧的一半key，从y搬到z
    for (int j = 0; j < t-1; j++) {
        z->keys[j] = y->keys[j+t];
    }

    // 接着搬child
    if (y->leaf == false) {
        for (int j = 0; j < t; j++) // 注意，这里是到t
            z->children[j] = y->children[j+t];
    }

    // 分裂后，y的度数由满时的2t-1缩小为t-1
    y->n = t - 1;

    // 新创建的结点z需要加入到当前结点的子结点序列中
    // 需要将children结点往后移动一格
    for (int j = n; j >= i+1; j--) {
        children[j+1] = children[j];
    }

    // 将z放在i后面一个位置
    children[i+1] = z;

    // 同理，还需要将当前结点的key也依次向后挪一个位置
    for (int j = n-1; j>= i; j--) {
        keys[j+1] = keys[j];
    }

    // 把从y中分裂出来的中间的那个key，放到该位置
    keys[i] = y->keys[t-1];

    // 因为s被提升到了x中，x的度提升1
    n = n + 1;
}
```

### 以沿树单程下行方式向B树插入关键字

B-TREE-INSERT利用B-TREE-SPLIT-CHILD来保证递归始终不会降至一个满结点上。

伪代码：
```
B-TREE-INSERT(T, k)
    r = T.root
    if r.n == 2t - 1  // 在一个已满的根节点上插入节点，得提升高度了
        s = ALLOCATE-NODE()
        T.root = s    // 已这个空节点作为新的根
        s.leaf = FALSE
        s.n = 0
        s.c1 = r  // 这个心的根只有一个子节点（原来那个满的根节点，准备分裂了）
        B-TREE-SPLIT-CHILD(s, 1) // 要分裂的就是s上，第1位的子节点，原来的根结点r
        
        // 分裂完成之后，再在新的，不满的树上执行插入
        B-TREE-INSERT-NOTFULL(s, k)
    else 
        // 如果直接就是一颗不满的根树，则直接调用NOTFULL插入
        B-TREE-INSERT-NOTFULL(r, k)
```

代码实现：
```cpp
// 向B-Tree中插入关键字
void BTree::insert(int k) {
    // 如果当前B-Tree为空，则需要创建一个结点
    if (root == nullptr) {
        root = new BTreeNode(t, true);
        root->keys[0] = k;  // 该结点只有一个key，就是这个新插入的key
        root->n = 1;
    } else {
        // 判断当前根结点是否已满
        if (root->n == 2*t - 1) {
            // 如果满了，则需要提升高度
            // 分配一个新的结点，作为新的root
            BTreeNode *s = new BTreeNode(t, false);

            // 这个新的根结点只有一个子结点，就是老的已满的根结点
            s->children[0] = root;

            // 需要对s的已满子结点进行分裂
            s->splitChild(0, root);

            // 分裂完之后，s是一个包含两个子结点的非满的结点
            // 此时，直接将k插入到s结点中
            s->insertNonFull(k);

            // 最后，将s作为新的root
            root = s;
        } else {
            // 如果不满，则直接调用NonFull插入
            root->insertNonFull(k);
        }
    }
}
```



![](http://og43lpuu1.bkt.clouddn.com/b_tree_summary/img/03_split_full_root.png)

如上图所示，分裂t=4(2t-1=7)的根。根结点一分为二，并创建了一个新结点s。在执行B-TREE-SPLIT-CHILD(s, 1)之后，新的根包含了r的中间关键字，且以r的两半作为孩子。当根被分裂后，B树的高度增加1。

通过调用B-TREE-INSERT-NOTFULL完成将关键字k插入以非满根结点x为根的树中。B-TREE-INSERT-NONFULL在需要时沿树向下递归，在必要时，通过调用B-TREE-SPLIT-CHILD来保证任何时刻他所递归处理的节点都是非满的。

伪代码：
```
B-TREE-INSERT-NONFULL(x, k)
    i = x.n
    if x.leaf
        // 如果是叶子节点，只需要把k放到key区的合适位置就可以了
        while i >= 1 and k < x.key_i  // i的初始位置是最后尾巴上，是整体往右移一格
            x.key_(i+1) = x.key_i  // 所以最后一个slot，变到了n+1
            i = i - 1
    
        // 此时，i的位置就是k应该放的位置的前一个位置，因为，此时k >= x.key_i
        // k应该放在x.key_i的后面，也就是i+1
        x.key_(i+1) = k
        x. n = x.n + 1 // x的度因为插入了k，而需要增加1
        DISK-WRITE(x) // 对于叶子节点，处理完key区的任务，就完成了
    else
        // 此时应该是要搬key，同时搬child
        while i >= 1 and k < x.key_i //同样还是要找到key的合理位置i+1
            i = i - 1
        i = i + 1 // 此时的i就是key应该到的位置
        
        // 这个时候，实际上是需要插入到i位置的child中
        // 那显然，得看看child_i的度是不是已经满了
        // 如果满了，为了确保一切OK，得先对child_i做split
        if x.c_i.n == 2t -1 
            B-TREE-SPLIT-CHILD(x, i)
            
            // split之后，x中会多插入一个key进来
            // 这个key可能正好插在k要去的节点i的位置
            // 如果正好在这里，则需要比较一下，新的节点插入之后
            // k是否大于这个新的加入的节点，如果是，则需要往后挪一个
            // 其实也就是插入split之前的x的第i个子节点c_i中去
            if k > x.key_i 
                i = i + 1
                
        //执行完上述操作之后， 我们已经可以确定：
        // 1. i的位置，即x.c_i就是k需要去的子树
        // 2. x.c_i是非满节点，可以继续调用NONFULL
        B-TREE-INSERT-NONFULL(x.c_i, k)
```

代码实现：
```cpp
// 辅助函数，用于向当前结点中插入一个关键字
// 调用该函数的前提假设是，当前结点是非满结点
void BTreeNode::insertNonFull(int k) {
    int i = n-1; // 记录最右关键字位置

    // 判断是否为叶子结点
    if (leaf) {
        // 如果是叶子结点，则直接找到一个合适的位置，将k插进去
        // 从最后面开始，如果大于k，都向后移
        while (i >=0 && keys[i] > k) {
            keys[i+1] = keys[i];
            --i;
        }

        // 上面循环遍历完，i的位置，是可以插入k的前一个位置
        keys[i+1] = k;
        // 度数相应加1
        n += 1;

    } else {
        // 如果不是叶子结点，则需要搬动key和children
        // 同样，还是得先找到key的合理位置
        while (i >= 0 && keys[i] > k) {
            --i;
        }

        // i+1才是key应当到的子结点路径
        i = i+1;
        if (children[i]->n == 2*t-1) {
            // 如果子结点已经满了，则需要切分
            splitChild(i, children[i]);

            // split之后，当前结点中会多插入一个key进来
            // 这个key可能正好插在k要去的节点i的位置
            // 如果正好在这里，则需要比较一下，新的节点插入之后
            // k是否大于这个新的加入的节点，如果是，则需要往后挪一个
            // 其实也就是插入到split之前的目标子结点去
            if (keys[i] < k) {
                ++i;
            }
        }

        //执行完上述操作之后， 我们已经可以确定：
        // 1. i的位置，即children[i]就是k需要去的子树
        // 2. children[i]是非满节点，可以继续调用NONFULL
        children[i]->insertNonFull(k);
    }
}
```

如下图所示，是一个key的插入过程：

![](http://og43lpuu1.bkt.clouddn.com/b_tree_summary/img/04_key_insert.png)

向这个B树中插入关键字。这颗B树的最小度数t为3，那最多包含2t-1=5个关键字。在插入过程中被修改的节点由浅阴影标记：

* (a). B树的初始状态
* (b). 插入B后的结果，这是一个对叶子节点的简单插入
* (c). 接着将Q插入。节点RSTUV被分裂为2个节点，分别包好RS和UV，同时父结点由原来的GMPX变成GMPTX，最终Q被出入到左子节点RS上，变成QRS。
* (d). 接着插入L。由于根结点现在已经满了，首先要对根结点进行分裂，树高度+1。原来的根结点GMPTX被split成：P挂GM和TX，然后L被插入到JK节点中，变成JKL
* (e). 最后插入F, ABCDE这个节点满了，先进行分裂，分裂之后是：CGM挂AB和DE，最终F被插入到DE中，变成DEF

## 从B树中删除关键字

B树上的删除操作与插入操作类似，我们可以从任意一个节点中删除一个关键字，不仅仅是叶子结点才可以这么做。当从一个内部节点删除一个关键字时，为防止这一删除操作违反B树的性质(t-1 &le; 剩余关键字个数 &le; 2t-1)，必须保证一个结点不会在删除之后变得太小（根结点除外，他允许有比最少关键字t-1还少的关键字个数）。与插入类似， 当在要删除的关键字的路径上的结点(非根结点)，有结点只有最少(t-1)个关键字时，则需要考虑向上回溯了。


B-TREE-DELETE从以x为根的子树中删除关键字k。设计时，调用该函数时，必须保证无论何时，结点x递归调用自身时，x中关键字个数至少为度数t。<font color="#0000FF">注意到，这个条件要比通常B树中的最少关键字个数多1</font>，这个要求，使得在递归下降进行调用时，有时候为保证子结点的度数满足t的要求，必须在递归下降之前，将当前结点的一个关键字，移到子结点上。有了这个要求，我们就可以保证在一趟下降过程中， 我们就可以将一个关键字从树中删除，无需再向上回溯。

现在我们来介绍删除操作如何工作：

1. 如果关键字k在节点x中，且x是叶结点，则从x中删除
2. 如果关键字k在节点x中，且x是内部节点，则做一下操作：
 a. 如果结点x中，前于k的子结点为y，y至少包含了t个关键字，则找出以y为根的子树中的前驱k'。递归的删除k'，并在x中用k'代替k。（找出k'并删除他，可在沿树下降的单次递归调用中完成）
 
 b.对称的，如果y有少于t个关键字，则检查x中后语k的子节点z。如果z至少有t个关键字，则找出k在以z为子树中的后继k'。递归的删除k'，并在x中用k'代替k。（同样，超出k'并删除他，可在沿树下降的单词递归调用中完成）
 
 c. 否则，如果y和z都只包含t-1个关键字，则将k和z的全部合并进y，这样x就失去了k和指向z的指针，并且y现在包含2t-1个关键字。然后释放z，并递归的从y中删除k。
 
3. 如果关键字k当前不在内部结点x中，则确定可能包含k的子树的根x.c<sub>i</sub>(如果k确实在树中的话)。如果x.c<sub>i</sub>只包含t-1个关键字，则必须先执行下面的3a或者3b来保证降至一个至少包含t个关键字的结点x.c<sub>i</sub>。然后，通过对x的合适的子节点x.c<sub>i</sub>进行递归而结束。

 a. 如果x.c<sub>i</sub>只包含有t-1个关键字，但是他的一个相邻的兄弟至少包含t个关键字，则将x中的某一个关键字降至x.c<sub>i</sub>中，将x.c<sub>i</sub>的相邻的左兄弟或者右兄弟的一个关键字提升至x，将该兄弟中相应的孩子指针移到x.c<sub>i</sub>中(这个过程类似于左旋或者右旋调整，如果找了左兄弟，则将左兄弟的最大关键字提升，将x上x.c<sub>i</sub>与目标子节点之间的关键字下降到x.c<sub>i</sub>；找右兄弟的做法类似)，这样就使得x.c<sub>i</sub>增加了一个额外的关键字。
 
 b. 如果x.c<sub>i</sub>以及x.c<sub>i</sub>的所有兄弟结点都只包含t-1个关键字，则将x.c<sub>i</sub>与一个兄弟结点合并，也就是将x的一个关键字移至新合并的节点，使之成为新结点的中间关键字。

如图所示，是关键字的删除过程示例：

![](http://og43lpuu1.bkt.clouddn.com/b_tree_summary/img/05_key_delete_part_01.png)
![](http://og43lpuu1.bkt.clouddn.com/b_tree_summary/img/06_key_delete_part_02.png)

这课B树的最小度数t=3，因此一个结点（非根结点）包含的关键字个数最少为2(t-1)。被修改的结点都以浅阴影标记。

(a). 初始状态
(b). 删除F。这是情况1：从一个叶节点中进行简单的删除
(c). 删除M。这是情况2a: M的前驱L提升并占据M的位置
(d). 删除G。这是情况2c: G下降以构成结点DEGJK，然后从这结点中删除G，这个结点是一个叶节点，因此回到情况1，将G中心的结点DEGJK中删除-->GEJK。
(e). 删除D。这个情况是3b：从根递归至结点CL，因为他仅有2个关键字，少于3(t)，同时CL的相邻子节点只有一个TX，他们两个都是2(t-1)。因此需要根据3b，将P下降，并与CL和TX合并，以构成CLPTX，(e')是在合并之后的新树，根结点被剔除，树的高度减1。然后将D从这个叶节点上删除，继续下降到节点DEJK上，该结点是叶结点，直接删除，得到EJK
(f). 删除B。这是情况3a：AB只包含2个关键字，但是他的兄弟节点EJK包含3个关键字，因此，是将x的关键字C落下来，同时，将EJK中的E提上去（相当于左旋），之后再将ABC中的B删除-->AC


删除关键字，针对各类场景的代码实现：
```cpp
// 辅助函数，在当前结点上找到第一个大于等于目标关键字k的索引位置
int BTreeNode::findKey(int k) {
    int idx = 0;
    while(idx<n && keys[idx] < k) {
        ++idx; // 如果目标k还是太大，则继续向后找
    }

    return idx;
}

// 从当前结点（叶子结点）上移除第idx位的关键字(k),(1)
void BTreeNode::removeFromLeaf(int idx) {
    // 直接将目标关键字移除（后面的关键字都往前移
    for (int i = idx+1; i < n; i++) {
        keys[i-1] = keys[i];
    }

    // 度数也减去1
    --n;

    return;
}


// 在当前结点上，获得第idx位关键字的前驱关键字(k'),(2.a)
int BTreeNode::getPred(int idx) {
    // 找到左子树上的最大key
    BTreeNode *cur=children[idx]; // children[idx]就是第idx位关键字的前驱

    // 因为B-Tree是平衡树，所以，如果当前结点不是叶子结点，则他必有最右子结点第n位
    while(!cur->leaf) {
        cur = cur->children[cur->n]; // 度是n，子结点数量是n+1
    }

    // 最后叶子结点的最右边的key，则为最大前驱
    return cur->keys[cur->n-1];
}

// 在当前结点上，获取第idx位关键爱你字的后继关键字(k'),(2.b)
int BTreeNode::getSucc(int idx) {
    // 找到右子树的最小key
    BTreeNode *cur=children[idx+1]; // children[idx+1]就是第idx位关键字的后继结点

    // 平衡树，不是也子结点，就必须有左子结点
    while(!cur->leaf) {
        cur = cur->children[0];
    }

    // 最左边的key，则为最小后继
    return cur->keys[0];
}

// 把第idx个子结点和第idx+1个子结点，以及第idx个关键字合并
// 这个合并的前提应该是第idx个关键字k的左右两个子结点都只包含t-1的度(2.c/3.b)
// 合并后的新结点在第idx位
void BTreeNode::merge(int idx) {
    /*
    //              [idx]
    //             /     \
    //        (idx)     (idx+1)
    */
    BTreeNode *child = children[idx];
    BTreeNode *sibling = children[idx+1];

    // child有t-1个key, sibling也有t-1个key
    // 目的是将sibling和目标关键字(keys[idx])合到child
    // [child.keys]-[keys[idx]]-[sibling.keys]
    // 目标关键字，在child上应该是在第t-1位
    child->keys[t-1] = keys[idx];

    // 接着把sibling的keys也copy过来
    for (int i = 0; i <sibling->n; ++i) {
        child->keys[t+i] = sibling->keys[i];
    }

    // copy完keys，还要copy子树部分
    if (!child->leaf) {
        for (int i = 0; i <= sibling->n; ++i) {
            child->children[t+i] = sibling->children[i];
        }
    }

    // 合并后的左子树，度数变为2t-1
    child->n += sibling->n + 1;

    // 现在已经把左子树，目标关键字，右子树合成了一个新的结点（左子树）
    // 接着需要将目标关键字在当前结点上移除
    // 首先移动keys部分
    for (int i = idx+1; i < n; ++i) {
        keys[i-1] = keys[i];
    }

    // 接着把子树也向左移动一格
    // idx位置的左子树不变，idx+1位置的右子树已经被合并调了
    // 移动是从idx+2移到idx+1开始
    for (int i = idx + 2; i <= n; ++i) {
        children[i-1] = children[i];
    }

    // 当前结点，由于去掉了一个目标结点，合并了两个子树到一个子树
    // 度数需要-1
    n--;

    // 合并完之后,右子树被释放
    delete(sibling);

    return;
}

// 从当前结点（非叶子结点）上移除第idx位的关键字(k), (2)
void BTreeNode::removeFromNonLeaf(int idx) {
    int k = keys[idx]; // 暂存目标关键字

    // 判断左子树是否满足>=t的度数条件
    if (children[idx]->n >= t) {

        // 从左子树找最大前驱
        int pred = getPred(idx);
        // 在左子树上，将pred(k')删除
        children[idx]->remove(pred);
        // 再将k替换为k'
        keys[idx] = pred;

    } else if (children[idx+1]->n >= t) {
        // 判断有子树是否满足>=t的度数条件

        // 从右子树找最小后继
        int succ = getSucc(idx);
        // 在右子树上，将succ(k')删除
        children[idx+1]->remove(succ);
        // 再将k替换为k'
        keys[idx] = succ;

    } else {
        // 如果左子树和右子树的度数都仅位t-1，则需要将左子树和右子树以及目标关键字左合并
        merge(idx);
        // 继而在合并的结点上进行第归删除
        children[idx]->remove(k);
    }
}

// 关键字在第idx个子结点上，且该子结点只有t-1的度
// 但是该子结点的左兄弟有至少t的度，则从他的左兄弟(prev)
// 借一个关键字（左兄弟上的最大关键字）提升到当前结点
// 把当前结点上介于左兄弟和目标子结点之间的关键字下降到目标子结点(3.a)
void BTreeNode::borrowFromPrev(int idx) {

    // 目标子树是第idx位，因此要从当前结点移到目标结点的关键字是第idx-1位的关键字
    /*
    //              [idx-1][idx]
    //             /      |      \
    //      (idx-1)      (idx)     (idx+1)
    */
    BTreeNode *child = children[idx];
    BTreeNode *prev = children[idx-1];

    // 首先是把child子树上的关键字，全部往后面挪一位，让出空间给准备从当前结点第idx-1位下降下来的关键字
    for (int i = child->n-1; i >= 0; --i) {
        child->keys[i+1] = child->keys[i];
    }

    // 如果是child不是叶子结点，则child的子树，也得往后挪一位
    if (!child->leaf) {
        for (int i = child->n; i >=0; --i){
            child->children[i+1] = child->children[i];
        }
    }

    // 挪好之后，把目标关键字移下来
    child->keys[0] = keys[idx-1];

    // 别忘了，这个时候，child的第0为的子树被往后移了一格，已经空了
    // 我们需要将prev的最后一个子树挂过来
    if (!child->leaf) {
        child->children[0] = prev->children[prev->n];
    }

    // 显然挂完之后，child的度多1
    child->n += 1;

    // 接着，prev的最后一个key(第n-1位)要上移到当前结点的idx-1的位置
    keys[idx-1] = prev->keys[prev->n-1];

    // prev子结点由于上移了一个关键字，度数减1
    // 度数减一之后，原来最左边的那个子树，就相当于抹掉了
    prev->n -= 1;

    return;
}


// 关键字在第idx个子结点上，且该子结点只有t-1的度
// 但是该子结点的右兄弟有至少t的度，则从他的右兄弟(next)
// 借一个关键字（右兄弟上的最小关键字）提升到当前结点
// 把当前结点上介于目标子结点和右兄弟之间的关键字下降到目标子结点(3.a)
void BTreeNode::borrowFromNext(int idx) {
    // 和borrowFromPrev的旋转方向正好相反
    // 目标子树是第idx位，因此要从当前结点移到目标结点的关键字是第idx位的关键字
    /*
    //              [idx-1][idx]
    //             /      |      \
    //     (idx-1)      (idx)     (idx+1)
    */
    BTreeNode *child = children[idx];
    BTreeNode *next = children[idx+1];

    // 把当前结点的第idx位的key移到目标子树child的keys中，放在最右侧
    child->keys[child->n] = keys[idx];

    // 同时，next的最左子树需要挂到child的最右侧
    if (!child->leaf) {
        child->children[(child->n)+1] = next->children[0];
    }

    // child子树的挪东已经完成，由于多增加了一个从当前结点落下来的key
    // 度数将增加1
    child->n += 1;

    // 接着，当前结点的keys部分,第idx位的key需要被右子树next的第一个key替换
    keys[idx] = next->keys[0];

    // 最后需要对右子树next进行调整
    // keys部分往左移一格
    for (int i = 1; i < next->n; ++i) {
        next->keys[i-1] = next->keys[i];
    }

    // 子树部分也往左移动一个
    if (!next->leaf) {
        for (int i = 1; i <= next->n; ++i) {
            next->children[i-1] = next->children[i-1];
        }
    }

    // 由于next最左边的关键字和子树被提走了，度数相应的减一
    next->n -= 1;

    return;
}


// 目标关键字不在当前结点上，idx是我们需要进一步第归的目标关键字
// 可能在的第idx个子结点，通过fill保证children[idx]的度不低于t
// 其处理了两中场景：3.a和3.b
// 3.a: 虽然children[idx]只有t-1的度，但是他有兄弟是t及以上的
//      通过旋转，完成子结点的度提升
// 3.b: children和他兄弟都只有t-1的度，就通过merge来完成子结点度的提升
// 返回fill之后的目标分支
int BTreeNode::fill(int idx) {

    int targetIdx = idx; // 默认在fill之后，目标分支还是idx

    // 如果左子树度数有多，则从左子树借
    if (idx != 0 && children[idx-1]->n >= t) {
        borrowFromPrev(idx);
    } else if (idx != n && children[idx+1]->n >= t) {
        // 如果右子树度数有多，则从右子树借
        borrowFromNext(idx);
    } else {
        // 如果左右子树都只有t-1，没有多于的
        // 如果不是最右的子树，则将目标子树和右边的子树合并
        // 合并后，还是挂在idx位
        if (idx != n)
            merge(idx);
        else {
            merge(idx-1);  // 注意，如果是在idx-1上进行的merge，则目标分支由原来的idx，变为idx-1!
            targetIdx = idx-1;
        }
    }

    return targetIdx;
}

// 基于当前结点删除关键字k
bool BTreeNode::remove(int k) {
    // 首先在当前结点上找到大于等于目标k的索引位置
    int idx = findKey(k);

    if (idx < n && keys[idx] == k) {
        // 如果正好找到了，则进一步判断当前结点是否位叶子结点
        if (leaf) {
            removeFromLeaf(idx);
        } else {
            removeFromNonLeaf(idx);
        }

        return true;// 总归是找到了，就返回true
    } else {
        // 如果没找到，那idx就是目标分支
        if (leaf) {
            // 如果已经到叶子结点了，都没找到，则找不到该key了
            return false;
        }

        // 接着判断，是否需要执行场景3的补足操作
        if (children[idx]->n < t) {
            idx = fill(idx);
        }

        // 补足之后，继续第归，在目标子树上进行移除
        return children[idx]->remove(k);
    }

}


bool BTree::remove(int k) {

    if (root == nullptr) {
        return false;
    }

    bool res = true;
    res = root->remove(k);

    // 如果remove完之后，度数已经降到0，则有两种情况
    // 要么已经没有数据了， 要么还有一个子结点，得降高度
    if (root->n == 0) {
        BTreeNode *tmp = root;
        if (root->leaf) {
            root = nullptr;
            // 什么时候root->leaf会变成true？
            // 只有在最后，先进入下面的else分支时，这个时候，root已经变成了root->children[0]
            // 而root->children[0]的leaf就是true，也就是新的root->leaf为true.
        } else {
            root = root->children[0];
        }

        // 释放老root
        delete tmp;
    }

    return res;
}
```

# 实际运行效果

针对上述代码，简单构建了两个场景：插入，删除。主要调用`BTree::insert`、`BTree::remove`、`BTree::search`和`BTree::traverse`接口，实际验证代码实现的正确性

## 插入关键字

```cpp
void insertDemo() {

    BTree t(3); // 构造一个t=3(度数介于[2,5]之间)的B树
    t.insert(10);
    t.insert(20);
    t.insert(5);
    t.insert(6);
    t.insert(12);
    t.insert(30);
    t.insert(7);
    t.insert(17);

    cout << "遍历B树:" << endl;
    t.traverse();
    cout << "" << endl;

    int k = 6;
    if (t.search(k) != nullptr) {
        cout << "关键字" << k << "存在" << endl;
    } else {
        cout << "关键字" << k << "不存在" << endl;
    }

    k = 15;
    if (t.search(k) != nullptr) {
        cout << "关键字" << k << "存在" << endl;
    } else {
        cout << "关键字" << k << "不存在" << endl;
    }

    return;
}
```

实际运行效果如下：
```bash
遍历B树:
 5 6 7 10 12 17 20 30
关键字6存在
关键字15不存在

```

##删除关键字
```cpp
void deleteDemo(ArgsT *pArgs) {

    BTree t(3); // 构造一个t=3(度数介于[2,5]之间)的B树

    // [1, 25]
    t.insert(1);
    t.insert(3);
    t.insert(7);
    t.insert(10);
    t.insert(11);
    t.insert(13);
    t.insert(14);
    t.insert(15);
    t.insert(18);
    t.insert(16);
    t.insert(19);
    t.insert(24);
    t.insert(25);
    t.insert(26);
    t.insert(21);
    t.insert(4);
    t.insert(5);
    t.insert(20);
    t.insert(22);
    t.insert(2);
    t.insert(17);
    t.insert(12);
    t.insert(6);

    cout << "遍历B树:" << endl;
    t.traverse();
    cout << "" << endl;

    // 尝试删除数据
    bool res = false;

    // 移除6
    int key = 6;
    res = t.remove(key);
    (res)?(cout << "成功移除"):(cout << "失败移除"); cout << key << endl;
    cout << "移除" << key << "后，遍历：" << endl;
    t.traverse();
    cout << endl;

    // 移除13
    key = 13;
    res = t.remove(key);
    (res)?(cout << "成功移除"):(cout << "失败移除"); cout << key << endl;
    cout << "移除" << key << "后，遍历：" << endl;
    t.traverse();
    cout << endl;

    // 移除7
    key = 7;
    res = t.remove(key);
    (res)?(cout << "成功移除"):(cout << "失败移除"); cout << key << endl;
    cout << "移除" << key << "后，遍历：" << endl;
    t.traverse();
    cout << endl;

    // 移除4
    key = 4;
    res = t.remove(key);
    (res)?(cout << "成功移除"):(cout << "失败移除"); cout << key << endl;
    cout << "移除" << key << "后，遍历：" << endl;
    t.traverse();
    cout << endl;

    // 移除2
    key = 2;
    res = t.remove(key);
    (res)?(cout << "成功移除"):(cout << "失败移除"); cout << key << endl;
    cout << "移除" << key << "后，遍历：" << endl;
    t.traverse();
    cout << endl;

    // 移除16
    key = 16;
    res = t.remove(key);
    (res)?(cout << "成功移除"):(cout << "失败移除"); cout << key << endl;
    cout << "移除" << key << "后，遍历：" << endl;
    t.traverse();
    cout << endl;

    // 尝试移除100
    key = 100;
    res = t.remove(key);
    (res)?(cout << "成功移除"):(cout << "失败移除"); cout << key << endl;
    cout << "移除" << key << "后，遍历：" << endl;
    t.traverse();
    cout << endl;

    return;
}
```

实际运行效果：
```bash
遍历B树:
 1 2 3 4 5 6 7 10 11 12 13 14 15 16 17 18 19 20 21 22 24 25 26
成功移除6
移除6后，遍历：
 1 2 3 4 5 7 10 11 12 13 14 15 16 17 18 19 20 21 22 24 25 26
成功移除13
移除13后，遍历：
 1 2 3 4 5 7 10 11 12 14 15 16 17 18 19 20 21 22 24 25 26
成功移除7
移除7后，遍历：
 1 2 3 4 5 10 11 12 14 15 16 17 18 19 20 21 22 24 25 26
成功移除4
移除4后，遍历：
 1 2 3 5 10 11 12 14 15 16 17 18 19 20 21 22 24 25 26
成功移除2
移除2后，遍历：
 1 3 5 10 11 12 14 15 16 17 18 19 20 21 22 24 25 26
成功移除16
移除16后，遍历：
 1 3 5 10 11 12 14 15 17 18 19 20 21 22 24 25 26
失败移除100
移除100后，遍历：
 1 3 5 10 11 12 14 15 17 18 19 20 21 22 24 25 26

```