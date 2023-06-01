设置map的个数！你设置的map个数为什么没生效？

参考大佬链接：https://hero78.blog.csdn.net/article/details/99438121



最牛批的oom调优学习

看大佬博客：https://hero78.blog.csdn.net/article/details/80143559



1. 大数据量的情况，用group by + count(1) 替换 count distinct 
2. map join
3. map aggr  map端进行聚合
4. join的时候，副表的过滤条件写在join条件中或者子查询中，避免放在where后形成全关联
5. 多表关联的时候起码需要保证其中一张表的key是唯一的，防止笛卡尔积
6. 有数据倾斜的情况可以考虑，把倾斜的key单独join
7. 行转列的时候，0特别多





