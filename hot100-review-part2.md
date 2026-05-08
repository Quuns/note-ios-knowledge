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
```

# 
>
```cpp

```


# 
>
```cpp

```