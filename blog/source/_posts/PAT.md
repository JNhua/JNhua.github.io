---
title: PAT
copyright: true
date: 2020-05-17 12:24:31
categories:
- Other
tags:
---

# 算法初步

## 排序

使用sort函数，必要时需要自定义cmp比较函数；

<!--more-->

## 散列

常用来打表，比如将字符串散列到一个长数组中；

## 递归

两个重点：递归边界，递推式；

## 贪心

局部最优，得到全局最优。应用例如：求最低价格。有时，需要加上对具体题目的规律推断。

## 二分

Upper_bound(begin, end, value) 找到第一个大于value的元素的地址，不存在则返回end；对数组或容器使用。

## two pointers

研究问题与序列的特性，使用两个互相牵制的下标对序列进行扫描；

# 数学问题

## 最大公约数

```c++
int gcd(int a,int b){
	return !b ? a : gcd(b, a%b);
}
```

## 分数的四则运算

利用结构体定义分子分母，分别运算；

## 素数

```c++
bool isPrime(int n){
  if(n<=1) return false;
  int sqr = (int)sqrt(1.0 *n);
  for(int i=2;i<=sqr;i++){
    if(n%i==0) return false;
  }
  return true;
}
```

## 质因子分解

```c++
struct factor{
	int x,cnt;
}fac[10];
```

从2到sqrt(n)范围的素数表中找质因子与对应的个数；如果没有被sqrt(n)以内的素数除尽，那么他本身是一个素数，质因子为本身；

## 大整数运算

```c++
struct bign{
	int d[1000];
	int len;
};
```

# STL

1. 只有vector和string支持*(it + i)的访问方式；
2. 遍历容器可以用`for(auto it: v)`；
3. String::nops是find函数查找失败的返回值，string的size_type的最大值；
4. queue在使用front()和pop()之前，要判断empty()；

# 简单数据结构

## 栈

STL的stack；

## 队列

STL的queue；

## 链表

```c++
struct Node{
  typename data;
  node* next;
};
```

静态链表

```c++
struct Node{
  typename data;
  int next;
  // 有时候需要加上其他标志
  // bool falg; //结点有效状态，排序可以剔除无效结点
  // int address; //记录结点地址
}node[size];
```

# 搜索

## DFS

常用递归解决；

### 剪枝

在进入分支前，先进行限制条件的判断；

```c++
void DFS(int index, int nowK, int nowSum,int fSum) {
    if (nowSum == n&&nowK == k) {    //递归边界（满足条件）
        if (fSum > maxfSum) {    //题目条件
            ans = temp;
            maxfSum = fSum;
        }
        return;
    }
    if (index == sqr||nowK>k||nowSum>n) return;    //递归边界（剪枝）
    temp.push_back(factor[index]);        //选择加入
    DFS(index , nowK + 1, nowSum + (int)pow(1.0*factor[index], p*1.0), fSum + factor[index]); // 如果不重复选，这里也该是index+1
    temp.pop_back();    //恢复，不加入
    DFS(index + 1, nowK, nowSum, fSum);
}
```

## BFS

队列实现，按层次遍历；

```c++
void BFS(int s){
  queue<int> q;
  q.push(s);
  falg[s] = true; // 标记s入队
  while(!q.empty()){
    int top = q.front(); // 取队首
    // 访问top，进行操作
    q.pop(); //队首出队
    //将top的下一层结点中，未曾入队的进行入队，并设置为已入队
  }
}
```

一次BFS/DFS不能够完全遍历完的（非连通），需要在外面再套一层循环。

当需要对队列中元素修改时，队列存储最好是他们的编号而不是元素本身。

# 树

- 二叉树结点

```c++
struct BTNode {
  int data;
  BTNode *l, *r;
  // int layer;
};
```

- 中序inorder+后序postorder 建立二叉树

```c++
BTNode* create(int postL, int postR, int inL, int inR) {
  if (postL > postR||inL>inR)
    return NULL;
  BTNode *root = new BTNode;
  //根据后序找到根结点
  root->data = post[postR]; 
  int i;
  for (i = inL; i <= inR; i++) {
    //找到中序中的根结点位置
    if (in[i] == post[postR]) {  
      break;
    }
  }
  //当前根结点左子树的结点个数
  int leftnum = i - inL;  
  root->l = create(postL,postL+leftnum-1 , inL, i-1);
  root->r = create(postL + leftnum, postR-1, i + 1, inR);
  return root;
}
```

  int w;

  vector<int> child;

  //int len,layer;

}Node[maxn];

- 树的先根遍历-dfs，层序遍历-bfs

```c++
// 子节点多于2个时，用静态链表方便
void preorder(int root, vector<int> &q,int sum) {
  sum += Node[root].w;
  //剪枝
  if (sum>s)    
    return;
  //加入
  q.push_back(Node[root].w);    
 //递归边界（满足题目条件）
 if (Node[root].child.size()==0&&sum == s) {  
    print(q);
    //恢复
    q.pop_back();    
    return;
  }
  sort(Node[root].child.begin(),
        Node[root].child.end(), cmp);
  for (int i = 0; i < Node[root].child.size(); i++) {   
     //先根遍历的核心
     preorder(Node[root].child[i], q,sum);
  }
  //不加入
  q.pop_back();        
}

void layerOrder(node *root){
	queue<BTNode*> q;
	q.push(root);
	int num = 0;
	while(q.size()>0){
		node* p = q.front();
		num++;
		cout << p->data;
		if (num < n)
			cout << " ";
		q.pop();
		if (p->l)
			q.push(p->l);
		if (p->r)
			q.push(p->r);
	}
}

void LayerOrder(int root){
  queue<int> q;
  q.push(root);
  Node[root].layer = 0;
  while(!q.empty()){
    int front = q.front();
    // ...访问根结点
    q.pop();
    for(int i=0;i<Node[front].child.size();i++){
      int child = Node[front].child[i];
      Node[child].layer = Node[front].layer + 1;
      q.push(child);
    }
  }
}
```

- 二叉查找树（BST）：中序遍历有序

  删除（1）root为空，直接返回；（2）root值为x，进入删除处理；（2.0）无左右孩子，直接删除；（2.1）root有左孩子，在左子树中找pre，让pre覆盖root，接着在左子树中删除pre；（2.2）root右孩子类似；（3）root->data 大于x，去root->l删；（4）root->data小于x，去root->r删；

  

  判断给出序列求是否是BST：（1）根据给出序列构建二叉树(insert)；（2）用vector比较是否一样（直接用==）；

  ```c++
  void insert(node *&root, int x,bool mirror) {
      if (root == NULL)
      {
          root = new node;
        //需要写这句，或者在struct node初始化里就写好
          root->l = root->r = NULL;
          root->data = x;
          return;
      }
      if (mirror) {
          if (x >= root->data) insert(root->l, x, mirror);
          else
              insert(root->r, x, mirror);
      }
      else {
          if (x >= root->data) insert(root->r, x, mirror);
          else
              insert(root->l, x, mirror);
      }
  }
  ```

- 完全二叉查找树：同时有两者的性质，用数组存储，父子结点之间的关系用下标表示，for循环即是层序遍历；
- 给n个结点关系，建立二叉查找树：静态链表存储，建树，然后按中序遍历将排序后的数组填入；
- AVL树

```c++
struct node {
    int data, height;    //记录当前树的高度
    node* l,* r;
};

node* newNode(int data) {
    node *root = new node;
    root->l = root->r = NULL;
    root->height = 1;
    root->data = data;
    return root;
}

int getHeight(node *root) {
    if (root == NULL)
        return 0;
    else
        return root->height;
}

void updateHeight(node* &root) {
    root->height = max(getHeight(root->l), getHeight(root->r)) + 1;
}

int getBalance(node* root) {
    return getHeight(root->l) - getHeight(root->r);    //一定是左-右
}
//左旋
void L(node* &root) {
    node* temp = root->r;
    root->r = temp->l;
    temp->l = root;
    updateHeight(root);    //一定先更新更低层的结点高度
    updateHeight(temp);
    root = temp;
}

//右旋
void R(node* &root) {
    node* temp = root->l;
    root->l = temp->r;
    temp->r = root;
    updateHeight(root);
    updateHeight(temp);
    root = temp;
}

void insert(node* &root,int data) {
    if (root == NULL)
    {
        root = newNode(data);
        return;
    }
    if (data >= root->data) {
        insert(root->r, data);    //递归插入，递归更新高度
        updateHeight(root);        //插入点先更新，然后往回退
        if (getBalance(root) == -2)
        {
            //RR
            if (getBalance(root->r) == -1) {
                L(root);
            }
            //RL
            else if (getBalance(root->r) == 1) {
                R(root->r);
                L(root);
            }
        }
    }
    else {
        insert(root->l, data);
        updateHeight(root);
        if (getBalance(root) == 2)
        {
            //LL
            if (getBalance(root->l) == 1) {
                R(root);
            }
            //LR
            else if (getBalance(root->l) == -1) {
                L(root->l);
                R(root);
            }
        }
    }
}
```



- 并查集（社交网络个数，树状结构找根结点却不关心中间其他结点）

```c++
void init(int n) {
    for (int i = 1; i <= n; i++)
        father[i] = i;
}

int findRoot(int x) {
    int a = x;
    while (x != father[x]) {
        x = father[x];
    }
    //路径压缩
    while (a != father[a]) {
        int z = a;
        a = father[a];
        father[z] = x;
    }
    return x;
}

void Union(int a, int b) {
    int fa = findRoot(a);
    int fb = findRoot(b);
    if (fa != fb) {
        father[fa] = fb;
    }
}
```



- 堆排序（求第k大/小的数）
- 建堆时downAdjust，插入结点用upAdjust；

```c++
//建大顶堆
void downAdjust(int low,int high) {
    int i = low, j=2*i;
    while (j <= high) {
        if (j+1<=high&&initial[j + 1] > initial[j])    //与较大孩子比较
            j++;
        if (initial[j] > initial[i]) {
            swap(initial[i],initial[j]);
            i = j;
            j = j * 2;
            continue;
        }
        else
            break;
    }
}

//建堆
void createHeap() {
    for (int i = n / 2; i >= 1; i--)    //从最后一个非叶结点开始
        downAdjust(i, n);
}


//deap sort，递增排序
void deap_sort() {
    createHeap();
    for (int i = n; i >= 2; i--) {
        swap(initial[i], initial[1]);    //取出最大值
        downAdjust(1, i - 1);
    }
}
```



- 哈夫曼树（非前缀编码），用priority_queue<long long,vector<long long>,greater<long long>> q;代表小顶堆，每次取出最小两个元素向上建树；



