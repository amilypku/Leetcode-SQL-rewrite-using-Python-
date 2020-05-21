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

# 262. Trips and Users
sql
```sql
SELECT
    request_at 'Day', round(avg(Status!='completed'), 2) 'Cancellation Rate'
FROM 
    trips t JOIN users u1 ON (t.client_id = u1.users_id AND u1.banned = 'No')
    JOIN users u2 ON (t.driver_id = u2.users_id AND u2.banned = 'No')
WHERE	
    request_at BETWEEN '2013-10-01' AND '2013-10-03'
GROUP BY 
    request_at
```

Python
```python
Trips = Trips[~Trips.Client_Id.isin.User[User['Banned'] == 'Yes']['Users_Id']]
Trips = Trips[~Trips.Driver_Id.isin.User[User['Banned'] == 'Yes']['Users_Id']]
Trips = Trips[(Trips.Request_at >= '2013-10-01') & (Trips.Request_at <= '2013-10-03')]
#case when 
Trips.Status[Trips.Status == 'Completed'] = 0
Trips.Status[Trips.Status != 'Completed'] = 1
dat = Trips.groupby('Request_at').agg({'sum':sum, 'count': count}).to_frame().reset_index
dat['Cancellation Rate'] = round(dat.sum/dat.count,2)
```
# 511. Game Play Analysis I
sql
```
select player_id, event_date as first_login
from (select *,
        row_number() over (partition by player_id order by event_date) as rn 
        from Activity) S 
where rn = 1
```

python
```python
dat = Activity.groupby(['player_id'])['event_date'].rank(method = first)
dat[dat['rn'] == 1][['player_id', 'event_date']]
```

# 512. Game Play Analysis II
sql
```sql
select player_id, device_id
from activity
where (player_id, event_date) in 
(select player_id, min(event_date)
from activity
group by player_id)
```

# 569. Median Employee Salary
sql
```sql
SELECT Id, Company, Salary FROM
(SELECT Id, Company, Salary, COUNT(Salary) OVER (PARTITION BY Company) AS CN,
ROW_NUMBER() OVER (PARTITION BY Company ORDER BY Salary) AS RN FROM Employee) T
WHERE RN = (CN+1)/2 OR RN = (CN+2)/2
```


# 534. Game Play Analysis III
sql
```sql
select A1.player_id, A1.event_date, sum(A2.games_played) as games_played_so_far
from Activity A1 left join Activity A2 on A1.player_id = A2.player_id and A1.event_date >= A2.event_date
group by A1.player_id, A1.event_date
order by A1.player_id, A1.event_date
```
python
Perform an asof merge. This is similar to a left-join except that we match on nearest key rather than equal keys.

```python

```

# 550. Game Play Analysis IV
sql
```sql
--注意这道题求的是首日注册后第二天连续登录的.不是任意两天连续登录就行.
#V1:
select round((select count(distinct(A1.player_id))
from (select player_id, min(event_date) as event_date from Activity group by player_id) A1 join Activity A2 on A1.player_id = A2.player_id and datediff(A2.event_date, A1.event_date) = 1)/count(distinct player_id),2) as fraction
from Activity

#V2:得出所有玩家次日登录时间，看原数据中是否存在，统计存在的个数，除以总人数。
SELECT
	ROUND(COUNT(DISTINCT player_id)/(SELECT COUNT(distinct player_id) FROM Activity), 
	2) AS fraction
FROM
    Activity
WHERE
	(player_id,event_date)
	IN
	(SELECT 
        player_id,
        Date(min(event_date)+1)
	FROM Activity
	GROUP BY player_id);
```
python
```python
A1 = Activity.groupby(['player_id'])['event_date'].min().to_frame('Date').reset_index()
A1['date1'] = A1['Date'].apply(lambda x: datetime.strptime(x, '%Y%m%d'))
A1['Date1']= A1['Date'] +  timedelta(days=1)
index1 = pd.MultiIndex.from_arrays([A1[col] for col in ['player_id','Date1']])
index2 = pd.MultiIndex.from_arrays([Activity[col] for col in ['player_id','event_date']])
New = Activity.loc[index2.isin(index1)]
fraction = round(New.player_id.nunique()/Activity.player_id.nunique(),2)
#

# 595. Big Countries
sql
```sql
select name, population, area
from World
where area > 3000000 or population > 25000000
order by name
```
python
```python
world = World[(world['area'] > 3000000) or (world['population'] > 25000000)]
world.sort_values(by=[‘name’])
```
# 596. Classes More Than 5 Students
sql
```sql
select class 
from courses 
group by class 
having count(distinct(student)) >= 5
```
python
```python
dat = courses.groupby(['class'])['student'].nunique().to_frame('count').reset_index
dat[dat['count'] >= 5]['class']
```
# 601. Human Traffic of Stadium
sql
```sql
select
	id, visit_date, people
from
	(select
		id, visit_date, people,
		count(*) over (partition by offset) cnt
	from
		(select
			id, visit_date, people,
			(row_number() over (order by id) - id) offset
		from stadium
		where people >= 100
		) R--get consecutive id
	) R1
where cnt >= 3   
order by id
```
python
```python 
dat = stadium[stadium['people'] >= 100]
dat['rn'] = dat.rank(method = 'first')
dat['offset'] = dat['rn'] - dat['id']
dat1 = dat.groupby(['offset']).count().to_frame('count').reset_index
data = pd.merge(dat, dat1, how = 'left', left_on = 'offset', right_on = 'offset')
data[data['count'] >= 3].sort_values(by=[‘id’])
```

# 620. Not Boring Movies
sql
```sql
select *
from cinema
where id % 2 <> 0 and description <> 'boring'
order by rating desc
```

python
```python
cinema[(cinema['id'] % 2 != 0) & (cinema['description'] != 'boring')].sort_value(by = ['rating'], ascending = 0)
```

# 626. Exchange Seats
sql
```sql
select 
    if(id%2=0,
        id-1,
        if(id=(select count(distinct id) from seat),
            id,
            id+1)) 
    as id,student 
from seat 
order by id
```
这道题目实际上很简单
查询id和student

若id是偶数，减1
若id是奇数，加1
问题在于当总数为奇数时，最后一个id应保持不变，加1会导致空出一位。那么我们找到最后一位，让它保持不变就可以了。

作者：fan-lu-5
链接：https://leetcode-cn.com/problems/exchange-seats/solution/jian-dan-yi-dong-xiao-lu-ji-bai-suo-you-by-fan-lu-/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

```sql
SELECT
    (CASE
        WHEN MOD(id, 2) != 0 AND counts != id THEN id + 1 --mod is to get remaider
        WHEN MOD(id, 2) != 0 AND counts = id THEN id
        ELSE id - 1
    END) AS id,
    student
FROM
    seat,
    (SELECT
        COUNT(*) AS counts
    FROM
        seat) AS seat_counts
ORDER BY id ASC;
```
```python
```

# 627. Swap Salary
sql
```sql
UPDATE salary
SET sex = if(sex='m','f','m')
```

python
```python 
salary.sex[salary.sex == 'm'] = 'f'
salary.sex[salary.sex == 'f'] = 'm'
```






