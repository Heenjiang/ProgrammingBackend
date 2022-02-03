Leetcode 刷题记录

说明：每天刷3个题（两个medium，一个easy或者hard），本文档只有题号和描述，代码实现（带*号）

### num 3. [medium]

**Longest Substring Without Repeating Characters(最长字符串)**

解题关键词：**HashSet，滑动窗口**

**描述**：Given a string s, find the length of the longest substring without repeating characters.

 Example 1:

Input: s = "abcabcbb"
Output: 3
Explanation: The answer is "abc", with the length of 3.
Example 2:

Input: s = "bbbbb"
Output: 1
Explanation: The answer is "b", with the length of 1.
Example 3:

Input: s = "pwwkew"
Output: 3
Explanation: The answer is "wke", with the length of 3.
Notice that the answer must be a substring, "pwke" is a subsequence and not a substring.

**题解思路**;滑动窗口法，设想在一个字符串中，我们设k是当前最长子字符串的开始下表，r是最长子字符串的最后一个字符的下标；此时我们将k移动到k+1，那么相对应的我们的r也可以向后至少移动一位（因为我们的子字符串中是不能有重复字符的），而且我们的r还可以继续往后移动（移动多少取决于是否碰到跟k+1相同的字符），当k移动到字符串末尾的时候，此时最长的一个子字符串就是答案。示例如下;

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220116215617825.png" alt="image-20220116215617825" style="zoom:50%;" />

1. 我们可以用两个指针，指向字符串子串的开始和结尾（窗口）

2. 每一次我们将左指针向右移动一位，此时我们不断尝试将右指针向右移动，同时检查是否重复了左指针指向的字符。（这里也是我们需要引入set数据结构来去重的地方）
3. 当左指针超过（原字符串长度-此时最长子字符串长度）时，就结束，返回此时最长的子字符串



### num 146. [medium]

**Design a data structure that follows the constraints of a Least Recently Used (LRU) cache.**

解题关键词：**双向链表，HashMap**

Implement the LRUCache class:

LRUCache(int capacity) Initialize the LRU cache with positive size capacity.
**int get(int key)** Return the value of the key if the key exists, otherwise return -1.
**void put(int key, int value)** Update the value of the key if the key exists. Otherwise, add the key-value pair to the cache. If the number of keys exceeds the capacity from this operation, evict the least recently used key.

**The functions get and put must each run in O(1) average time complexity.**

Example 1:

Input
["LRUCache", "put", "put", "get", "put", "get", "put", "get", "get", "get"]
[[2], [1, 1], [2, 2], [1], [3, 3], [2], [4, 4], [1], [3], [4]]
Output
[null, null, null, 1, null, -1, null, -1, 3, 4]

Explanation
LRUCache lRUCache = new LRUCache(2);
lRUCache.put(1, 1); // cache is {1=1}
lRUCache.put(2, 2); // cache is {1=1, 2=2}
lRUCache.get(1);    // return 1
lRUCache.put(3, 3); // LRU key was 2, evicts key 2, cache is {1=1, 3=3}
lRUCache.get(2);    // returns -1 (not found)
lRUCache.put(4, 4); // LRU key was 1, evicts key 1, cache is {4=4, 3=3}
lRUCache.get(1);    // return -1 (not found)
lRUCache.get(3);    // return 3
lRUCache.get(4);    // return 4

首先我们确认LRU中需要数据先进先出，所以满足queue的特质，但是同时对刚使用的数据需要重新入队（先将节点入队然后删除原来的节点），而且队列的长度是一定的（需要及时删除头部节点）。然后最重要的性质：在put和get时需要O(1) average time complexity。我们联想到put和get时都需要查找元素，那么其实HashMap是最合适的，我们在HashMap中使用queue中节点的引用作为value，put和get操作的调用参数作为key。那么queue底层用什么实现呢？其实我们可以在此处用doubleLinkedList（方便删除节点）来作为queue的底层实现，同时我们维护一个双向链表的伪头部和尾部（便于出队和入队）。

具体操作：

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220116225642065.png" alt="image-20220116225642065" style="zoom:50%;" />

### num 206. [easy]

Reverse Linked List

Given the head of a singly linked list, reverse the list, and return the reversed list.

Example 1:

![img](https://assets.leetcode.com/uploads/2021/02/19/rev1ex1.jpg)

Input: head = [1,2,3,4,5]
Output: [5,4,3,2,1]

**解题关键字：1. 迭代；2. 递归；3. 栈**

迭代: 传入参数为链表的head。

1. current为当前正在遍历的结点，pre为current的前驱结点（刚开始时为null），next为current的后驱结点；

2. 将current.next 赋值为pre(反转)；

3. 然后移动pre,current,next三个结点 （遍历下一个结点）
4. 重复上述过程，直到current为null，然后返回pre结点（新的头结点）

代码：

```java
class Solution {
    public ListNode reverseList(ListNode head) {
        ListNode prev = null;
        ListNode curr = head;
        while (curr != null) {
            ListNode next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
        }
        return prev;
    }
}
```



### num 25. [hard]

**Reverse nodes in K-group**

description:Given the head of a linked list, reverse the nodes of the list k **at a time,** and return the modified list.

k is a positive integer and is **less than or equal to the length of the linked list**. **If the number of nodes is not a multiple of k then left-out nodes, in the end, should remain as it is.**

You may not alter the values in the list's nodes, only nodes themselves may be changed.

解题关键字：反转链表，分组

其实这个题在前面反转链表的基础上并不难，但是需要处理的细节比较多（链表的题目，细节处理很重要，稍不注意就会存在环），我们的思路同样是将链表按k个节点数分为一组，每一组都去反转链表，但是当反转完成后，我们需要将反转后的子链表接到原始链表中，所以我们调用反转链表方法时可以直接返回反转后的hea和tail节点，当然我们也需要记住子链表在原始链表的前驱和后驱节点，然后链接上去就行了。当我们遇到不足k个节点时候，直接返回新的head节点就行了。

代码;

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode reverseKGroup(ListNode head, int k) {
        if (head == null || head.next == null){
            return head;
        }
        //定义一个假的节点。
        ListNode dummy=new ListNode(0);
        //假节点的next指向head。
        // dummy->1->2->3->4->5
        dummy.next=head;
        //初始化pre和end都指向dummy。pre指每次要翻转的链表的头结点的上一个节点。end指每次要翻转的链表的尾节点
        ListNode pre=dummy;
        ListNode end=dummy;

        while(end.next!=null){
            //循环k次，找到需要翻转的链表的结尾,这里每次循环要判断end是否等于空,因为如果为空，end.next会报空指针异常。
            //dummy->1->2->3->4->5 若k为2，循环2次，end指向2
            for(int i=0;i<k&&end != null;i++){
                end=end.next;
            }
            //如果end==null，即需要翻转的链表的节点数小于k，不执行翻转。
            if(end==null){
                break;
            }
            //先记录下end.next,方便后面链接链表
            ListNode next=end.next;
            //然后断开链表
            end.next=null;
            //记录下要翻转链表的头节点
            ListNode start=pre.next;
            //翻转链表,pre.next指向翻转后的链表。1->2 变成2->1。 dummy->2->1
            pre.next=reverse(start);
            //翻转后头节点变到最后。通过.next把断开的链表重新链接。
            start.next=next;
            //将pre换成下次要翻转的链表的头结点的上一个节点。即start
            pre=start;
            //翻转结束，将end置为下次要翻转的链表的头结点的上一个节点。即start
            end=start;
        }
        return dummy.next;


    }
    //链表翻转
    // 例子：   head： 1->2->3->4
    public ListNode reverse(ListNode head) {
         //单链表为空或只有一个节点，直接返回原单链表
        if (head == null || head.next == null){
            return head;
        }
        //前一个节点指针
        ListNode preNode = null;
        //当前节点指针
        ListNode curNode = head;
        //下一个节点指针
        ListNode nextNode = null;
        while (curNode != null){
            nextNode = curNode.next;//nextNode 指向下一个节点,保存当前节点后面的链表。
            curNode.next=preNode;//将当前节点next域指向前一个节点   null<-1<-2<-3<-4
            preNode = curNode;//preNode 指针向后移动。preNode指向当前节点。
            curNode = nextNode;//curNode指针向后移动。下一个节点变成当前节点
        }
        return preNode;

    }


}
```



### 