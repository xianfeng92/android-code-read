# Red-Black Trees

红黑树，本质上来说就是一棵二叉查找树，但它在二叉查找树的基础上增加了着色和相关的性质使得红黑树相对平衡，
从而保证了红黑树的查找、插入、删除的时间复杂度最坏为O(log n)。

但它是如何保证一棵 n 个结点的红黑树的高度始终保持在　h = logn　的呢？这就引出了红黑树的5条性质:

1. 每个结点要么是红的，要么是黑的。 
2. 根结点是黑的。 
3. 每个叶结点（叶结点即指树尾端NIL指针或NULL结点）是黑的。 
4. 如果一个结点是红的，那么它的俩个儿子都是黑的。 
5. 对于任一结点而言，其到叶结点树尾端NIL指针的每一条路径都包含相同数目的黑结点。 

正是红黑树的这5条性质，使得一棵 n 个结点是红黑树始终保持了 logn 的高度，从而也就解释了上面我们所说的“红黑树的查找、
插入、删除的时间复杂度最坏为O(log n)”这一结论的原因。

![RedBlackTree.png](https://github.com/xianfeng92/Awsome-Android/blob/master/images/RedBlackTree.png)

[教你透彻了解红黑树](https://github.com/julycoding/The-Art-Of-Programming-By-July/blob/master/ebook/zh/03.01.md)

