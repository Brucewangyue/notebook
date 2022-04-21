## While循环的存储过程

```sql
Create proc [dbo].[proc_get_company_by_account]
	@Account nvarchar(20)
as
    declare @T int
    -- 表变量，存储多行数据
    declare @account_ou_id_list table( id nvarchar(50))
    declare @account_ou_id_list_count int
    declare @current_id nvarchar(50)
    -- 结果集
    declare @result table(OUID int,OUName nvarchar(90),Code nvarchar(50))
begin
	-- 获取用户的直接所在ou的id列表
	insert @account_ou_id_list 
	Select OUID 
	from bpm_BPMSysOUMembers
	where UserAccount = @Account and LeaderTitle is not null;

	-- list总数
	Select @account_ou_id_list_count=count(*) 
	from @account_ou_id_list

	-- 遍历list
	while @account_ou_id_list_count > 0
	begin
		-- 取首行
		select top 1 @current_id = id from @account_ou_id_list;

		with cte_sql(OUID,ParentOUID,OULevel,OUName,Code)
		as(
			select OUID,ParentOUID,OULevel,OUName,Code 
			from bpm_BPMSysOUs
			where OUID = @current_id

			union all

			select t1.OUID,t1.ParentOUID,t1.OULevel,t1.OUName,t1.Code 
			from bpm_BPMSysOUs t1
			join cte_sql t2 on t1.OUID = t2.ParentOUID 
			where t2.OULevel not in('控股集团','子集团','公司')
		)
		insert into @result 
		select OUID,OUName,Code
		from cte_sql 
		where OULevel in('控股集团','子集团','公司')

		-- 删除当前处理行
		delete from @account_ou_id_list where id = @current_id;
		-- 总行数减1
		set @account_ou_id_list_count = @account_ou_id_list_count -1;
	end
	
	select * from @result;
end
GO
```



## 存储过程调用存储过程

动态调用SQL

- 不支持表变量，因为表变量的作用域和exec()内的作用域不同

临时表

- 本地临时表：`#`开头，作用域为一个connection连接
  - 使用线程池是否会有并发问题：不会，因为线程池的一个连接的多个请求是单线程的，有序的
- 全局临时表：`##`开头，作用局是数据库，一般用作缓存

```sql
Create proc [dbo].[proc_get_company_by_account_from_cw]
	@in_account nvarchar(20)
as
	declare @is_admin int
begin
	-- 判断是否是财务管理员
	select top 1 @is_admin = 1
	from bpm_BPMSecurityGroupMembers t1
	join bpm_BPMSysUsers t2 on t1.SID = t2.SID
	where GroupName = '财务凭证管理组' and t2.Account = @in_account
	--print @is_admin

	declare @sql nvarchar(500)= 'select distinct UnitName,UnitCode from (
			select UnitName,UnitCode from CW_Expenses where UnitName is not null group by UnitName,UnitCode
			union
			select UnitName,UnitCode from CW_Payment where UnitName is not null group by UnitName,UnitCode
			union
			select UnitName,UnitCode from CW_TravelExpenses where UnitName is not null group by UnitName,UnitCode
        ) t ';

	if @is_admin is null
	begin
		print '非管理员执行'
		-- 临时表：解决表变量的问题，临时表效率比表变量高很多
		create table #account_company_list (OUID int,OUName nvarchar(90),Code nvarchar(50))
		-- 存储过程结果放入临时表
		insert #account_company_list
	    exec proc_get_company_by_account @in_account

		set @sql += 'where t.UnitCode in (select code from #account_company_list)'
	end

	set @sql += ' order by UnitCode asc'
	--print @sql

	exec(@sql)
	-- 删除临时表
	if @is_admin is null
		drop table #account_company_list
end
GO
```

