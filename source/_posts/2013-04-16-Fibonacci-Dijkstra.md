---
layout: post
title: Fibonacci堆实现的Dijkstra算法
description: Dijkstra算法的Fibonacci Heap实现，实现时间复杂度O(m+nlgn)
category: 算法
tags: [Dijkstra]
---

写在最前，因为我目前的一个项目里面需要Dijkstra的实现，为了实现效率的最大化，我尝试去实现一个Fibonacci堆来提升Dijkstra算法执行效率，网上这方面的资料很杂乱，这篇文章中是我整理的内容，少量自己的代码工作。一些参考资料在文章最末。


## Dijkstra算法

首先（虽然，我觉得这不是重点）先了解Dijkstra算法：

### Dijkstra简介

Dijkstra算法是图中的典型的单源最短路径算法，算法解决的是图中单个源点到其他顶点的最短路径问题。

这个算法是通过为每个顶点 v 保留目前为止所找到的从s到v的最短路径来工作的。初始时，原点 s 的路径长度值被赋为 0 （d[s] = 0），若存在能直接到达的边（s,m），则把d[m]设为w（s,m）,同时把所有其他（s不能直接到达的）顶点的路径长度设为无穷大，即表示我们不知道任何通向这些顶点的路径（对于 V 中所有顶点 v 除 s 和上述 m 外 d[v] = ∞）。当算法退出时，d[v] 中存储的便是从 s 到 v 的最短路径，或者如果路径不存在的话是无穷大。 Dijkstra 算法的基础操作是边的拓展：如果存在一条从 u 到 v 的边，那么从 s 到 v 的最短路径可以通过将边（u, v）添加到尾部来拓展一条从 s 到 u 的路径。这条路径的长度是 d[u] + w(u, v)。如果这个值比目前已知的 d[v] 的值要小，我们可以用新值来替代当前 d[v] 中的值。拓展边的操作一直运行到所有的 d[v] 都代表从 s 到 v 最短路径的花费。这个算法经过组织因而当 d[u] 达到它最终的值的时候每条边（u, v）都只被拓展一次。

算法维护两个顶点集 S 和 Q。集合 S 保留了我们已知的所有 d[v] 的值已经是最短路径的值顶点，而集合 Q 则保留其他所有顶点。集合S初始状态为空，而后每一步都有一个顶点从 Q 移动到 S。这个被选择的顶点是 Q 中拥有最小的 d[u] 值的顶点。当一个顶点 u 从 Q 中转移到了 S 中，算法对每条外接边 (u, v) 进行拓展。

### Dijkstra伪代码

这里依然引用[Wiki][1]中的内容:

	function Dijkstra(Graph, source):
		for each vertex v in Graph:                                // Initializations
			dist[v] := infinity ;                                  // Unknown distance function from 
                                                                   // source to v
            previous[v] := undefined ;                             // Previous node in optimal path
        end for                                                    // from source
        
        dist[source] := 0 ;                                        // Distance from source to source
        Q := the set of all nodes in Graph ;                       // All nodes in the graph are
                                                                   // unoptimized - thus are in Q
        while Q is not empty:                                      // The main loop
            u := vertex in Q with smallest distance in dist[] ;    // Source node in first case
            remove u from Q ;
            if dist[u] = infinity:
                break ;                                            // all remaining vertices are
            end if                                                 // inaccessible from source
            
            for each neighbor v of u:                              // where v has not yet been 
                                                                   // removed from Q.
                alt := dist[u] + dist_between(u, v) ;
                if alt < dist[v]:                                  // Relax (u,v,a)
                    dist[v] := alt ;
                    previous[v] := u ;
                    decrease-key v in Q;                           // Reorder v in the Queue
                end if
            end for
        end while
    return dist;
	
注意：如果我们只对在 s 和 t 之间查找一条最短路径的话，我们可以在第12行添加条件如果满足 u = t 的话终止程序。

如果需要记录最佳路径的轨迹，则标记该路径上每个点的前趋，即可通过迭代来回溯出 s 到 t 的最短路径：

	S := empty sequence
	u := target
	while previous[u] is defined:                                   // Construct the shortest path with a stack S
       insert u at the beginning of S                              // Push the vertex into the stack
       u := previous[u]                                            // Traverse from target to source
    end while ;

S 即从 s 到 t 的最短路径的顶点集。

### Dijkstra的时间复杂度

最简单的Dijkstra实现中我们利用数组（或者链表）来存储所有顶点的集合Q，此时时间的复杂度是`O(n*n)`;

而利用一个二项堆实现的Dijkstra算法可以将时间复杂度优化至`O((m + n)log n)`

[Fibonacci Heap][2]则可以将时间复杂度进一步优化到`O(m + n log n)`

注意，这里m代表边的数量，n代表点的数量。

## Fibonacci heap

斐波那契堆(Fibonacci heap)是计算机科学中最小堆有序树的集合。它和二项式堆有类似的性质，可用于实现合并优先队列。 不涉及删除元素的操作有O(1)的平摊时间。 Extract-Min和Delete的数目和其它相比，较小时效率更佳。 稠密图每次Decrease－key只要O(1)的平摊时间，和二项堆的O(lgn)相比是巨大的改进。

斐波那契堆的关键思想在于尽量延迟对堆的维护。例如，当向斐波那契堆中插入新结点或合并两个斐波那契堆时，并不去合并树，而是将这个工作留给EXTRACT-MIN操作。

更多[Fibonacci heap 的 Wiki][2]

### Finbonacci heap的结构

每个结点x有以下内容：

* 结点值 x
* 父节点p[x]
* 指向任一子女的指针child[x] (结点x的子女被链接成一个环形双链表，称为x的子女表)
* 左兄弟left[x]
* 右兄弟right[x] (当left[x] = right[x] = x时，说明x是独子)
* 子女的个数degree[x]
* 布尔值域mark[x] (标记是否失去了一个孩子)

堆H的结构：

* 结点个数n[H]
* 指向最小结点的指针 *min

![FibHeap](/images/fibonacci/FibHeapPic.jpg)

### 创建Fibonacci heap



	Make-Fibonacci-Heap()
		n[H] := 0
		min[H] := NIL 
		return H

### 插入结点

插入一个结点x，对结点的各域初始化，赋值，然后构造自身的环形双向链表后，将x加入H的根表中。

	Fibonacci-Heap-Insert(H,x)
		degree[x] := 0
		p[x] := NIL
		child[x] := NIL
		left[x] := x
		right[x] := x
		mark[x] := FALSE
		concatenate the root list containing x with root list H
		if min[H] = NIL or key[x]<key[min[H]]
        	then min[H] := x
		n[H]:= n[H]+1

如图是将关键字为21的结点插入斐波那契堆。该结点自成一棵最小堆有序树，从而被加入到根表中，成为根的左兄弟。	
	
![FibHeapInsert](/images/fibonacci/FibHeapInsertPic.jpg)

### 合并两个Fibonacci heap

将H1和H2的两根表并置，然后确定一个新的最小结点。

	Fibonacci-Heap-Union(H1,H2)
		H := Make-Fibonacci-Heap()
		min[H] := min[H1]
		Concatenate the root list of H2 with the root list of H
		if (min[H1] = NIL) or (min[H2] <> NIL and min[H2] < min[H1])
  	 		then min[H] := min[H2]
		n[H] := n[H1] + n[H2]
		free the objects H1 and H2
		return H

### 获得最小结点

	Fibonacci-Heap-Minimum(H)
		return min[H]
		

### 抽取最小结点

抽取最小这个操作比较麻烦。该过程中还用到一个辅助过程`CONSOLIDATE`。

这个过程先使最小结点的每个子女都成为一个根，并将最小结点从根表中取出。然后，通过将度数相同的根链接起来，直至对应每个度数至多只有一个根来调整根表。`FIB-HEAP-EXTRACT-MIN`中，3~5行中使z的所有子女成为根（将他们放入根表）来从H中删除结点z，并在第6行中将z从根表中去掉。如果z为根节点中唯一的结点且没有子女，则第8行返回空即可；否则，让指针min[H]指向根表中的一个非z的结点（伪代码中为`right[z]`)。这个min[H]只是临时值，并不是真正的最小结点。第9行之前程序执行过程。

`CONSOLIDATE`过程要做的工作是：使每个度数的二项树唯一，也就是使每个根都有一个不同的degree值为止。对根表的合并过程是反复执行下面的步骤：

1）在根表中找出两个具有相同度数的根x和y，且key[x] <= key[y].

2)将y链接到x：将y从根表中移出，成为x的一个孩子。这个过程由`FIB-HEAP-LINK`完成。

	Fibonacci-Heap-Link(H,y,x)
		remove y from the root list of H
		make y a child of x
		degree[x] := degree[x] + 1
		mark[y] := FALSE


	CONSOLIDATE(H)
		for i:=0 to D(n[H])
    		Do A[i] := NIL
		for each node w in the root list of H
    		do x:= w
       			d:= degree[x]
       			while A[d] <> NIL
           			do y:=A[d]
              			if key[x]>key[y]
                			then exchange x<->y
              			Fibonacci-Heap-Link(H, y, x)
              			A[d]:=NIL
             		d:=d+1
      	 		A[d]:=x
		min[H]:=NIL
		for i:=0 to D(n[H])
    		do if A[i]<> NIL
          		then add A[i] to the root list of H
              		if min[H] = NIL or key[A[i]]<key[min[H]]
                  		then min[H]:= A[i]


	Fibonacci-Heap-Extract-Min(H)
		z:= min[H]
		if x <> NIL
        	then for each child x of z
             	do add x to the root list of H
                	p[x]:= NIL
             	remove z from the root list of H
             	if z = right[z]
                	then min[H]:=NIL
                	else min[H]:=right[z]
                     	CONSOLIDATE(H)
             	n[H] := n[H]-1
		return z

看图说话

![FibExtract1](/images/fibonacci/FibExtractPic1.jpg)

![FibExtract2](/images/fibonacci/FibExtractPic2.jpg)


### 减小一个关键字操作
	
减小一个关键字的困难在于，如果减小后的结点破坏了最小堆的性质，如何维护Fibonacci heap的性质。这里用到一个操作：级联剪枝（`Cascading Cut`）

基本流程：如果减小后的结点破坏了最小堆性质，则把它切下来(cut)，即从所在双向链表中删除，并将其插入到由最小树根节点形成的双向链表中，然后再从parent[x]到所在树根节点递归执行级联剪枝。

	CUT(H,x,y)
		Remove x from the child list of y, decrementing degree[y]
		Add x to the root list of H
		p[x]:= NIL
		mark[x]:= FALSE
 
	CASCADING-CUT(H,y)
		z:= p[y]
		if z <> NIL
  			then if mark[y] = FALSE
       			then mark[y]:= TRUE
      		 	else CUT(H, y, z)
            		CASCADING-CUT(H, z)

	Fibonacci-Heap-Decrease-Key(H,x,k)
		if k > key[x]
  	 		then error "new key is greater than current key"
		key[x] := k
		y := p[x]
		if y <> NIL and key[x]<key[y]
  			then CUT(H, x, y)
        		CASCADING-CUT(H,y)    
		if key[x]<key[min[H]]
   			then min[H] := x	

如图，a),b)46减小为5；  c),d),e)35减小为5

![FibHeapDecrease](/images/fibonacci/FibHeapDecrease.jpg)


### 删除一个结点

过程很简单，先减小直到min[H], 然后直接剔除最小值即可 

	Fibonacci-Heap-Delete(H,x)
		Fibonacci-Heap-Decrease-Key(H,x,-infinity)
		Fibonacci-Heap-Extract-Min(H)


## 完整代码

这里是我在用的一个c++代码，从工程里直接摘出来，有改动，未测试，仅供参考。

	class fib_heap  
	{  
    class fib_node  
    {  
    public:  
        fib_node(const vt& v, const int& n)  
            : parent(NULL)  
            , child(NULL)  
            , degree(0)  
            , marked(false)  
            , value(v)   
        {  
		}  
        fib_node *parent;  
        fib_node *prev;  
        fib_node *next;  
        fib_node *child;  
        size_t degree;  
        bool marked;  
        vt value;  
    };  
	public:  
    class iterator  
    {  
    public:  
        iterator()  
            : p(NULL)  
        {  
        }  
  
        iterator(fib_node *p_)  
            : p(p_)  
        {  
        }  
  
        vt& operator*() {return p->value;}  
        vt* operator->() {return &p->value;}  
        bool operator==(iterator& other){return p==other.p;}  
        bool operator!=(iterator& other){return p!=other.p;}  
    private:  
        friend class fib_heap;  
        fib_node *p;  
    };  
  
    fib_heap()  
        : minRoot(NULL)  
        , combineVec(16)  
    {  
    }  
  
    ~fib_heap()  
    {  
        clear();  
    }  
  
    iterator insert(const vt& v,const int& n)   
    {  
        fib_node *newNode = new fib_node(v,n);  
        if (!minRoot) {  
            minRoot = newNode->next = newNode->prev = newNode;  
        } else {  
            newNode->prev = minRoot;  
            newNode->next = minRoot->next;  
            minRoot->next->prev = newNode;  
            minRoot->next = newNode;  
            if (newNode->value < minRoot->value)  
                minRoot = newNode;  
        }  
        return iterator(newNode);  
    }  
  
    bool empty()  
    {  
        return minRoot == NULL;  
    }  
  
    vt& minR()  
    {  
        return minRoot->value;  
    }  
  
    // heap must not be empty  
    void deleteMin()  
    {  
        // make minRoot's children roots  
        fib_node *child = minRoot->child;  
        if (child) {  
            fib_node *curchild = child;  
            do {  
                curchild->parent = NULL;  
                curchild->marked = false;  
                curchild = curchild->next;  
            } while (child != curchild);  
            child->prev->next = minRoot->next;  
            minRoot->next->prev = child->prev;  
            child->prev = minRoot;  
            minRoot->next = child;  
        }  
  
        // combine root with equal degree  
        fib_node *curNode = minRoot->next;  
        fib_node *nextNode;  
        while (curNode != minRoot) {  
            nextNode = curNode->next;  
            size_t degree = curNode->degree;  
            fib_node *target = combineVec[degree];  
            while (target) {  
                if (target->value < curNode->value) {  
                    fib_node *tmp = curNode;  
                    curNode = target;  
                    target = tmp;  
                }   
                // combine target to curNode  
                target->next->prev = target->prev;  
                target->prev->next = target->next;  
                if (!curNode->child) {  
                    curNode->child = target->next = target->prev = target;  
                } else {  
                    fib_node *child = curNode->child;  
                    target->prev = child;  
                    target->next = child->next;  
                    child->next->prev = target;  
                    child->next = target;  
                }  
                target->parent = curNode;  
                combineVec[degree] = NULL;  
                degree = ++curNode->degree;  
                if (degree > combineVec.size())   
                    combineVec.resize(degree);  
                target = combineVec[degree];  
            }  
            combineVec[degree] = curNode;  
            curNode = nextNode;  
        }  
  
        // find new min and clear combineVec  
        curNode = minRoot->next;  
        fib_node *curMin = NULL;  
        while (curNode != minRoot) {  
            combineVec[curNode->degree] = NULL;  
            if (!curMin || curNode->value < curMin->value)  
                curMin = curNode;  
            curNode = curNode->next;  
        }  
        minRoot->next->prev = minRoot->prev;  
        minRoot->prev->next = minRoot->next;  
        delete minRoot;  
        minRoot = curMin;  
    }  
  
    void decrease(iterator it)  
    {  
        fib_node *node = it.p;  
        fib_node *parent = node->parent;  
        if (parent) 
		{  
            if (!(node->value < parent->value))  
                return;  
            cut(node);  
        }  
        if (node->value < minRoot->value)  
            minRoot = node;  
    }  
  
    void erase(iterator it)  
    {  
        fib_node *node = it.p;  
        if (node->parent) {  
            cut(node);  
        } else if (node == minRoot) {  
            deleteMin();  
            return;  
        }  
        fib_node *child = node->child;  
        if (child) {  
            fib_node *curchild = child;  
            do {  
                curchild->parent = NULL;  
                curchild->marked = false;  
                curchild = curchild->next;  
            } while (child != curchild);  
            child->prev->next = node->next;  
            node->next->prev = child->prev;  
            child->prev = node->prev;  
            node->prev->next = child;  
        } else {  
            node->next->prev = node->prev;  
            node->prev->next = node->next;  
        }  
        delete node;  
    }  
  
    void clear()  
    {  
        clearDescendants(minRoot);  
        minRoot = NULL;  
    }  
  
	private:  
    // node must not be root  
    void cut(fib_node *node)  
    {  
        do {  
            fib_node *parent = node->parent;  
            if (--parent->degree == 0) {  
                parent->child = NULL;  
            } else {  
                node->next->prev = node->prev;  
                node->prev->next = node->next;  
                parent->child = node->next;  
            }  
            node->parent = NULL;  
            node->marked = false;  
            node->next = minRoot->next;  
            node->prev = minRoot;  
            minRoot->next->prev = node;  
            minRoot->next = node;  
            node = parent;  
        } while (node->marked);  
        if (node->parent)  
            node->marked = true;  
    }  
  
    void clearDescendants(fib_node *child)  
    {  
        if (!child)  
            return;  
        fib_node *curchild = child;  
        do {  
            clearDescendants(curchild->child);  
            fib_node *tmp = curchild;  
            curchild = curchild->next;  
            delete tmp;  
        } while (curchild != child);  
    }  
  
	public:  
    	fib_node *minRoot;  
    	std::vector<fib_node *> combineVec;  
	};

下面是Dijkstra：

	float gDijkstra(Graph& gGraph,int src, int trg)
	{
	float lfTemp;
	float lfCost;//最短距离
	ANode* anpTemp;
	
	fib_heap<float> heap;
	fib_heap<float>::iterator *fdiTemp;
	fib_heap<float>::iterator *fdiNodes;
	fdiNodes = (fib_heap<float>::iterator*)malloc(sizeof(fib_heap<float>::iterator) * gGraph.vvnNodes.size());
	
	int *ipVisited;
	ipVisited = (int*)malloc(sizeof(int) * gGraph.vvnNodes.size());
	int* ipParent;
	ipParent = (int*)malloc(sizeof(int) * gGraph.vvnNodes.size());
	
	for(i = 0; i < gGraph.vvnNodes.size(); i++)
	{
		ipParent[i] = -1;
		ipVisited[i] = 0;
	}

	fdiNodes[src] = heap.insert(0,src);
	ipVisited[src] = 1;
	
	while(!heap.empty())
	{
		int minRootNum = heap.minRoot->iNodeNum;
		if(minRootNum == trg)
		{
			lfCost = heap.minR() - time;
			vector<int> viTemp;
			float iArrival = heap.minR();
			
			cout<<"Leave at: "<<time<<endl<<"Earliest reach at: "<<iArrival<<endl<<"Travel time: "<<lfCost<<endl;
			iTemp = trg;
			while(iTemp != -1)
			{
				viTemp.push_back(iTemp);
				iTemp = ipParent[iTemp];
			}
		
			while(!viTemp.empty())
			{
			
				cout<<viTemp.at(viTemp.size() - 1)<<",";
				viTemp.pop_back();
			}
		}
		heap.clear();
		free(ipVisited);
		free(ipParent);
	
		return lfCost;
		}
		anpTemp = gGraph.vvnNodes.at(minRootNum).anFirstArc;
		while(anpTemp != NULL)
		{
			lfTemp = anpTemp->eEdge->w
				
			if(ipVisited[anpTemp->iNodeNum] == 0)
			{
				fdiNodes[anpTemp->iNodeNum] = heap.insert(heap.minR() + lfTemp,anpTemp->iNodeNum);
				ipParent[anpTemp->iNodeNum] = minRootNum;
				ipVisited[anpTemp->iNodeNum] = 1;
			}
			else if(ipVisited[anpTemp->iNodeNum] == 1 && *fdiNodes[anpTemp->iNodeNum] > heap.minR() + lfTemp)
			{
				*fdiNodes[anpTemp->iNodeNum] = heap.minR() + lfTemp;
				ipParent[anpTemp->iNodeNum] = minRootNum;
				heap.decrease(fdiNodes[anpTemp->iNodeNum]);	
			}
			anpTemp = anpTemp->anNextArc;
		}
		ipVisited[minRootNum] = -1;
		heap.deleteMin();
	}
	heap.clear();
	free(ipVisited);
	free(ipParent);
	return -1;
	}

这里还要注意，这个实现中（也就是[raomeng1的博客][5]中的实现）有一个问题是其中有一个`min()`函数（我改成了`minR()`)和`windows.h`中的代码重定义了，所以如果你需要`windows.h`就需要修改代码。


## 参考资料

* [Dijkstra 的 Wiki][1]
* [Fibonacci heap 的 Wiki][2]
* [Michael L. Fredman , Robert Endre Tarjan, Fibonacci heaps and their uses in improved network optimization algorithms, Journal of the ACM (JACM), v.34 n.3, p.596-615, July 1987][3]
* [斐波那契堆(Fibonacci heaps) by 酷~行天下][4]
* [c++实现的Fibonacci heap模版类 by raomeng1][5]
* [http://www.cs.princeton.edu/~wayne/cs423/fibonacci/FibonacciHeapAlgorithm.html][6]


[1]: http://en.wikipedia.org/wiki/Dijkstra's_algorithm
[2]: https://en.wikipedia.org/wiki/Fibonacci_heap
[3]: http://dl.acm.org/citation.cfm?id=28874
[4]: http://mindlee.net/2011/09/29/fibonacci-heaps/
[5]: http://blog.csdn.net/raomeng1/article/details/7618228
[6]: http://www.cs.princeton.edu/~wayne/cs423/fibonacci/FibonacciHeapAlgorithm.html

