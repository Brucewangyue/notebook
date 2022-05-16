## 同义词

经常会遇到夸库请求，当遇到夸库请求的时候

```sql
select * from [BPM_V6].[dbo].[BPMSysOUs]
```

而经常夸库的这个库名需要经常变换，如：

- 生成环境和测试环境的库名不同
- 测试环境有多套测试库，名字不同
- 版本升级，分发出多套以版本命名的库名

那么此时如果在存储环境、函数中直接写了[BPM_V6].[dbo].[BPMSysOUs]，如果库名换了，那就麻烦了，需要去修改每个涉及到夸库的存储过程和函数

那么此时同义词可以解决这个问题

```sql server
CREATE SYNONYM bpm_BPMSysOUs FOR [BPM_V6].[dbo].[BPMSysOUs];
```

```sql
select * from bpm_BPMSysOUs
```

那么以后夸库的那个库名BPM_V6换成了其他的，那么只需要统一修改同义词