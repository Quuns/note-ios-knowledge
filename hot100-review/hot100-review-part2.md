# 链表相交
>
```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode *getIntersectionNode(ListNode *headA, ListNode *headB) {
        if (!headA || !headB) return nullptr;
        ListNode *x = headA;
        ListNode *y = headB;
        while (x != y) {
            x = (x ? x->next : headB);
            y = (y ? y->next : headA);
        }
        return x;
    }
};
```

# 反转链表
> 
```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        if (!head) return head;
        ListNode *pre = nullptr;
        while (head) {     
            ListNode *nxt = head->next;
            head->next = pre;
            pre = head;
            head = nxt;
        }
        return pre;
    }
};
```

# 回文链表
> 请你判断该链表是否为回文链表。如果是，返回 true ；否则，返回 false 
```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* middleNode(ListNode *head) {
        ListNode *slow = head;
        ListNode *fast = head;
        while (fast && fast->next) {
            fast = fast->next->next;
            slow = slow->next;
        }
        return slow;
    }
    ListNode* reverse(ListNode *head) {
        if (head->next == nullptr) {
            return head;
        }
        ListNode *resHead;
        resHead = reverse(head->next);
        ListNode *nxt = head->next;
        nxt->next = head;
        head->next = nullptr;
        return resHead;
    }
    bool isPalindrome(ListNode* head) {
        ListNode *mid = middleNode(head);
        ListNode *head2 = reverse(mid);
        while (head2) {
            if (head2->val != head->val) {
                return false;
            }
            head = head->next;
            head2 = head2->next;
        }
        return true;
    }
};
```

# 环形链表 I
>
```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    bool hasCycle(ListNode *head) {
        // 快慢指针
        ListNode *fast = head;
        while (fast && fast->next) {
            fast = fast->next->next;
            head = head->next;
            if (fast == head) {
                return true;
            }
        }
        return false;
    }
};
```

# 环形链表 II
>
```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode *detectCycle(ListNode *head) {
        ListNode *fast = head;
        ListNode *slow = head;
        while (fast && fast->next) {
            fast = fast->next->next;
            slow = slow->next;
            if (fast == slow) {
                while (head != slow) {
                    head = head->next;
                    slow = slow->next;
                }
                return slow;
            }
        }
        return NULL;
    }
};
```

# 合并两个有序链表
> 递归
```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* mergeTwoLists(ListNode* list1, ListNode* list2) {
        if (!list1) {
            return list2;
        }
        if (!list2) {
            return list1;
        }
        if (list1->val <= list2->val) {
            // list1表头 的值小于等于 list2表头的值，list1表头拆出来
            list1->next = mergeTwoLists(list1->next, list2);
            return list1;
        } else {
            // list1表头 的值大于 list2表头的值，list2表头拆出来
            list2->next = mergeTwoLists(list1, list2->next);
            return list2;
        }
    }
};
```

# 两数相加
> 链表相加
```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
        // 输入：l1 = [9,9,9,9,9,9,9], l2 = [9,9,9,9]
        // 输出：[8,9,9,9,0,0,0,1]
        // 输入：l1 = [2,4,3], l2 = [5,6,4]
        // 输出：[7,0,8]
        // 解释：342 + 465 = 807.
        ListNode* L1 = l1;
        ListNode* L2 = l2;
        ListNode* pre = nullptr;
        ListNode* head;
        int sum = 0;
        while(L1 && L2) {
            int val = L1->val + L2->val + sum;
            sum = val / 10;
            if (pre) {
                pre->next = new ListNode(val % 10);
                pre = pre->next;
            } else {
                pre = new ListNode(val % 10);
                head = pre;
            }
            L1 = L1->next;
            L2 = L2->next;
        }
        while (L1) {
            int val = L1->val + sum;
            sum = val / 10;
            pre->next = new ListNode(val % 10);
            pre = pre->next;
            L1 = L1->next;
        }
        while (L2) {
            int val = L2->val + sum;
            sum = val / 10;
            pre->next = new ListNode(val % 10);
            pre = pre->next;
            L2 = L2->next;
        }
        if (sum) {
            pre->next = new ListNode(sum);
            pre = pre->next;
        }
        return head;
    }
};
```

# 删除链表的倒数第N个节点
> 快慢指针
```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* removeNthFromEnd(ListNode* head, int n) {
        // 输入：head = [1,2,3,4,5], n = 2
        // 输出：[1,2,3,5]
        ListNode *pos = head;
        ListNode *del = head;
        ListNode *pre = nullptr;
        while (n) {
            pos = pos->next;
            n--;
        }

        while (pos) {
            pos = pos->next;
            pre = del;
            del = del->next;
        }
        if (pre) {
            pre->next = del->next;
            return head;
        } else {
            return head->next;
        }
    }
};
```

# 两两交换链表中的节点
> 递归
```cpp
class Solution {
public:
    //输入：head = [1,2,3,4]
    //输出：[2,1,4,3]
    ListNode* swapPairs(ListNode* head) {
        if (!head) return nullptr;
        ListNode* a = head;
        ListNode* b = head->next;
        if (!b) {
            return a;
        }
        ListNode* c = b->next;
        b->next = a;
        a->next = swapPairs(c);
        return b;
    }
};
```

# **K 个一组翻转链表**
> 递归
```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* reverse(ListNode* head, int step, int n) {
        if (step == n - 1) {
            return head;
        }
        ListNode* p = reverse(head->next, step + 1, n);
        head->next->next = head;
        head->next = nullptr;
        return p;
    }
    ListNode* reverseKGroup(ListNode* head, int k) {
        int sz = 0;
        if (k == 1) return head;
        for (auto p = head; p; p = p->next) {
            sz++;
        }
        int cnt = sz / k;
        ListNode* tmp = head;
        ListNode* pre = nullptr;
        ListNode* pretail = nullptr;
        ListNode* c = new ListNode();
        ListNode* ans = nullptr;
        int idx = 0;
        for (int i = 1; i <= sz; ++i) {
            ListNode *nxt = tmp->next;
            if ((idx % k) == k - 1) {
                c = reverse(pre, 0, k);
                if (!ans) ans = c;
                if (pretail != nullptr) pretail->next=c;
                pretail = pre;
            } else if (idx % k == 0) {
                if (pretail != nullptr) pretail->next=tmp;
                pre = tmp;
            }
            tmp = nxt;
            idx++;
        }
        return ans;
    }
};


// 2

class Solution {
public:
    ListNode* reverseKGroup(ListNode* head, int k) {
        // 统计节点个数
        int n = 0;
        for (ListNode* cur = head; cur; cur = cur->next) {
            n++;
        }

        ListNode dummy(0, head);
        ListNode* p0 = &dummy;
        ListNode* pre = nullptr;
        ListNode* cur = head;

        // k 个一组处理
        for (; n >= k; n -= k) {
            for (int i = 0; i < k; i++) { // 同 92 题
                ListNode* nxt = cur->next;
                cur->next = pre; // 每次循环只修改一个 next，方便大家理解
                pre = cur;
                cur = nxt;
            }

            // 见视频
            ListNode* nxt = p0->next;
            p0->next->next = cur;
            p0->next = pre;
            p0 = nxt;
        }
        return dummy.next;
    }
};
```

# 随机链表的复制
>
```cpp
/*
// Definition for a Node.
class Node {
public:
    int val;
    Node* next;
    Node* random;
    
    Node(int _val) {
        val = _val;
        next = NULL;
        random = NULL;
    }
};
*/

class Solution {
public:
    Node* copyRandomList(Node* head) {
        // 相邻copy
        for (auto p = head; p; p = p->next->next) {
            Node *nxt = p->next;
            p->next = new Node(p->val);
            p->next->next = nxt;
        }

        // random复制
        for (auto p = head; p; p = p->next->next) {
            if (p->random) {
                p->next->random = p->random->next;
            }
        }

        Node* newHead = NULL;
        for (auto p = head; p;) {
            // 新节点
            Node* curNew = p->next;
            if (!newHead) {
                newHead = curNew;
            }
            Node* nxt = curNew->next;
            if (nxt) curNew->next = nxt->next;
            p->next = nxt;
            p = nxt;
        }
        return newHead;
        
    }
};
```


# 排序链表
> 归并排序
```cpp
class Solution {
    // 876. 链表的中间结点（快慢指针）
    ListNode* middleNode(ListNode* head) {
        ListNode* pre = head;
        ListNode* slow = head;
        ListNode* fast = head;
        while (fast && fast->next) {
            pre = slow; // 记录 slow 的前一个节点
            slow = slow->next;
            fast = fast->next->next;
        }
        pre->next = nullptr; // 断开 slow 的前一个节点和 slow 的连接
        return slow;
    }

    // 21. 合并两个有序链表（双指针）
    ListNode* mergeTwoLists(ListNode* list1, ListNode* list2) {
        ListNode dummy; // 用哨兵节点简化代码逻辑
        ListNode* cur = &dummy; // cur 指向新链表的末尾
        while (list1 && list2) {
            if (list1->val < list2->val) {
                cur->next = list1; // 把 list1 加到新链表中
                list1 = list1->next;
            } else { // 注：相等的情况加哪个节点都是可以的
                cur->next = list2; // 把 list2 加到新链表中
                list2 = list2->next;
            }
            cur = cur->next;
        }
        cur->next = list1 ? list1 : list2; // 拼接剩余链表
        return dummy.next;
    }

public:
    ListNode* sortList(ListNode* head) {
        // 如果链表为空或者只有一个节点，无需排序
        if (head == nullptr || head->next == nullptr) {
            return head;
        }
        // 找到中间节点 head2，并断开 head2 与其前一个节点的连接
        // 比如 head=[4,2,1,3]，那么 middleNode 调用结束后 head=[4,2] head2=[1,3]
        ListNode* head2 = middleNode(head);
        // 分治
        head = sortList(head);
        head2 = sortList(head2);
        // 合并
        return mergeTwoLists(head, head2);
    }
};

```


# 合并K个排序链表
> 分治 + 合并两个有序链表
```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* merge2List(ListNode* a, ListNode *b) {
        if (!a) {
            return b;
        } 
        if (!b) {
            return a;
        }
        ListNode *head;
        if (a->val > b->val) {
            head = b;
            head->next = merge2List(a, b->next);
        } else {
            head = a;
            head->next = merge2List(a->next, b);
        }
        return head;
    }
    ListNode* mergeKLists(vector<ListNode*>& lists) {
        int sz = lists.size();
        if (sz == 0) return nullptr;
        auto dfs = [&](this auto && dfs, int l, int r) -> ListNode*  {
            if (l == r) {
                return lists[l];
            }
            int mid = (l + r) / 2;
            ListNode *left = dfs(l, mid);
            ListNode *right = dfs(mid + 1, r);
            return merge2List(left, right);
        };
        return dfs(0, sz - 1);
    }
};
```


# LRU缓存
> 双向链表 + 哈希表
```cpp
struct Node {
    Node *pre;
    Node *nxt;
    int key;
    int val;
    Node (int _key, int _val) {
        val = _val;
        key = _key;
        pre = nullptr;
        nxt = nullptr;
    }
};
class LRUCache {
    Node* dummy;
    unordered_map<int, Node *>mp;
    int LRUCapacity;
    void remove(Node *x) {
        // 将指针x移除
        x->pre->nxt = x->nxt;
        x->nxt->pre = x->pre;
        // debugg(1);
    }
    void push(Node *x) {
        // 将指针x移动动dummy后第一个位置上 头插法
        x->pre = dummy;
        x->nxt = dummy->nxt;
        dummy->nxt->pre = x;
        dummy->nxt = x;
        // debugg(2);
    }
    void set_to_head(Node *x) {
        // 将一个指针设置到头 先删除，再push
        remove(x);
        push(x);
    }
    void debugg(int x) {
        cout << x << "----"<<endl;
        Node *tmp = dummy;
        while(tmp->nxt != dummy) {
            tmp = tmp->nxt;
            cout << tmp->key << " ";
        }
        cout << endl;
    }
public:
    LRUCache(int capacity) {
        LRUCapacity = capacity;
        dummy = new Node(-1, 0);
        dummy->nxt = dummy;
        dummy->pre = dummy;
    }
    
    int get(int key) {
        if (!mp.contains(key)) {
            // 不在缓存中
            return -1;
        }
        Node *tmp = mp[key];
        // 推到插头处
        set_to_head(tmp);
        return tmp->val;
    }
    
    void put(int key, int value) {
        if (!mp.contains(key)) {
            // key 不存在
            Node *tmp = new Node(key, value);
            push(tmp);
            mp[key] = tmp;
            if (mp.size() > LRUCapacity) {
                // 删除末尾的元素 dummy的前一个元素是最末尾的
                Node *tmp = dummy->pre;
                remove(tmp);
                mp.erase(mp.find(tmp->key));// 在哈希表中删除对应节点key
                delete tmp;
            }
        } else {
            // key 存在-更新value-更新链表
            Node *tmp = mp[key];
            tmp->val = value;
            set_to_head(tmp);// 把对应key的Node节点放到最前面
        }
    }
};

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache* obj = new LRUCache(capacity);
 * int param_1 = obj->get(key);
 * obj->put(key,value);
 */
```

# 
>
```cpp

```