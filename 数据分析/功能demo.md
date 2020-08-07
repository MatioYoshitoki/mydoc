### 根据周进行分表之后将当前周与上一周的表动态得拼接起来

###### 方案一，使用存储过程

```mysql
create PROCEDURE test()
BEGIN
    set @baseTableName = 'chat_';
    set @tableName1 = CONCAT(@baseTableName, date_format(subdate(curdate(),date_format(curdate(),'%w')-1), '%Y%m%d'));
    set @tableName2 = CONCAT(@baseTableName, date_format(subdate(curdate(),date_format(curdate(),'%w')+6), '%Y%m%d'));
    set @sqlStr = CONCAT('select * from ', @tableName1, ' union all select * from ', @tableName2);
    prepare stmt from @sqlStr;
    execute stmt ;
    deallocate prepare stmt;
END;

call test();
```
