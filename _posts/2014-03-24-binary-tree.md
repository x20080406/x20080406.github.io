---
layout: post
title:  先序遍历二叉树（非递归算法）
date:   2014-03-24 02:16:21
tags:
- java 
---

###算法思路###
_使用栈来保存已经访问过的根节点，为访问完该节点的左子树之后再访问其右子树做准备。开始时栈为空，用空引用指向二叉树的跟节点。接着，若指向根节点的引用非空或者栈非空，则继续循环完成操作。_

####树###
```java
			a
		  /   \
		 b     c
		/ \	 /  \
	   d   e f    h
	  /            \
	 g              i```

###Node节点及初始化树代码###

```java
	package personal.tianjie.datastruct;

	public class Node {
		private String nodeName;
		private Node left;
		private Node right;
		
		public Node(String nodeName, Node left, Node right) {
			super();
			this.nodeName = nodeName;
			this.left = left;
			this.right = right;
		}

		public String getNodeName() {
			return nodeName;
		}
		public void setNodeName(String nodeName) {
			this.nodeName = nodeName;
		}

		public Node getLeft() {
			return left;
		}

		public void setLeft(Node left) {
			this.left = left;
		}

		public Node getRight() {
			return right;
		}

		public void setRight(Node right) {
			this.right = right;
		}

		public static Node genBinTree(){
			
			Node g = new Node("g", null, null);
			Node d = new Node("d", g, null);
			Node b = new Node("b", d, e);
			Node i = new Node("i", null, null);
			Node h = new Node("h", null, i);
			Node f = new Node("f", null, null);
			Node c = new Node("c", f, h);

			Node a = new Node("a", b, c);

			return a;
		}

	}```

###算法###
```
import java.util.Stack;

public class BinTreeTestCase {
	public static void main(String[] args) {
		Stack<Node> s = new Stack<>();
		Node node = Node.genBinTree();

		// 先序遍历
		while (node != null || !s.empty()) {
			if (node != null) {
				System.out.println(node.getNodeName());
				s.push(node);
				node = node.getLeft();
			} else {
				node = s.pop().getRight();
			}
		}
	}

}
```