```xml
<?xml version="1.0"?>

<XForm>
   <Header> 
       <Method>Post</Method>
       <ProcessName>【城投版】付款申请(1)</ProcessName>
       <ProcessVersion>1.0</ProcessVersion>
       <DraftGuid></DraftGuid>
       <OwnerMemberFullName></OwnerMemberFullName>
       <Action>提交</Action>
       <Comment></Comment>
       <UrlParams></UrlParams>
       <ConsignEnabled>false</ConsignEnabled>
       <ConsignUsers>[]</ConsignUsers>
       <ConsignRoutingType>Parallel</ConsignRoutingType>
       <ConsignReturnType>Return</ConsignReturnType>
       <InviteIndicateUsers>[]</InviteIndicateUsers>
       <Context>{&quot;Routing&quot;:{}}</Context>
   </Header>

   <FormData>

   </FormData>
</XForm>
```

- XForm 根节点
- Header 头节点：存放BPM流程发起信息
- FormData 表单数据节点：存放流程中的表单数据



### Header

- ProcessName 需要发起的流程名称（必填）：
- ProcessVersion 流程版本（必填）：
- Comment 意见栏 ：可以不写
- OwnerMemberFullName 流程拥有人：这个是通过excel导入时自动配置的，不需要手写
  - 这个是流程拥有人，如果新建任务时另外配置了流程发起人，那么流程中会显示为流程发起人代流程拥有人发



### FormData

这个节点下的每一个子节点就是一张表，有主表和可重复表(Repeat="true")

#### 主表节点

主表只有一个唯一名字的节点，必须在节点上配置 BpmAccount BpmDepartmentName 用于指示使用哪些列作为参数查询BPM用户信息

- BpmAccount：Bpm账号子节点名
- BpmDepartmentName：Bpm账号所属部门子节点名

**两个属性配置确定一个用户和他的流程发起职位**

##### 主表子节点

主表的子节点就是表的字段

- BpmBind：通过表节点的属性指定的值，能查出该流程发起人相关的BPM信息，BpmBind指定使用该BPM信息的那些数据，目前只提供了：
  - UnitName：所属公司名称
  - UnitCode：所属公司编码
  - JobNo：工号
  - Position：职位
- Date：配置流程发起式的动态时间，目前只支持 Now
- Ref：配置当前字段引用别的字段的值，格式：表名.列名
- Alias：别名，用于给excel中指定中文列名，然后匹配到模板的字段
- 默认值：直接在节点中间写该字段的默认内容



#### 可重复表节点

多行表，通常和主表有关联

- Repeat：True表示是可重复表
- RepeatKey：可重复表的虚拟主键，excel导入的时候判断该列如果没数据，那么直接跳过当前表节点的构建
- ExcelPrefix：excel中的列名前缀，防止多张表存在重复字段名时造成混乱

##### 可重复表子节点

- AutoRepeatCount：如果配置了RepeatKey，然后excel的一行中该RepeatKey列没值，那么就会判断RepeatKey指定的字段是否有默认值，如果有就自动构建一条表节点数据，目前只支持1







