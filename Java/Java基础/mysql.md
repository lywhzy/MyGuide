## mysql数据库

### 联表查询

1. 内连接

select * from info_data,info_type where info_data.tid = info_type.id;

select * from info_data as d inner join info_type as t on d.tid = t.id;

2. 左外连接

select * from info_data as d left outer join info_type as t on d.tid = t.id;

3. 右外连接

select * from info_data as d right outer join info_type as t on d.tid = t.id;
