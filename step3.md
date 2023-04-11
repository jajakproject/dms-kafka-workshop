# Setup MS-CDC for table without Primary Key
## Table setup
create person_dump
```SQL
CREATE TABLE person_dump (  
    LastName varchar(255),  
    FirstName varchar(255),  
    Address varchar(255),  
    City varchar(255)  
);
```
insert a record into person
```SQL
insert into person_dump values ('A','A','A','HK')
```

## MS-CDC
Run the following update T-SQL in MS SQL.
```SQL
update person_dump set FirstName = 'B' where lastName = 'A'
update person set full_name = 'Brittany A.J.K.W' where ID = 1
```
Then go back to check Table Statistic, there should be changes in person table but not person_dump.
This is because person_dump is a table without PK/UI. We need to enable MS-CDC for it.

Run the following command to enable MS-CDC for that table
```SQL
EXEC sys.sp_cdc_enable_db

EXEC sys.sp_cdc_enable_table @source_schema = N'dbo', @source_name = N'person_dump', @role_name = NULL;
```

Run the update again
```SQL
update person_dump set FirstName = 'B' where lastName = 'A'
```

You should be able to find the changes in Table Statistic
