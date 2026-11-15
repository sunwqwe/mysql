#修改id为1的数据，将name修改为itheima
update employee set name = 'itmeiea' where id = 1;

#修改id为1的数据，将name修改为 小前 , gender 修改为 女
update employee set name = '小前',gender = '女' where id = 1;

#将所有的员工入职修改为 2008-01-01
update employee set entrydate = '2008-01-01';


#删除 gender 为女的员工
delete from employee where gender = '女';

#删除所有员工
delete from employee;