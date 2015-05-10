---
layout: post
title:  test
date:   2014-03-24 02:16:21
tags:
- java 
---

###�㷨˼·###
_ʹ��ջ�������Ѿ����ʹ��ĸ��ڵ㣬Ϊ������ýڵ��������֮���ٷ�������������׼������ʼʱջΪ�գ��ÿ�����ָ��������ĸ��ڵ㡣���ţ���ָ����ڵ�����÷ǿջ���ջ�ǿգ������ѭ����ɲ�����_

####��###
```
        a
      /   \
     b     c
    / \	 /  \
   d   e f    h
  /            \
 g              i
```

###Node�ڵ㼰��ʼ��������###
```
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

}
```

###�㷨###
```
import java.util.Stack;

public class BinTreeTestCase {
	public static void main(String[] args) {
		Stack<Node> s = new Stack<>();
		Node node = Node.genBinTree();

		// �������
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