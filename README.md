# Leetcode-SQL-rewrite-using-Python-
This is a repository created by Lei Huang to record Leetcode SQL practice.

# 175. Combine Two Tables

sql
```sql
select FirstName, LastName, City, State
from Person P left join Address A on P.PersonId = A.PersonId
```
Python
```python
table = Person.merge(Address, left_on = ['PersonId'], right_on = ['PersonId'], how = 'left')
table['FirstName', 'LastName', 'City', 'State']
```

#176. Second Highest Salary
sql
```sql
#if there is no tie and consider null value
V1: select (select distinct salary from Employee order by salary desc limit 1,1) as SecondHighestSalary 

V2:
ifnull(x，y)，若x不为空则返回x，否则返回y，这道题y=null
limit x，y，找到对应的记录就停止
distinct，过滤关键字

select 
ifnull
(
    (select distinct Salary
    from Employee
    order by Salary desc
    limit 1,1),
    null
)as 'SecondHighestSalary'

#if there is a tie
V1: 
select max(salary) SecondHighestSalary
from employee
where salary < (select max(salary) from employee)

V2: 
SELECT max(salary) SecondHighestSalary
 FROM (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) rank_num
         FROM Employee
      )
WHERE rank_num = 2
```

python
```python
employee['salary_rank'] = employee['Salary'].rank(method = 'dense', ascending = 0)
employee['Salary'].drop_duplicates().loc[employee['salary_rank'] == 1]
```

