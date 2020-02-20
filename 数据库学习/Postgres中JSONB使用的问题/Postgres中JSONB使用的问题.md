## JSONB拼接
```  sql
INSERT INTO table_name (x,y) VALUES(jsonb_build_array(数据),y) ON CONFLICT (y) 
UPDATE SET x=table_name.x||数据
```
并发是大量使用||去给原先的JSONB拼接内容会导致硬盘占有量急速提升，我测试了200w次左右拼接，第二天别的同事反映服务器硬盘不够用了，我存的实际大小才几个MB的东西，硬生生的占了150G的硬盘
## JSONB的set操作
```  sql
INSERT INTO table_name (x,y) VALUES(jsonb_build_array(数据),y) ON CONFLICT (y) 
UPDATE SET x=jsonb_set(table_name.x,路径，数据)
```
还是和上面一样的并发操作，硬盘占有量还是居高不下，即使使用vacuum还是不行
## 当前采用的解决方案
不在使用并发的拼接操作，采用在代码中汇聚一批数据后，直接存储成一个单独的json，不采用set和拼接操作，或者直接存成text
``` sql
INSERT INTO table_name (x,y) VALUES(jsonb_build_array(数据),y) ON CONFLICT (y) 
UPDATE SET x=数据::jsonb
```