![image-20220419142055695](assets/image-20220419142055695.png)

**执行步骤：**类似递归

1. sql第一段结果直接放入临时表（临时表**一**条数据）
2. sql第一段（子查询）和第二段sql关联查询
3. 关联查询结果放入临时表（**两**条数据）
4. 关联查询结果（子查询）和第二段sql关联查询
5. 关联查询结果放入临时表（**三**条数据）
6. 循环4，5，直到没有查询结果
7. as() 最后必须接一条sql，代表如何处理临时表

