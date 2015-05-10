---
layout: post
title:  分页辅助类(java版)
date:   2014-03-27 13:47:25
tags:
- java 
---

<p>参照flask_sqlalchemy写了一个分页辅助类，主要吸收其分页工具栏中展示页号的算法。可配置左右展示的数目。</p>
<p>场景：40条数据，每页展示3条，共14页，当前展示第6页</p>
<p>分页工具条效果： 1 ... 4 5 <strong>6</strong> 7 8 ... 14</p>
<p>代码：</p>
<pre><code>import java.util.ArrayList;
import java.util.List;
/**
 * 
 * @filename:     Pagenation.java
 * @author:      tianjie 
 * @version:     1.0
 * @create time: 26 Mar 2014 14:59:44
 * @record
 */
public class Pagenation&lt;T&gt;  {
    private List&lt;T&gt; items;
    private int page, totalCount, pageSize = 10;

    //**********************
    public Pagenation(List&lt;T&gt; items, int page, int totalCount) {
        this.items = items;
        this.totalCount = totalCount;
        if(page &lt;= 0 || page &gt; this.getPages())
            throw new IllegalArgumentException(&quot;错误的页号&quot;);

        this.page = page;
    }

    public Pagenation(List&lt;T&gt; items, int page, int totalCount,int pageSize) {
        if(pageSize&lt;0)
            throw new IllegalArgumentException(&quot;页大小不能为0&quot;);
        this.items = items;
        this.totalCount = totalCount;
        this.pageSize = pageSize;

        if(page &lt;= 0 || page &gt; this.getPages()){
            this.page = 1;
        }else{
            this.page = page;
        }

    }

    // getter and setter

    /**
     * 默认展示最左（右）展示1个，中间展示5个，当前页居中
     * @see Pagenation#getIterPages(int, int, int, int) 
     */
    public List&lt;Integer&gt; getIterPages(){
        int left_edge=2, left_current=3,
                right_current=3, right_edge=2;
        return getIterPages(left_edge, left_current, right_current, right_edge);
    }

    /**
     * 返回分页工具条可用的页数&lt;br&gt;
     * 如没页显示10条数据，166条数据，当前第6页。则结果为：&lt;br&gt;
     * [1, null, 5, 6, 7, 8, 9, 10, null, 17]&lt;br&gt;
     * 结果中值为null的元素用省略号等字符表示
     * @see flask_sqlalchemy#Pagination.iter_pages
     * @return
     * @author tianjie
     * @date 26 Mar 2014 15:31:04
     * @version v1.0
     */
    public List&lt;Integer&gt; getIterPages(int left_edge, int left_current,
            int right_current, int right_edge) {
        List&lt;Integer&gt; ips = new ArrayList&lt;Integer&gt;();
        int last = 0;
        for (int num = 1; num &lt; this.getPages() + 1; num++) {
            if (num &lt; left_edge
                    || (num &gt; this.page - left_current - 1 &amp;&amp; num &lt; this.page
                            + right_current)
                    || num &gt; this.getPages() - right_edge) {
                if (last + 1 != num)
                    ips.add(null);
                else
                    ips.add(num);
                last = num;
            }
        }
        return ips;
    }

    /**
     * 总页数
     * @return
     */
    public int getPages(){
        int pages = 0;
        if (pageSize == 0)
            pages = 0;
        else
            pages = (this.totalCount + pageSize - 1) / pageSize;
        return pages;
    }

    public int getPageSize() {
        return pageSize;
    }

    public void setPageSize(int pageSize) {
        this.pageSize = pageSize;
    }

    public List&lt;T&gt; getItems() {
        return items;
    }

    public int getCurrentPage(){
        return this.page;
    }

    public boolean isHasFirst(){
        return this.getPages() &gt;= 1;
    }

    public boolean isHasPre(){
        return this.page &gt; 1;
    }

    public boolean isHasLast(){
        return this.page &lt; this.getPages();
    }

    public boolean isHasNext(){
        return this.page&lt;this.getPages();
    }

    public int getFirstPage(){
        return 1;
    }
    public int getLastPage(){
        return this.getPages();
    }

    public int getNextPage(){
        return this.page + 1;
    }
    public int getPrePage(){
        return this.page - 1;
    }

}
</code></pre>