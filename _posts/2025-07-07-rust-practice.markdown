---
layout: post
title:  "初尝Rust"
date:   2025-07-07 20:34:00 +0800
---

## 重要概念

**Ownership and mutable reference**:
 * can have one mutable reference or multiple immutable references, not both

**Move by default**:
 * 变量赋值，行为是从老的变量move到新的变量
 * rust 的 move 和 c++ 的 move 不同：
    * move 后源变量就变成了废弃的内存，不再执行drop逻辑，只释放内存；
    * move 动作实际做的是裸内存copy，没有自定义move逻辑，因为源变量内存无效了，不做drop，就不存在残留指针引用的问题；

**mem::replace()/mem::take()**:
 * 成员函数如果要move走一个成员字段，因为持有self的mutable引用，move成员字段就是对self的partial move，会使得self失效, 所以不允许；
 * 此时需要move走的同时，把源变量替换为一个有效的值，mem::replace就是做这个事情。这有点像c++的move了。
 * mem::take() 是特殊的 mem::replace() ， 替换为变量类型的零值。特别的，Option<T>有个take()方法, 替换自己为None。
 * 这个虽然不算复杂，但给人感受很怪，这么常见的场景，却需要一个特殊的mem操作。

**自引用数据结构**:
 * 字段直接引用自己时，move后，自引用字段的地址值会是老的地址，变成无效引用；
 * 如果有自引用字段，通常是需要一个mutable引用，但本身在外面也会有一个可变引用，两个就没法共存；
 * 即使是间接引用也不行, 比如循环链表的尾节点的next指向head, 或者单链表头节点里保存一个尾指针;


## unsafe pointer 实践

链表挑战，在头节点维护一个尾指针，尾指针用raw pointer, next 分别用raw pointer 和 `Option<Box<>>` 实现了两个版本。

完整代码见 <https://github.com/fantasyzh/rust-test/tree/main/linked_list/src>

这里展示一下两个版本的diff：

```diff
--- head_tail_ptr.rs	2025-06-20 15:54:41
+++ head_tail.rs	2025-06-20 15:54:41
@@ -1,21 +1,21 @@
 use crate::mylist::MyList;
 
 pub struct List<T: std::fmt::Debug> {
     size: u64,
-    nodes:  *mut ListNode<T>,
+    nodes:  Option<Box<ListNode<T>>>,
     tail_ptr: *mut ListNode<T>
 }
 
 struct ListNode<T> {
     value: T,
-    next: *mut ListNode<T>
+    next: Option<Box<ListNode<T>>>
 }
 
 impl<T> ListNode<T> {
     fn new(v: T) -> ListNode<T> {
-        ListNode { value: v, next: ptr::null_mut() }
+        ListNode { value: v, next: None }
     }
 }
 
@@ -25,63 +25,56 @@
 
 impl<T: std::fmt::Debug> MyList<T> for List<T> {
     fn new() -> Self {
-        List { size: 0, nodes: ptr::null_mut(), tail_ptr: ptr::null_mut() }
+        List { size: 0, nodes: None, tail_ptr: ptr::null_mut() }
     }
 
     // link to tail
     fn push(&mut self, v: T) {
         println!("push {:#?}", v);
-        let new_node = Box::into_raw(Box::new(ListNode::new(v)));
+        let mut new_node = Box::new(ListNode::new(v));
+        let new_tail_ptr = &mut *new_node as *mut ListNode<T>;
 
-        if !self.nodes.is_null() {
+        if !self.tail_ptr.is_null() {
             assert!(self.size > 0);
             let tail_node = unsafe { &mut *self.tail_ptr };
-            tail_node.next = new_node;
-            self.tail_ptr = new_node;
+            tail_node.next = Some(new_node);
+            self.tail_ptr = new_tail_ptr;
         } else {
             // empty
             assert!(self.size == 0);
-            self.nodes = new_node;
-            self.tail_ptr = self.nodes;
+            self.nodes = Some(new_node);
+            self.tail_ptr = new_tail_ptr;
         }
         self.size += 1;
     }
 
     // pop from head
     fn pop(&mut self) -> Option<T> {
-        if self.nodes.is_null() {
-            None
-        } else {
-            let head = self.nodes;
-            let value = unsafe {
-                self.nodes = (*head).next;
-                let r = Box::from_raw(head);
-                let value = r.value;
-                //drop(r);
-                value
-            };
+        // XXX move from self will consume self, make self not usable, thus mem::replace
+        // XXX mem::replace for Option can be simplified as option.take()
+        //let nodes = mem::replace(&mut self.nodes, None);
+        //let nodes = self.nodes.take();
+        let head = self.nodes.take()?;
+        self.nodes = head.next;
 
-            self.size -= 1;
-            if self.nodes.is_null() {
-                self.tail_ptr = ptr::null_mut();
-            }
-            Some(value)
+        self.size -= 1;
+        if self.size == 0 {
+            self.tail_ptr = ptr::null_mut();
         }
+        Some(head.value)
     }
 
     fn visit(&self, f: impl Fn(&T)) {
         println!("start visit list nodes");
-        Self::visit_nodes(self.nodes, &f);
+        Self::visit_nodes(&self.nodes, &f);
     }
-
 }
 
 impl<T: std::fmt::Debug> List<T> {
-    fn visit_nodes(nodes: *const ListNode<T>, f: impl Fn(&T)) {
-        if !nodes.is_null() {
-            let head = unsafe { &(*nodes) };
+    fn visit_nodes(nodes: &Option<Box<ListNode<T>>>, f: impl Fn(&T)) {
+        if let Some(head) = nodes {
             f(&head.value);
-            Self::visit_nodes(head.next, f);
+            Self::visit_nodes(&head.next, f);
         }
     }
 
@@ -89,19 +82,16 @@
         Iter { 
             // XXX use as_deref to deref Box, return Option<&T::Target> by Deref trait
             //next: self.nodes.map(|n| &**n) 
-            next: if self.nodes.is_null() { None } else { unsafe { Some(&(*self.nodes)) } }
+            next: self.nodes.as_deref()
         }
     }
 }
 
 impl<T: std::fmt::Debug> Drop for List<T> {
     fn drop(&mut self) {
-        while !self.nodes.is_null() {
-            let head = self.nodes;
-            unsafe {
-                self.nodes = (*head).next;
-                drop(Box::from_raw(head));
-            }
+        let mut nodes = self.nodes.take();
+        while let Some(head) = &mut nodes {
+            nodes = head.next.take();
         }
     }
 }
@@ -111,7 +101,7 @@
 
     fn next(&mut self) -> Option<Self::Item> {
         self.next.map(|node| {
-            self.next = if node.next.is_null() { None } else { unsafe { Some(&(*node.next)) } };
+            self.next = node.next.as_deref();
             &node.value
         })
     }
```
