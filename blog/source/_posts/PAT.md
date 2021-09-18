---
title: PAT
date: '2020/6/16 15:06:08'
updated: '2020/11/23 19:56:46'
tags: []
category:
  - 考研
  - 数据结构
mathjax: true
toc: false
abbrlink: 16991c0b
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

## 建立二叉树

中序inorder+后序postorder 

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

## 树的遍历

树的结点

```c++
struct TNode{  
	int w;
  vector<int> child;
  //int len,layer;
}Node[maxn];
```



树的先根遍历-dfs，层序遍历-bfs

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

## 二叉查找树

特点：中序遍历有序。

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

- 完全二叉查找树：同时有两者的性质，用数组存储，父子结点之间的关系用下标表示，for循环即是层序遍历；完全二叉树的形状已经确定了，又因为是查找树，对数组进行一次中序遍历，即可构建出树。
- 给n个结点关系，建立二叉查找树：静态链表存储，建树，然后按中序遍历将排序后的数组填入；

## AVL树

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
//左旋，注意是引用
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

## 并查集

求相同属性个数，树状结构找根结点却不关心中间其他结点。

```c++
void init(int n) {
    for (int i = 1; i <= n; i++)
        father[i] = i;
}

int findRoot(int x) {
  //备份
    int a = x;
  // 找到根
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

## 堆排序（求第k大/小的数）

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

# 图

## 遍历

### DFS

```c++
//每次DFS遍历一个连通块
void DFS(int u) {
    visit[u] = true;
    for (int i = 0; i < Node[u].adj.size(); i++) {
        int v = Node[u].adj[i];
        if (visit[v])
            continue;
        DFS(v);
    }
}

void DFSTrave() {
    for (int u = 0; u < maxn; u++) {
        if (visit[u])
            continue;
        DFS(u);
    ｝
}
```

### BFS

```c++
struct node {
    int v;    //顶点编号
    int layer;
    node(int _v, int _lay) :v(_v), layer(_lay) {};
};
vector<node> adj[maxn];

void BFS(int query,int l,int &num) {
    bool inq[maxn] = { false };
    queue<node> q;
    node u(query, 0);    //初始点
    q.push(u);
    inq[query] = true;    //入队
    while (q.size() > 0) {
        node now = q.front();
        q.pop();
        int u = now.v;
        for (int i = 0; i < adj[u].size(); i++) {
            node next = adj[u][i];
            next.layer = now.layer + 1;
            if (inq[next.v] == false&&next.layer<=l)    //题目条件
            {
                q.push(next);
                inq[next.v] = true;
                num++;
            }
        }
    }
}
```

* 无向图添加x条边使其连通：x=连通块个数-1； 计算连通块个数：DFS或者并查集

## 单源最短路径

### Dijkstra

```c++
const int INF = (1 << 31) - 1;
// s：起点；D：题目要求的终点
void Dijkstra(int s,int D) {
  // d[i]：起点到i点的最短路径长度
    fill(d,d+maxn,INF);
    fill(num,num+maxn,0);
    d[s] = 0;
    c[s] = 0;    //第二标尺
    num[s]=1;        //经过该结点的最短路径条数
    for (int i = 0; i < n; i++)
        pre[i] = i;        //记录前驱结点
    for (int i = 0; i < n; i++) {
        int u = -1, MIN = INF;
        for (int j = 0; j < n; j++) {
            if (visit[j] == false && d[j] < MIN)    //找出最短边
            {
                u = j;
                MIN = d[j];
            }
        }
        if (u == -1 || u == D) return;    //退出
        visit[u] = true;        //加入集合
        for (int v = 0; v < n; v++) {
            if (visit[v] == false && G[u][v] != INF) {
                if (d[u] + G[u][v] < d[v]) {
                    d[v] = d[u] + G[u][v];
                    c[v] = c[u] + cost[u][v];
                    pre[v] = u;
                    num[v]=num[u];
                }
                else if (d[u] + G[u][v] == d[v]) {
                    if (c[u] + cost[u][v] < c[v])
                    {
                        c[v] = c[u] + cost[u][v];
                        pre[v] = u;
                        num[v]+=num[u];
                    }
                }
            }
        }
    }
}

//输出路径
void DFS(int v) {
    if (v == s) {
        printf("%d ", v);
        return;
    }
    DFS(pre[v]);
    printf("%d ", v);
}
```

多标尺要求通用模板，用Dijkstra+DFS

```c++
void Dijkstra(int s,int ed) {
    fill(d, d+n, INF);
    d[s] = 0;
    for (int i = 0; i < n; i++) {
        int u = -1, MIN = INF;
        for (int j = 0; j < n; j++) {
            if (visit[j] == false && d[j] < MIN)
            {
                MIN = d[j];
                u = j;
            }
        }
        if (u == -1 || u == ed) return;
        visit[u] = true;
        for (int v = 0; v < n; v++) {
            if (visit[v] == false && G[u][v] != INF) {
                if (d[u] + G[u][v] < d[v]) {
                    d[v] = d[u] + G[u][v];
                    pre[v].clear();        //pre是一个vector<int>
                    pre[v].push_back(u);
                }
                else if (d[u] + G[u][v] == d[v]) {
                    pre[v].push_back(u);
                }
            }
        }
    }
}

vector<int> temppath, path;
int send = INF, back = INF;
void DFS(int v) {
    if (v == 0) {       //到达源点
        temppath.push_back(v);
      
      // 题目条件的第二标尺判断
        int need = 0, remain = 0;
        for (int i = temppath.size() - 2; i >= 0; i--) {
            int id = temppath[i];
            if (ci[id] > 0) {
                remain += ci[id];
            }
            else {
                if (remain > abs(ci[id])) {
                    remain -= abs(ci[id]);
                }
                else {
                    need += abs(ci[id]) - remain;
                    remain = 0;
                }
            }
        }
        if (need < send) {        //根据条件更新最佳路径
            send = need;
            back = remain;
            path = temppath;
        }
        else if (need == send&&remain < back) {
            back = remain;
            path = temppath;
        }
        temppath.pop_back();
        return;
    }
    temppath.push_back(v);    //加入v
    for (int i = 0; i < pre[v].size(); i++) {
        DFS(pre[v][i]);
    }
    temppath.pop_back();        //不加入v
}

// 打印path
```

### SPFA

判断是否有负环 

```c++
bool SPFA(int s){
    fill(inq,inq+maxn,false);
    fill(num,num+maxn,0);
    fill(d,d+maxn,INF);
    queue<int> q;
    q.push(s);
    inq[s]=true;
    num[s]++;
    d[s]=0;
    while(!q.empty()){
        int u=q.front();
        q.pop();
        inq[u]=false;
        for(int j=0;j<adj[u].size();j++){
            int v=adj[u][j].v;
            int dis=adj[u][j].dis;
            if(d[u]+dis<d[v]){
                d[v]=d[u]+dis;
                if(!inq[v]){
                    q.push(v);
                    inq[v]=true;
                    num[v]++;
                    if(num[v]>=n) return false;
                }
            }
        }
    }
    return true;
}
```

## 全源最短路径

### Floyd

```c++
void Floyd(){
    for(int k=0;k<n;k++){        //依次加入中间结点
        for(int i=0;i<n;i++){
            for(int j=0;j<n;j++){    //更新i,j之间的最短距离
                if(dis[i][k]!=INF&&dis[k][j]!=INF&&
                dis[i][k]+dis[k][j]<dis[i][j]){
                    dis[i][j]=dis[i][k]+dis[k][j];
                }
            }
        }
    }
}
```

## 最小生成树

### prime算法

从源点st开始，依次加入集合S的最短边和对应的结点，代码类似Dijkstra ; d[v]=G\[u][v] ；//d[]表示与集合S的距离

### kruskal算法

不需要源点，边贪心，先对边排序，选择全图中最短边；用并查集，加入边的时候判断会不会构成回路；

## 拓扑排序

判断有向无环图：用队列；` if(num==n) return true;`

## 关键路径

拓扑排序，建立正拓扑，得到`ve`（最长路径）；逆拓扑，得到`vl`（较小值）；

# 动态规划

主要是状态转移方程和边界。状态转移方程需要保证状态无后效性。

## 数塔

状态转移方程：`dp[i][j] = max(dp[i+1][j], dp[i+1][j+1]) + f[i][j]`

## 序列问题

注意：子序列可以不连续，子串必须是连续的。

### 最大连续子序列和

O(n^2) -> O(n)

状态转移方程：`dp[i]=max{A[i],dp[i-1]+A[i]}`。

备注： `dp[i]`表示以`A[i]`结尾的连续子序列最大和。

边界：`dp[0]=A[0]`。

### 最长不下降子序列

Longest Increasing Sequence, LIS。O(2^n) -> O(n^2)

状态转移方程：`dp[i]=max{dp[i], dp[j]+1}`。

备注：`dp[i]`表示以`A[i]`结尾的LIS长度。

边界：`dp[i]=1`。

应用：最爱的颜色，给出序列，子序列需要满足eva最爱的颜色序列，求满足要求的最长序列长度； 同样可以用最长公共子序列做（修改方程）。

```c++
for(int i=1; i<N; i++)
	{
		for(int j=0;j<i;j++)
		{
			if(A[i] >= A[j])
			{
				dp[i] = Math.max(dp[i], dp[j]+1); //状态转移方程
			}
		}
}
```



### 最长公共子序列

Longest Common Subsequence, LCS。O(2^(m+n) * max(m,n)) -> O(nm)

状态转移方程：`dp[i][j]=dp[i-1][j-1]+1,A[i]==B[j] `；`dp[i][j]=max{dp[i-1][j],dp[i][j-1]}`,`A[i]!=B[j]`；

备注：`dp[i][j]`表示`A[i]`和`B[j]`之前的LCS长度。

 边界:`dp[i][0]`=`dp[0][j]`=0；

### 最长回文子串

状态转移方程：`dp[i][j]`=`dp[i+1][j-1]`,` s[i]`==`s[j]`；`dp[i][j]`=0, `s[i]`!=`s[j]`； 

备注：`dp[i][j]`表示`s[i]`到`s[j]`的子串是否回文，是为1，不是为0；

边界：`dp[i][i]`=1,`dp[i][i+1]`=(`s[i]`==`s[i+1]`)?1:0

枚举方式需要改变：

```c++
int ans = 1;
// 长度为1和2的字符串进行初始化
for (int i = 0; i < len; i++) {
        dp[i][i] = 1;
        if (i < len - 1) {
            if (str[i] == str[i + 1])
            {
                dp[i][i + 1] = 1;
                ans = 2;
            }
        }
}
//取到等号,l为字符串长度
for (int l = 3; l <= len; l++) {    
        for (int i = 0; i + l - 1 < len; i++) {
            int j = i + l - 1;
            if (str[i] == str[j] && dp[i + 1][j - 1] == 1)
            {
                dp[i][j] = 1;
                ans = l;
            }
        }
}
```

## DAG最长路

状态转移方程：`dp[i] = max{dp[i], dp[j]+G[i][j]}`

备注：`dp[i]`表示从`i`号顶点出发能获得的最大路径长度。`j`为`i`能到达的顶点。

边界：出度为0的顶点`dp[k]=0`。当求到达终点T的最大路径长度时，边界为：dp初始化为-INF，`dp[T]=0`，还需要`visit`数组标识是否被计算过。

有向无环图：不使用拓扑排序，可以用递归DFS输出路径。

 矩形嵌套问题；

序列满足一定前后继关系，求最长序列；

## 背包

### 01背包

状态转移方程：`dp[i][v]=max{dp[i-1][v],dp[i-1][v-w[i]]+c[i]}`

备注：`dp[i][v]`表示前`i`件物品恰好装入容量为v的背包中能获得的最大价值。可以把`i`纬去掉，但是v必须逆序，从V到0（为了保证`dp[v-w[i]]`是`i-1`的值）。

边界：`dp[v]=0`。

```c++
fill(dp, dp + maxn, 0);
// 此处因为要按字典序顺序，需要字典序最小的元素最后更新choice
for (int i = n - 1; i >=0; i--) {
  for (int v = V; v >= w[i]; v--) {
    if (dp[v] <= dp[v - w[i]] + w[i]) {
      dp[v] = dp[v - w[i]] + w[i];
      choice[i][v] = true;
    }
    else
      choice[i][v] = false;
  }
}
```



### 完全背包

状态转移方程：`dp[i][v]=max{dp[i-1][v], dp[i][v-w[i]]+c[i]}`

备注：`dp[i][v]`表示前`i`件物品恰好装入容量为v的背包中能获得的最大价值。把`i`维去掉，v需要正向枚举，因为`dp[i][v-w[i]]`是`i`时刻的值，需要已经计算过。

