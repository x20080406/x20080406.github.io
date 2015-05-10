---
layout: post
title:  先序遍历二叉树（非递归算法）
date:   2014-03-24 02:16:21
tags:
- java 
---
<hr />
<p>layout: post
title:  先序遍历二叉树（非递归算法）
date:   2014-03-24 02:16:21
tags:
- java </p>
<hr />
<h3 id="">算法思路</h3>
<p><em>使用栈来保存已经访问过的根节点，为访问完该节点的左子树之后再访问其右子树做准备。开始时栈为空，用空引用指向二叉树的跟节点。接着，若指向根节点的引用非空或者栈非空，则继续循环完成操作。</em></p>
<pre><code>function test() {
  console.log(&quot;notice the blank line before this function?&quot;);
}
</code></pre>

<h4 id="_1">树</h4>
<pre><code class="java">            a
          /   \
         b     c
        / \  /  \
       d   e f    h
      /            \
     g              i
</code></pre>

<h3 id="node">Node节点及初始化树代码</h3>
<pre><code class="java">    package personal.tianjie.datastruct;
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
            Node g = new Node(&quot;g&quot;, null, null);
            Node d = new Node(&quot;d&quot;, g, null);
            Node b = new Node(&quot;b&quot;, d, e);
            Node i = new Node(&quot;i&quot;, null, null);
            Node h = new Node(&quot;h&quot;, null, i);
            Node f = new Node(&quot;f&quot;, null, null);
            Node c = new Node(&quot;c&quot;, f, h);
            Node a = new Node(&quot;a&quot;, b, c);
            return a;
        }
    }
</code></pre>

<h3 id="_2">算法</h3>
<pre><code class="java">import java.util.Stack;
public class BinTreeTestCase {
    public static void main(String[] args) {
        Stack&lt;Node&gt; s = new Stack&lt;&gt;();
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
</code></pre>