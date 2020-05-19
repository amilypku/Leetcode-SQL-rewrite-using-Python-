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

# 176. Second Highest Salary
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

# 177. Nth Highest Salary (create functions)
sql
```sql
# V1: group by to drop duplicate values, or you can use distinct
CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
BEGIN
  set N := N-1;
  RETURN (
      # Write your MySQL query statement below.
      select Salary
      from Employee
      group by Salary
      order by Salary desc
      limit N,1
  );
end

#v2:Find out the n-1 salary by getting n-1 salaries higher than this salary
CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
BEGIN
  set N := N-1;
  RETURN (
      # Write your MySQL query statement below.
      select distinct e1.Salary
      from Employee e1 
      where (select count(distinct Salary) from Employee e2 where e2.Salary > e1.Salary) = N 
  );
end

#v3: count how many salary is higher than this salary
CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
BEGIN
  set N := N-1;
  RETURN (
      # Write your MySQL query statement below.
      select distinct e1.Salary
      from Employee e1 left join Employee e2 on e1.Salary < e2.Salary
      group by e1.Salary 
      having count(distinct e2.Salary) = N 
  );
end

#v4: window function
#row_number(): 同薪不同名，相当于行号，例如3000、2000、2000、1000排名后为1、2、3、4
#rank(): 同薪同名，有跳级，例如3000、2000、2000、1000排名后为1、2、2、4
#dense_rank(): 同薪同名，无跳级，例如3000、2000、2000、1000排名后为1、2、2、3

CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
BEGIN
  RETURN (
      # Write your MySQL query statement below.
      select distinct Salary
      from (select Salary, dense_rank() over (order by Salary Desc) as RN
            from Employee) E
      where RN = N   
  );
end
```

Python
```python
def getNthHighestSalary(N):
    employee['salary_rank'] = employee['Salary'].rank(method = 'dense', ascending = 0)
    Salary = employee['Salary'].drop_duplicates().loc[employee['salary_rank'] == N]
    return Salary
```
# 178. Rank Scores
sql
```sql
select Score, RN as Rank
from (select Score, 
        dense_rank() over (order by Score desc) as RN
        from Scores) s
order by RN

#v2
select a.Score as Score,
(select count(distinct b.Score) from Scores b where b.Score >= a.Score) as Rank
from Scores a
order by a.Score DESC
```
python
```python
Scores['rank'] = Scores['Score'].rank(method = 'dense', ascending = 0)
Scores[['rank','Score']].sort_values(by= ['rank'])
```

# 180. Consecutive Numbers
sql
```sql
select distinct l1.Num as ConsecutiveNums
from Logs l1, Logs l2, Logs l3 
where l1.Id = l2.Id - 1 and l2.Id = l3.Id - 1 and l1.Num = l2.Num and  l2.Num = l3.Num

#window function
select distinct(num) ConsecutiveNums
from (
	select num,(row_number() over(order by id )-row_number() over(partition by num order by id)) as rn 
	from Logs
) tmp
group by rn,num
having count(rn)>=3;
```

python 
```python
#translating window function
Logs['rn'] = Logs['Id'].rank(method = 'first') 
Logs['rn1'] = Logs.groupby(['Num'])['Id'].rank(method = 'first')
Logs['dif'] = Logs['rn']-Logs['rn1']
Logs1 = Logs.groupby(['dif', 'Num'])['dif'].count().to_frame('count').reset_index()
Logs1[Logs1['count'] >= 3]['Num'].drop_duplicates()

#transating the three tables join
#？？？
```

# 181. Employees Earning More Than Their Managers

sql
```sql
select e2.Name as Employee
from Employee e1 join Employee e2 on e1.Id = e2.ManagerId and e1.Salary < e2.Salary
```

python
```python
```

# 182. Duplicate Emails
sql
```sql
select Email
from Person
group by Email
having count(*) > 1
```
python
```python
New = Person.groupby(['Email'])['Id'].count().to_frame('count').reset_index()
New[New['count'] > 1]['Email']
```
# 183. Customers Who Never Order
sql
```sql
select c.Name as Customers 
from Customers c 
left join Orders o on o.CustomerId = c.Id 
where o.Id is null
```

python
```python
new_df = pd.merge(Customers, Orders,  how='left', left_on=['id'], right_on = ['id'], suffixes = ('_t1','_t2'))
new_df[new_df['id_t2'] is null]['Name']
```
# 184. Department Highest Salary
sql
```sql
select D.Name as Department, E.Name as Employee, E.Salary
from (select *,
        dense_rank() over (partition by DepartmentId order by Salary desc) as rn
        from Employee) E join 
        Department D on E.DepartmentId = D.Id
where rn = 1
order by E.Salary

#v2:
select d.Name Department,e.Name Employee,Salary
from  Employee e
join Department d 
on e.DepartmentId=d.Id
where(e.DepartmentId , Salary) IN(
    select DepartmentId, max(salary)
    from Employee
    group by DepartmentId
);
```

python 
```python
Employee = Employee.groupby(['DepartmentId'])['Salary'].rank(method = 'dense', ascending = 0)
dat = pd.merge(Employee, Department, how = 'inner', left_on = 'DepartmentId', right_on = 'Id',suffixes = ('_t1','_t2'))
result = dat[dat['rn'] = 1][['Name_t2','Name_t1','Salary']]
result.columns = ['Department', 'Employee','Salary']
```

# 185. Department Top Three Salaries
sql
```sql
select D.Name as Department, E.Name as Employee, E.Salary
from (select *,
        dense_rank() over (partition by DepartmentId order by Salary desc) as rn
        from Employee) E join 
        Department D on E.DepartmentId = D.Id
where rn <= 3
order by E.Salary
```

python
```python
Employee = Employee.groupby(['DepartmentId'])['Salary'].rank(method = 'dense', ascending = 0)
dat = pd.merge(Employee, Department, how = 'inner', left_on = 'DepartmentId', right_on = 'Id',suffixes = ('_t1','_t2'))
result = dat[dat['rn'] <= 3][['Name_t2','Name_t1','Salary']]
result.columns = ['Department', 'Employee','Salary']
```

# 196. Delete Duplicate Emails
sql
```sql
DELETE p1 FROM Person p1,
    Person p2
WHERE
    p1.Email = p2.Email AND p1.Id > p2.Id
    
#v2: not in subquery
DELETE from Person 
Where Id not in (
    Select Id 
    From(
    Select MIN(Id) as id
    From Person 
    Group by Email
   ) t
)
```

python
```python
P1 = Person.groupby(['Email'])['Id'].min().to_frame('id').reset_index()
Person[~Person.Id.isin(P1['id'])] 
#should be Person.Id since isin need series type
#.isin(P1['id']) should be P1['id'] since inside isin need list type
```

# 197. Rising Temperature
sql
```sql
select w1.Id
from Weather w1, Weather w2
where dateDiff(w1.RecordDate,w2.RecordDate) = 1 
 and w1.Temperature > w2.Temperature
```

python 
```python
#datediff in python
from datetime import datetime

def daysdiff(d1, d2):
    d1 = datetime.strptime(d1, "%Y-%m-%d")
    d2 = datetime.strptime(d2, "%Y-%m-%d")
    return (d1 - d2).days
```

