## MySql千万级limit优化方案
优化方案：
1. 模仿百度、谷歌方案（前端业务控制）
类似于分段。我们给每次只能翻100页、超过一百页的需要重新加载后面的100页。这样就解决了每次加载数量数据大 速度慢的问题了

2. 记录每次取出的最大id， 然后where id > 最大id
`select * from table_name Where id > 最大id limit 10000, 10;`
这种方法适用于：除了主键ID等离散型字段外，也适用连续型字段datetime等
最大id由前端分页pageNum和pageIndex计算出来。

3. IN获取id
`select * from table_name Where id in (select id from table_name where ( user = xxx )) limit 10000, 10;`

4. join方式 + 覆盖索引（推荐）
参考：https://www.jianshu.com/p/efecd0b66c55
参考：https://blog.csdn.net/qq_41642932/article/details/81986835

`select * from table_name inner join ( select id from table_name where (user = xxx) limit 10000,10) b using (id)`

如果对于有where 条件，又想走索引用limit的，必须设计一个索引，将where 放第一位，limit用到的主键放第2位，而且只能select 主键！
`select id from test where pid = 1 limit 100000,10;`
索引：`alter table test add index idx_pid_id(pid, id)`

