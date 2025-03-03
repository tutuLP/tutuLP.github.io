---
title: "LeetCode算法总结"
date: 2025-03-02
categories:
  - 面试
tags:
  - 面试
---

#链表

## 总结

递归

链表方向转换+nullptr不断向前+尾节点成为头节点

~~~c++
class Solution {
    public:
    ListNode* reverseList(ListNode* head) {
        if (head == nullptr || head->next == nullptr) {
                return head;
        }
        ListNode* newHead = reverseList(head->next);
        head->next->next = head;  
            head->next = nullptr;     
        return newHead;
    }
};
~~~

栈(vector+rbegin+rend    或   stack)

其中变式题穿插双指针

## 反转链表

###206 反转链表  

第一遍自解 耗时0ms

~~~c++
class Solution {
    public:
    ListNode* reverseList(ListNode* head) {
        //遍历两遍，第一遍依次取出节点的值然后放入栈中，第二遍从栈中拿出元素构建链表
        int i=0;
        vector<int> tempList;
        ListNode * tempHead=head;
        while(tempHead!= nullptr){//注意这里不是tempHead->next
            tempList.push_back(tempHead->val);
            tempHead=tempHead->next;
            i++;
        }
        tempHead=head;//这里犯错了，用tempHead来移动，不要用head
        for (auto it = tempList.rbegin(); it != tempList.rend(); ++it) {
            tempHead->val=(*it);
            tempHead=tempHead->next;
        }
        return head;
    }
};
 
~~~

这里使用satck是一样的效果   

stack st; st.push(); 

while (!st.empty()) 

st.top(); 

st.pop();

####递归实现

递归思路  递归到最深处再依次返回，递归到末尾然后指针指向的顺序调转 

终止条件：递归到链表的末尾

递归到最深处返回的head(链表的尾节点)就是反转完的链表的头节点，作为newhead返回，递归回来的时候，指针方向不断反转，nullptr被不断往前移动，最终移动到反转前的头节点的下一位

~~~c++
class Solution {
    public:
    ListNode* reverseList(ListNode* head) {
        if (head == nullptr || head->next == nullptr) { //终止条件 最后一个节点最先满足终止条件head->next == nullptr，返回最后一个节点
                return head;
        }
        // 递归反转子链
        ListNode* newHead = reverseList(head->next); // 
        head->next->next = head;  //执行逻辑  最后一个节点触发return head;所以此时head是倒数第二个节点
            head->next = nullptr;     
        return newHead;
    }
};
~~~

* 终止条件只看一次，返回值返回后就不会变一直传递到最后  作为最终的返回
* 执行逻辑 最后一个节点触发终止条件返回，执行逻辑则从第四个节点开始到第一个节点

####一个循环

也就是声明三个变量，当前遍历的节点，上一个和下一个节点?

~~~c++
class Solution {
    public:
    ListNode* reverseList(ListNode* head) {
        if(!head || !head->next) 
            return head;
        ListNode* pre = nullptr;
        auto cur = head;
        auto next = head->next;
        while(next) {
            cur->next = pre;
            pre = cur;
            cur = next;
            next = next->next;
        }
        cur->next = pre;
        return cur;
    }
};
~~~

###92 反转链表

给出头指针和两个位置，将两个位置之间的节点反转 第一遍自解 0ms

我的思路：找到区间起点记录指针到终点依次压栈然后再拿出来

~~~c++
class Solution {
    public:
    ListNode* reverseBetween(ListNode* head, int left, int right) {
        if(head==nullptr){
            return head;
        }
        stack<int> st;
        ListNode * tempHead = head;
        ListNode * leftNode;
        int index=1;
        int temp=0;
        while(tempHead!=nullptr){
            if(index==left){
                leftNode=tempHead;
                temp=1;
            }
            if(temp==1){
                st.push(tempHead->val);
            }
            if(index==right){
                break;
            }
            tempHead=tempHead->next;
            index++;
        }
        while(!st.empty()){
            leftNode->val=st.top();
            st.pop();
            leftNode=leftNode->next;
        }
        return head;
    }
};
~~~

####递归实现

双递归 思路：为了简化递归操作，反转a，b之间的节点修改为反转1到b-a+1之间的节点，对应递归起点后移多少位

~~~c++
class Solution {
    public:
    ListNode* reverseBetween(ListNode* head, int left, int right) {
        // 基本情况
        if (left == 1) {
            return reverseN(head, right);
        }
        // 前进到反转起点
        head->next = reverseBetween(head->next, left - 1, right - 1);
        return head;
    }
    private:
    ListNode* successor = nullptr;//用于记录right的下一个节点
    ListNode* reverseN(ListNode* head, int n) {
        if (n == 1) {
            successor = head->next;
            return head;
        }
        ListNode* newHead = reverseN(head->next, n - 1);
        head->next->next = head;
    }
};
~~~

###25 k个一组翻转链表

自解 0ms     

设置一个循环，终止条件是到达末尾，每次指针往后移动，对k取模，为0则传入当前指针和k进行反转(压栈)，反转时只对节点的值进行操作指针结构没有破坏

~~~c++
class Solution {
    public:
    ListNode* reverseKGroup(ListNode* head, int k) {
        //判空
        if(head==nullptr){
            return head;
        }
        //移动，根据条件调用函数
        auto temp = head;
        int count = 0;
        while(temp->next!=nullptr){
            //取余 调用函数-代表遍历了k个节点
            if(count%k==0){
                //调用函数
                reserve(temp,k);
            }
            temp=temp->next;
            count++;
        }
        return head;
    }
    private:
    void reserve(ListNode* head,int k){
        auto temp = head;
        stack<int> st;
        int sum = k;
        while(sum){
            sum--;
            if(temp==nullptr){ //当链表剩余不足k个直接返�?
                return;
            }
            st.push(temp->val);
            temp=temp->next;
        }
        temp = head;
        sum = k;
        while(sum){
            sum--;
            temp->val=(st.top());
            st.pop();
            temp=temp->next;
        }
    }
};
~~~

一开始我用压栈的方式只是改变了值，但是定眼一看题目要求是改变指针的指向，那就要修改一下了

真正反转�?

~~~c++
class Solution {
public:
    ListNode* reverseKGroup(ListNode* head, int k) {
        if (!head || k == 1) return head;
        
        // 创建虚拟头节�?
        ListNode* dummy = new ListNode(0);
        dummy->next = head;
        ListNode* pre = dummy;
        
        while (head) {
            // 检查剩余节点是否足够k�?
            ListNode* tail = pre;
            for (int i = 0; i < k; i++) {
                tail = tail->next;
                if (!tail) {
                    return dummy->next;
                }
            }
            
            // 保存下一组的起始位置
            ListNode* next = tail->next;
            
            // 反转k个节�?
            pair<ListNode*, ListNode*> result = reverseList(head, tail);
            head = result.first;
            tail = result.second;
            
            // 连接反转后的子链�?
            pre->next = head;
            tail->next = next;
            
            // 移动到下一�?
            pre = tail;
            head = next;
        }
        
        ListNode* ret = dummy->next;
        delete dummy;
        return ret;
    }

private:
    // 反转链表，返回新的头和尾
    pair<ListNode*, ListNode*> reverseList(ListNode* head, ListNode* tail) {
        ListNode* prev = tail->next;
        ListNode* curr = head;
        while (prev != tail) {
            ListNode* next = curr->next;
            curr->next = prev;
            prev = curr;
            curr = next;
        }
        return {tail, head};
    }
};
~~~

####递归

反转的递归思路

reverse传入 1->2->3->null  反转后 null<-1<-2<-3   newhead移动到最后 null改为tail 

reverse传入 1(head)->2->3(cur)->tail  tail<-1(head)<-2<-3(cur)  3->2->1->tail

~~~c++
class Solution {
public:
    ListNode* reverseKGroup(ListNode* head, int k) {
        //判空
        if(!head)   
            return head;
        //cur是区间的最后一个节�?
        auto cur = head;
        for(int i = 0; i < k-1; i++) {//移动cur并判断剩余节点是否满足k个
            if(!cur || !cur->next)  
                return head;
            cur = cur->next;
        }
        //next是区间最后一个节点的下一个节点
        auto next = cur->next;
        reverse(head, next); //(head，null)
        head->next = reverseKGroup(next, k);//head节点经过反转已经变成最后一个节点
        return cur;//返回的cur是第一个反转后的头或者没有经过反转的head
    }
    
    ListNode* reverse(ListNode* head, ListNode* tail) {
        if(head == tail || head->next == tail)
            return head;
        auto newHead = reverse(head->next, tail);
        head->next->next = head;
        head->next = tail;
        return newHead;
    }
};

~~~

###24 两两交换链表中的节点

//这道题不就是k个一组把k换成了2吗

第一遍自解 0ms 递归

其实也可以不用swap函数，我只是为了与k个一组保持一致

> 返回的是当前区域的最后一个节点也就是反转之后的头节点，反转之后head变成了最后一个节点，所以head->next=递归函数的返回值,不满足条件的都会在反转之前返回head

~~~c++
class Solution {
public:
    ListNode* swapPairs(ListNode* head) {
        //判空
        if(!head || !head->next) return head;
        //寻找当前区域的最后一个节点和尾节点
        auto end=head->next;
        auto tail=end->next;
        swap(head,tail);
        head->next=swapPairs(tail);
        return end;    
    }
    
    ListNode* swap(ListNode*head,ListNode*tail){
        if(head==tail||head->next==tail){
            return head;
        }
        auto temphead=swap(head->next,tail);
        head->next->next=head;
        head->next=tail;
        return temphead;
    }
};
~~~

## 删除链表元素

###83 删除链表中的重复元素

常规遍历思路

~~~c++
class Solution {
public:
    ListNode* deleteDuplicates(ListNode* head) {
        if(!head) return head;
        auto tempHead=head;
        int temp;
        while(tempHead->next){
            temp=tempHead->val;
            if(tempHead->next->val==temp){
                if(tempHead->next->next){
                    tempHead->next=tempHead->next->next;
                    //tempHead=tempHead->next;   如果下一个值与当前相同则还是在当前节点，避免连续相同
                }else{
                    tempHead->next=nullptr;
                }
            }else{
                tempHead=tempHead->next;
            }
        }
        return head;
    }
};
~~~

可以用两个节点遍历不用辅助变量来减少代码量

### 82 删除排序链表中的重复元素 II

第一遍自解 0ms

这里用的INT_MAX来避免与节点中的值重复，也可以换成循环中的临时变量，在判断有重复之后用一个while处理完所有重复

注意虚拟的节点用new

~~~c++
class Solution {
public:
    ListNode* deleteDuplicates(ListNode* head) {
        ListNode* virtual_head = new ListNode(0);
        virtual_head->next=head;

        int tempnum=INT_MAX;//记录这个重复值

        auto temphead=virtual_head;//用于遍历

        while(temphead->next!=nullptr){
            if(temphead->next->next!=nullptr){
                if(temphead->next->val==temphead->next->next->val){//处理连续两个相同
                    tempnum=temphead->next->val;
                    temphead->next=temphead->next->next->next;
                    continue;
                }
            }
            if(temphead->next->val==tempnum){//处理连续三个相同
                temphead->next=temphead->next->next;
                continue;
            }
            temphead=temphead->next;
        }
        return virtual_head->next;
    }
};

~~~

### 19 删除链表的倒数第N个节点

遍历两遍，第一遍找出个数，第二便删除
全部压栈，然后取出n+1个执行
双指针(只用遍历一遍)：两个间隔为n，当前面的到头删除第一个指针的下一个节点

~~~c++
class Solution {
public:
    ListNode* removeNthFromEnd(ListNode* head, int n) {
        // 创建虚拟头节点
        ListNode* dummy = new ListNode(0);
        dummy->next = head;
        // 定义快慢指针，都从虚拟头节点开始
        ListNode* fast = dummy;
        ListNode* slow = dummy;
        // 快指针先移动n+1步
        for(int i = 0; i <= n; i++) {
            if(fast == nullptr) return head;
            fast = fast->next;
        }    
        // 同时移动快慢指针，直到快指针到达末尾
        while(fast != nullptr) {
            fast = fast->next;
            slow = slow->next;
        }
        // 删除目标节点
        slow->next = slow->next->next;
        // 获取新的头节点
        ListNode* result = dummy->next;
        // 释放虚拟节点内存
        delete dummy;
        
        return result;
    }
};
~~~

## 合并链表

### 23 合并k个升序链表

自解6ms

使用优先队列，最小值优先，这里是存储的节点值，可以直接改为存储节点

~~~c++
class Solution {
public:
    ListNode* mergeKLists(vector<ListNode*>& lists) {
        auto tempHead=new ListNode(0);
        if(lists.empty()) return tempHead->next;
        //使用priority_queue的最小堆存储每一个节点
        priority_queue<int, vector<int>,greater<int>> proque;
        for(auto tempNode : lists){
            while(tempNode){
                proque.push(tempNode->val);
                tempNode=tempNode->next;
            }
        }
        auto temp=tempHead;
        while(!proque.empty()){
            temp->next = new ListNode(proque.top());
            proque.pop();
            temp=temp->next;
        }
        return tempHead->next;
    }
};
~~~

改为存储节点 3ms

~~~c++
class Solution {
public:
    struct CompareMyClass {  
        bool operator()(ListNode* lhs, ListNode* rhs) const {  
            return lhs->val > rhs->val; 
        }  
    };

    ListNode* mergeKLists(vector<ListNode*>& lists) {
        auto tempHead = new ListNode(0); 
        if (lists.empty()) return tempHead->next;

        priority_queue<ListNode*, vector<ListNode*>, CompareMyClass> proque; 

        // 将所有链表的头节点入队
        for (auto tempNode : lists) {
            if (tempNode) proque.push(tempNode);
        }

        auto temp = tempHead;
        while (!proque.empty()) {
            auto current = proque.top();
            proque.pop();
            temp->next = current;  // 拼接链表
            temp = temp->next;
            if (current->next) {
                proque.push(current->next);  // 如果有后续节点，继续入队
            }
        }

        return tempHead->next;
    }
};

~~~





# 树

##前序遍历

### 144 二叉树的前序遍历

看了思路后自解 

关键是nums要声明在外面

~~~c++
class Solution {
private:
    vector<int> nums;
public:
    vector<int> preorderTraversal(TreeNode* root) {
        if(!root) return nums;
        nums.push_back(root->val);
        preorderTraversal(root->left);
        preorderTraversal(root->right);
        return nums;
    }
};
~~~

#### 非递归实现

根节点入栈-循环遍历栈-栈非空，取出节点-取出节点值-压入右节点-压入左节点

~~~c++
class Solution {
public:
    vector<int> preorderTraversal(TreeNode* root) {
        if(!root) 
            return {};
        stack<TreeNode*> q; // 栈
        vector<int> ans;
        q.push(root);
        while(!q.empty()) {
            auto front = q.top();
            q.pop();
            ans.push_back(front->val);
            if(front->right)
                q.push(front->right);
            if(front->left)
                q.push(front->left);
        }
        return ans;
    }
};
~~~

### 112 路径总和

难点：遍历的基础上回退的时候要撤销累加值

用两个栈，一个栈压入节点 另一个栈压入节点和 比如1 2 3 节点压入1 取出1 压入3，2 值压入1 取出1 压入1+3，1+2

~~~c++
class Solution {
public:
    bool hasPathSum(TreeNode* root, int targetSum) {
        //sum的值满足且没有左右孩子则返回true
        stack<TreeNode*> st;
        stack<int> sum;
        if(root){
            st.push(root);
            sum.push(root->val);
        }
        while(!st.empty()){
            auto tempNode=st.top();
            st.pop();
            int temp=sum.top();
            sum.pop();
            if(temp==targetSum && !tempNode->left && !tempNode->right) return true;
            if(tempNode->right){
                st.push(tempNode->right);
                sum.push(temp+tempNode->right->val);
            }
            if(tempNode->left){
                st.push(tempNode->left);
                sum.push(temp+tempNode->left->val);
            }
        }
        return false;
    }
};
~~~

### 113 路径总和Ⅱ

在上一题的基础上再加一个stack<vector<int>> 与存储路径和的stack<int> stnum;思路一致即可

17ms 

~~~c++
class Solution {
public:
    vector<vector<int>> pathSum(TreeNode* root, int targetSum) {

        vector<vector<int>> vv; 
        if (!root) return vv;
        stack<vector<int>> v;
        v.push(vector<int>{root->val});

        stack<TreeNode*> stnode;
        stnode.push(root);

        stack<int> stnum;
        stnum.push(root->val);

        while(!stnode.empty()){
            auto tempNode=stnode.top();
            stnode.pop();

            int tempnum=stnum.top();
            stnum.pop();

            vector<int> currentPath = v.top();
            v.pop();

            if(!tempNode->left&&!tempNode->right){
                if(tempnum==targetSum) vv.push_back(currentPath);
            }

            if(tempNode->right){
                stnode.push(tempNode->right);

                stnum.push(tempnum+tempNode->right->val);

                vector<int> rightPath = currentPath;
                rightPath.push_back(tempNode->right->val);
                v.push(rightPath);
            }

            if(tempNode->left){
                stnode.push(tempNode->left);

                stnum.push(tempnum+tempNode->left->val);

                vector<int> leftPath = currentPath;
                leftPath.push_back(tempNode->left->val);
                v.push(leftPath);
            }
        }
        return vv;
    }
};
~~~

没有递归的效率高，优化：使用一个栈，减少赋值操作等

优化后的版本，仍然不是0ms，可见手动建栈效率不如递归栈的效率

~~~c++
class Solution {
public:
    vector<int> tempPath;

    vector<vector<int>> pathSum(TreeNode* root, int targetSum) {
        vector<vector<int>> result;
        if (!root) return result;

        stack<tuple<TreeNode*, vector<int>, int>> stk;
        stk.push({root, {root->val}, root->val});

        while (!stk.empty()) {
            auto [node, path, pathSum] = stk.top();
            stk.pop();

            if (!node->left && !node->right && pathSum == targetSum) {
                result.push_back(path);
            }

            if (node->right) {
                tempPath = path;
                tempPath.push_back(node->right->val);
                stk.push({node->right, tempPath, pathSum + node->right->val});
            }

            if (node->left) {
                tempPath = path;
                tempPath.push_back(node->left->val);
                stk.push({node->left, tempPath, pathSum + node->left->val});
            }
        }

        return result;
    }
};
~~~



#### 递归

~~~c++
class Solution {
public:
    vector<vector<int>> res;
    vector<int> tmp;

    vector<vector<int>> pathSum(TreeNode* root, int targetSum) {
        if(!root)
            return res;
        tmp.push_back(root->val);
        backtrack(root, targetSum-root->val);
        return res;
    }
    
    void backtrack(TreeNode* root, int sum) {
        if(sum == 0 && !root->left && !root->right) {
            res.push_back(tmp);
            return;
        }
        if(root->left) {
            tmp.push_back(root->left->val);
            backtrack(root->left, sum-root->left->val);
            tmp.pop_back();
        }
        if(root->right) {
            tmp.push_back(root->right->val);
            backtrack(root->right, sum-root->right->val);
            tmp.pop_back();
        }
    }
};
~~~

更好理解的递归：

accumulate函数计算vector的和

~~~c++
class Solution {
public:
    void dfs(TreeNode* root, int targetSum, vector<int>& currentPath, vector<vector<int>>& result) {
        if (!root) return;
        
        currentPath.push_back(root->val);  // 加入当前节点到路径
        
        // 如果到达叶节点并且路径和等于目标值
        if (!root->left && !root->right && accumulate(currentPath.begin(), currentPath.end(), 0) == targetSum) {
            result.push_back(currentPath);
        }
        
        // 递归访问左子树和右子树
        dfs(root->left, targetSum, currentPath, result);
        dfs(root->right, targetSum, currentPath, result);
        
        // 回溯，移除当前节点
        currentPath.pop_back();
    }
    
    vector<vector<int>> pathSum(TreeNode* root, int targetSum) {
        vector<vector<int>> result;
        vector<int> currentPath;
        
        dfs(root, targetSum, currentPath, result);
        
        return result;
    }
};
~~~

###437 路径总和 III

递归的时候使用减法可以减少一个变量的使用

开始的时候判空，则递归左右子树的时候无需判空

注意使用long long避免溢出

以每个节点为根节点分别遍历

~~~c++
class Solution {
public:
    int pathSum(TreeNode* root, int targetSum) {
        if (!root) return 0;
        return dfs(root, (long long)targetSum) + pathSum(root->left, targetSum) + pathSum(root->right, targetSum);
    }

    int dfs(TreeNode* root, long long targetSum) {
        if (!root) return 0;
        int res = 0;
        if (root->val == targetSum) res++;
        res += dfs(root->left, targetSum - root->val);
        res += dfs(root->right, targetSum - root->val); 
        return res;
    }
};
~~~

这样达不到0ms，因为有很多重复的遍历。所以使用前缀和

利用哈希表记录路径和的出现次数来避免重复计算

前缀和是从根节点到当前节点的路径和。用哈希表记录前缀和出现的次数，当当前节点的路径和减去targetSum的结果在哈希表中存在这个键则结果加上其对应的值

~~~c++
class Solution {
public:
    unordered_map<long long, int> umap = {{0,1}}; //前缀和 出现的次数
    int res = 0;
    
    void pathS(TreeNode* root, long long pre, int targetSum) {//pre记录当前路径和
        if (!root) return;
        pre += root->val;
        if (umap.count(pre-targetSum)) res += umap[pre-targetSum];//如果存在pre-targetSum=前缀和
        umap[pre] ++;
        pathS(root->left, pre, targetSum);
        pathS(root->right, pre, targetSum);
        umap[pre] --;//一个节点的子节点都遍历完之后则在hashmap中移除这个前缀和
    }

    int pathSum(TreeNode* root, int targetSum) {
        long long pre = 0;
        pathS(root, pre, targetSum);
        return res;
    }
};
~~~

###226 翻转二叉树

~~~c++
class Solution {
public:
    TreeNode* invertTree(TreeNode* root) {
        //从底往上交换先交换1 3 6 9 再交换2 7
        //所以使用递归从深处交换
        if(!root) return root;
        if(!root->left&&!root->right) return root;
        invertTree(root->left);
        invertTree(root->right);
        swap(root->left,root->right);
        return root;
    }
};
~~~

### 257 二叉树的所有路径(就看这个)

####递归版本

0ms

~~~c++
class Solution {
public:
    vector<string> result;
    string currentPath;
    void dfs(TreeNode* root, string& currentPath, vector<string>& result) {
        if (!root) return;
        
        // 保存当前路径长度，用于回溯
        int len = currentPath.length();
        
        // 如果不是空路径，需要添加箭头
        if (len > 0) {
            currentPath += "->";
        }
        currentPath += to_string(root->val);
        
        // 如果到达叶节点，将当前路径加入结果
        if (!root->left && !root->right) {
            result.push_back(currentPath);
        }
        
        // 递归访问左子树和右子树
        dfs(root->left, currentPath, result);
        dfs(root->right, currentPath, result);
        
        // 回溯，将路径恢复到进入当前节点之前的状态
        currentPath.resize(len);
    }
    
    vector<string> binaryTreePaths(TreeNode* root) {
        dfs(root, currentPath, result);
        return result;
    }
};

~~~

1. 使用全局的变量，减少递归时每次创建副本 从3ms优化到0ms

#### 非递归版本

0ms

~~~c++
class Solution {
public:
    vector<string> result;
    vector<string> binaryTreePaths(TreeNode* root) {
        if(!root) return result;
        stack<pair<TreeNode*, string >>stk; //存储节点(遍历)  路线
        stk.push({root, ""});

        while (!stk.empty()) {
            auto [node, path] = stk.top();
            stk.pop();

            if (!path.empty()) {
            path += "->";
            }
            path += to_string(node->val);

            if (!node->left && !node->right) {
                result.push_back(path);
            }

            if (node->right) {
                stk.push({node->right, path});
            }

            if (node->left) {
                stk.push({node->left, path});
            }
        }
        return result;
    }
};
~~~

# 动态规划

~~~
# 自顶向下递归的动态规划
def dp(状态1, 状态2, ...):
    for 选择 in 所有可能的选择:
        # 此时的状态已经因为做了选择而改变
        result = 求最值(result, dp(状态1, 状态2, ...))
    return result

# 自底向上迭代的动态规划
# 初始化 base case
dp[0][0][...] = base case
# 进行状态转移
for 状态1 in 状态1的所有取值：
    for 状态2 in 状态2的所有取值：
        for ...
            dp[状态1][状态2][...] = 求最值(选择1，选择2...)
~~~

## 64最小路径和

状态转移方程

dp\[i][j] = min(dp\[i - 1][j], dp\[i][j - 1]) + grid[i][j\]

当前最小路径为=以下两者的最小者：左边的最小路径加上当前值，上边的最小路径加上当前值

创建二维向量->初始化第一个元素->初始化第一行和第一列->公式推导

~~~c++
class Solution {
public:
    int minPathSum(vector<vector<int>>& grid) {
        int m = grid.size();//m行
        int n = grid[0].size();//n列
        vector<vector<int>> dp(m, vector<int>(n, 0));
        dp[0][0] = grid[0][0];
        // 初始化第一行
        for (int j = 1; j < n; ++j) {
            dp[0][j] = dp[0][j - 1] + grid[0][j];
        }
        // 初始化第一列
        for (int i = 1; i < m; ++i) {
            dp[i][0] = dp[i - 1][0] + grid[i][0];
        }
        // 填充 dp 数组
        for (int i = 1; i < m; ++i) {
            for (int j = 1; j < n; ++j) {
                dp[i][j] = min(dp[i - 1][j], dp[i][j - 1]) + grid[i][j];
            }
        }
        return dp[m - 1][n - 1];
    }
};
~~~

##647回文子串

1. 中心扩展法

遍历每个可能为中心的位置，一个为中心+两个为中心

~~~c++
class Solution {
public:
    int countSubstrings(std::string s) {
        int n = s.length();
        int count = 0;

        // 遍历每个可能的中心位置
        for (int i = 0; i < n; ++i) {
            // 以单个字符为中心扩展
            int left = i, right = i;
            while (left >= 0 && right < n && s[left] == s[right]) {
                ++count;
                --left;
                ++right;
            }

            // 以两个相邻字符为中心扩展
            left = i;
            right = i + 1;
            while (left >= 0 && right < n && s[left] == s[right]) {
                ++count;
                --left;
                ++right;
            }
        }

        return count;
    }
};
~~~



2. 动态规划

dp\[i][j] 表示字符串从索引 i 到索引 j 的子串是否为回文串

分类讨论：

1. i==j 是
2. j-1=i 如果s\[i]==s\[j] 
3. j-1>i s\[i]=\=s\[j] 且 dp\[i+1]\[j-1]==true

~~~c++
class Solution {
public:
    int countSubstrings(std::string s) {
        int n = s.length();
        std::vector<std::vector<bool>> dp(n, std::vector<bool>(n, false));
        int count = 0;

        // 枚举子串的长度
        for (int len = 0; len < n; ++len) {
            for (int i = 0; i < n - len; ++i) {
                int j = i + len;
                if (len == 0) {
                    dp[i][j] = true;
                } else if (len == 1) {
                    dp[i][j] = (s[i] == s[j]);
                } else {
                    dp[i][j] = (s[i] == s[j]) && dp[i + 1][j - 1];
                }

                if (dp[i][j]) {
                    ++count;
                }
            }
        }

        return count;
    }
};
~~~

##518 零钱兑换Ⅱ

valid[i]表示是否可以使用给定的硬币面额组合出金额   dp[i]表示凑齐i的组合数  初始化都为1

先检查是否能凑出目标金额，状态转移方程：valid[i] |= valid[i - coin];

计算组合数：dp[i] += dp[i - coin];

~~~c++
class Solution {
public:
    int change(int amount, vector<int>& coins) {
        vector<int> dp(amount + 1), valid(amount + 1);
        dp[0] = 1;
        valid[0] = 1;
        for (int& coin : coins) {
            for (int i = coin; i <= amount; i++) {
                valid[i] |= valid[i - coin];
            }
        }
        if (!valid[amount])
            return 0;
        for (int& coin : coins) {
            for (int i = coin; i <= amount; i++) {
                dp[i] += dp[i - coin];
            }
        }
        return dp[amount];
    }
};
~~~

~~~c++
class Solution {
public:
    int change(int amount, vector<int>& coins) {
        vector<uint64_t> dp(amount+1,0);
        dp[0]=1;
        for(int i=0;i<coins.size();i++){
            for(int j=coins[i];j<=amount;j++){
                dp[j]=dp[j]+dp[j-coins[i]];
            }
        }
        return dp[amount];
    }
};
~~~

#二分查找

关键：

int left = 0, right = nums.size() - 1;

while(left <= right) 

mid = left + (right - left) / 2;

left = mid + 1;

right = mid - 1; 

~~~c++
int binary_search(vector<int>& nums, int target) {
    int left = 0, right = nums.size() - 1; 
    while(left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] < target) {
            left = mid + 1;
        } else if (nums[mid] > target) {
            right = mid - 1; 
        } else if(nums[mid] == target) {
            // 直接返回
            return mid;
        }
    }
    // 直接返回
    return -1;
}
//找到 target 的最左侧索引
int left_bound(vector<int>& nums, int target) {
    int left = 0, right = nums.size() - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] < target) {
            left = mid + 1;
        } else if (nums[mid] > target) {
            right = mid - 1;
        } else if (nums[mid] == target) {
            // 别返回，锁定左侧边界
            right = mid - 1;
        }
    }
    // 判断 target 是否存在于 nums 中
    // 此时 target 比所有数都大，返回 -1
    if (left == nums.size()) return -1;
    // 判断一下 nums[left] 是不是 target
    return nums[left] == target ? left : -1;
}
////找到 target 的最右侧侧索引
int right_bound(vector<int>& nums, int target) {
    int left = 0, right = nums.size() - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] < target) {
            left = mid + 1;
        } else if (nums[mid] > target) {
            right = mid - 1;
        } else if (nums[mid] == target) {
            // 别返回，锁定右侧边界
            left = mid + 1;
        }
    }
    // 此时 left - 1 索引越界
    if (left - 1 < 0) return -1;
    // 判断一下 nums[left] 是不是 target
    return nums[left - 1] == target ? (left - 1) : -1;
}
~~~

排除不在区间

~~~c++
class Solution {
public:
    int searchInsert(vector<int>& nums, target) {
        int n = nums.size();
        int l = 0, r = n - 1;
        while(l < r) {	// 注意此时不能有等于了，因为我们这里是排除不可能区间
            int mid = (l+r)/2;
            if(nums[mid] < target)	// 这里不能判断等于的情况，因为我们用的排除思维，只需要排除目标一定不在的元素区间
                l = mid + 1;
            else r = mid;	// 这是nums[mid] >= target的情况，说明目标在mid及左边, 往左缩小
        }
        return nums[l] == target ? nums[l] : -1;	// 退出循环，要么找到，要么没找到，如果找到的话，left和right都指向它了
    }
};
~~~

# 回溯 DFS BFS

## 695 岛屿的最大面积

~~~c++
class Solution {
public:
    int ans, cur;
    vector<vector<int>> dirs = {{1,0},{-1,0},{0,1},{0,-1}};//四个方向
    int maxAreaOfIsland(vector<vector<int>>& grid) {
        int m = grid.size(), n = grid[0].size();
        for(int i = 0; i < m; i++) {
            for(int j = 0; j < n; j++) {
                if(!grid[i][j]) continue;
                cur = 1;
                dfs(grid, i, j, m, n);
                ans = max(cur, ans);
            }
        }
        return ans;
    }
    void dfs(vector<vector<int>>& grid, int x, int y, int m, int n) {
        grid[x][y] = 0;//遍历过的设为0
        for(auto& dir : dirs) {
            int nx = dir[0]+x, ny = dir[1]+y;//周围四个方向坐标
            if(nx < 0 || ny < 0 || nx >= m || ny >= n || !grid[nx][ny]) continue;//满足任意情况会跳过这个循环
            cur++;
            dfs(grid, nx, ny, m, n);
        }
    }
};
~~~

