map
HashMap
jdk1.7：底层是Entry数组加链表
jdk1.8：底层是Node（继承Entry数组）数组加链表加红黑树，数组长度大于64，链表长度大于8，链表转红黑树（平衡的二叉搜索树，搜索效率高）

![[Pasted image 20250711143817.png]]

<div align="center"> <img src="Pasted image 20250711143817.png" width=""/> </div>


```
<div align="center"> <img src="Pasted image 20250711143817.png" width=""/> </div>
```