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
Scores['rank'] = Scores['Score'].rank(




= 'dense', ascending = 0)
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

#lead and lag solutions
with x as
(select id,
lead(Num,1,0) over (order by id asc) Lead,
lag(Num,1,0) over (order by id asc) Lag
from Logs
)

select
distinct Num as "ConsecutiveNums"
from
Logs L
join x y on L.id = y.id and y.Lead = L.Num and y.Lag = L.Num;
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

# 569. Median Employee Salary
sql
```sql
SELECT Id, Company, Salary FROM
(SELECT Id, Company, Salary, COUNT(Salary) OVER (PARTITION BY Company) AS CN,
ROW_NUMBER() OVER (PARTITION BY Company ORDER BY Salary) AS RN FROM Employee) T
WHERE RN = (CN+1)/2 OR RN = (CN+2)/2
```
# 570. Managers with at Least 5 Direct Reports
sql
```sql
select Name 
from Employee
where Id in (select ManagerId
            from Employee
            group by ManagerId
            having count(distinct Id) >= 5
	    
#v2:self join 
select e2.Name
from employee e1, employee e2
where e1.ManagerId = e2.Id
group by e2.Id
having count(*)  >= 5 	    
```
python
```python
dat = Employee.groupby(['ManagerId'])['Id'].nunique().to_frame('count')
dat1 = Employee[Employee.Id.isin(dat[dat['count'] >= 5].index)].Name
```
# 571. Find Median Given Frequency of Numbers
sql
```sql
#建两个累计频率，用员工薪水中位数的思路来做
select
avg(t.number) as median
from
(
select
n1.number,
n1.frequency,
(select sum(frequency) from numbers n2 where n2.number<=n1.number) as asc_frequency,
(select sum(frequency) from numbers n3 where n3.number>=n1.number) as desc_frequency
from numbers n1
) t
where t.asc_frequency>= (select sum(frequency) from numbers)/2
and t.desc_frequency>= (select sum(frequency) from numbers)/2
```

# 574. Winning Candidate
sql
```sql
select Name 
from Candidate
where id in (
    select CandidateId
    from Vote
    group by CandidateId
    having count(distinct id) >= all(select count(distinct id)
                                    from Vote
                                    group by CandidateId))
```				    
python
```python
dat = Vote.groupby(['CandidateId'])['id'].nunique().to_frame('count').reset_index
max = dat['count'].max()
Candidate[Candidate.id.isin[dat[dat['count'] >= max].CandidateId]].Name
```
# 578. Get Highest Answer Rate Question
sql
```sql
select question_id as survey_log
from 
(select question_id, sum(case when action = 'answer' then 1 else 0 end)/sum(case when action = 'show' then 1 else 0 end) as rate
        from survey_log
        group by question_id) s
order by rate desc
limit 1 
```

python 
```python
dat = survey_log.groupby(['question_id'])['action'].value_counts().to_frame('count').reset_index()
...???
```

# 579. Find Cumulative Salary of an Employee
sql
```sql
#v1: window function
select Id, Month, total_salary as salary
from (select *, 
            sum(Salary) over (partition by Id order by Month rows 2 preceding) as total_salary,
            row_number() over (partition by Id order by Month desc) as RN
        from Employee) E
where RN > 1
order by Id, Month desc

#v2: self join and exclude max value
select e1.Id, e1.Month, sum(e2.Salary) as Salary
from Employee e1 join Employee e2
on e1.Id = e2.Id
    and e1.Month >= e2.Month
    and e1.Month < e2.Month + 3
where (e1.Id,e1.Month) not in (select Id, max(Month) as Month from Employee group by Id) 
group by e1.Id, e1.Month
order by e1.Id, e1.Month desc
```

python
```python
???
```
# 580. Count Student Number in Departments
sql
```sql
select d.dept_name, ifnull(count(distinct s.student_id),0) as student_number
from student s right join department d on d.dept_id = s.dept_id
group by d.dept_id
order by student_number desc
```

python
```python
dat = student.merge(department,how = 'right', left_on = 'dept_id', right_on = 'dept_id' )
dat1 = dat.groupby(['dept_name'])['student_id'].nunique().to_frame('student_number').reset_index()
dat1 = dat1.sort_values(by=[‘student_number’],asending = 0)
```

# 585. Investments in 2016
sql
```sql
select sum(TIV_2016) as TIV_2016
from insurance
where concat(lat,lon) in (select concat(lat,lon)
                            from insurance 
                            group by concat(lat,lon)
                            having count(concat(lat,lon)) <= 1)
                    and 
         tiv_2015 in (select tiv_2015 
                           from insurance
                           group by tiv_2015
                           having count(tiv_2015) > 1) 

```

python 
```python
insurance['concat'] = insurance['lat'].astype(str) + insurance['lon'].astype(str)
#v2: insurance['concat'] = insurance.lat.str.cat(df.lon.str)
dat1 = insurance.groupby(['concat']).count().to_frame('count').reset_index
dat1 = dat1[dat1['count'] <= 1]
dat2 = insurance.groupby(['tiv_2015']).count().to_frame('count').reset_index
dat2 = dat1[dat1['tiv_2015'] > 1]
dat3 = insurance[(insurance.concat.isin(dat1['concat'])) & (insurance.tiv_2015.isin(dat2['tiv_2015']))]
dat3.TIV_2016.sum()
```
# 597. Friend Requests I: Overall Acceptance Rate
sql
```sql
select round(ifnull(count(distinct b.requester_id, b.accepter_id)/count(distinct a.sender_id, a.send_to_id),0), 2) accept_rate 
from friend_request a, request_accepted b

select round(
    ifnull(
    (select count(distinct requester_id ,accepter_id) from request_accepted) / 
    (select count(distinct sender_id ,send_to_id) from friend_request)
    ,0)
    ,2) as accept_rate ;
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

# 602. Friend Requests II: Who Has the Most Friends
sql
```sql
select accepter_id as id,sum(num) as num
from
    ((select accepter_id, count(*) as num
    from request_accepted
    group by accepter_id) 
    union all 
    (select requester_id, count(*) as num
    from request_accepted
    group by requester_id))  a
group by accepter_id
order by sum(num) desc
limit 1 
```

python
```python
dat1 = request_accepted.groupby(['accepter_id'])['accepter_id'].count().to_frame('num').reset_index()
dat2 = request_accepted.groupby(['requester_id'])['requester_id'].count().to_frame('num').reset_index()
dat = pd.concat([dat1,dat2.rename(columns={'requester_id':'accepter_id'})])
dat = dat.groupby(['accepter_id'])['num'].sum().to_frame('num').reset_index()
dat = dat.sort_values(by = ['num'],ascending = 0)
head(dat,1)
```

# 608. Tree Node
sql
```sql
select id, (case when p_id is null then 'Root'
                when id in (select p_id from tree) then 'Inner'
                else 'Leaf' end) as Type
from tree
```
python 
```python
tree.loc[tree['p_id'].isnull(),'type'] = 'Root'
tree.loc[(tree.id.isin(tree['p_id'])) & (tree['p_id'].notnull()),'type'] = 'Inner'
tree.loc[tree['type'].isnull(),'type'] = 'Leaf'
```

# 612. Shortest Distance in a Plane
sql
```sql
select round(min(sqrt(POWER((p1.x-p2.x),2) + POWER((p1.y-p2.y),2))),2) as shortest
from point_2d p1 join point_2d p2 on (p1.x,p1.y) <> (p2.x, p2.y)
```

# 615. Average Salary: Departments VS Company
sql
```sql
select c.pay_month,c.department_id,
case 
    when c.salary_d>d.salary_c then 'higher'
    when c.salary_d<d.salary_c then 'lower'
    else 'same'
    end 'comparison'
from 
(
 select date_format(a.pay_date,'%Y-%m') pay_month,b.department_id,
avg(a.amount) salary_d from salary a 
join employee b 
on a.employee_id=b.employee_id
group by date_format(a.pay_date,'%Y-%m'),b.department_id   
) c
inner join 
(
select date_format(a.pay_date,'%Y-%m') pay_month,b.department_id,
avg(a.amount) salary_c from salary a 
 join employee b 
on a.employee_id=b.employee_id
group by date_format(a.pay_date,'%Y-%m')
) d 
on c.pay_month=d.pay_month;
```
python
```python

```
# 618. Students Report By Geography
sql
```sql
select max(case when continent = 'America' then name else null end) as America,
        max(case when continent = 'Asia' then name else null end) as Asia,
         max(case when continent = 'Europe' then name else null end) as Europe
from
    (select 
        name, 
        continent, 
        row_number()over(partition by continent order by name) cur_rank
    from
        student)t 
group by cur_rank
```
python need to correct
```python
student.groupby(['continent'])['name'].str().rank(method = 'frist') # error: 'NoneType' object is not callable
Output = student.pivot(index= 'rank', columns= 'continent', values= 'name')
```
# 619. Biggest Single Number
sql
```sql
select max(num) as num
from
(select num
from my_numbers
group by num 
having count(num) = 1) a
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

# 1045. Customers Who Bought All Products
sql
```sql
select t1.customer_id
from (select customer_id, count(distinct product_key) cnt from customer group by customer_id) t1, (select count(*) cnt from product) t2
where t1.cnt = t2.cnt

select c.customer_id
from Product p left join Customer c on p.product_key = c.product_key
where p.product_key = 5
      and c.customer_id in (select c.customer_id
from Product p left join Customer c on p.product_key = c.product_key where p.product_key = 6)
order by customer_id
```
# 1098. Unpopular Books
sql
```sql
select b.book_id, b.name
from Books b left join Orders o on b.book_id = o.book_id
where b.book_id in (select book_id 
                        from Books 
                         where datediff(month, available_from, '2019-06-23') > 1)
group by b.book_id, b.name
having isnull(sum(case when o.dispatch_date < '2018-06-23' then 0 else quantity end),0) < 10  
order by b.book_id
```

# 1107. New Users Daily Count
sql
```sql
#mysql
select login_date,count(user_id) user_count
from (select user_id, min(activity_date) login_date from Traffic
where activity='login'
group by user_id) t
where datediff('2019-06-30',login_date)<=90
group by login_date;


#sql server
SELECT
    login_date,
    COUNT(DISTINCT user_id) user_count
FROM
    (SELECT
        user_id, 
        MIN(activity_date) OVER(PARTITION BY user_id) login_date
    FROM Traffic
    WHERE activity = 'login') t
WHERE datediff(day,login_date,'2019-06-30') <= 90 
GROUP BY login_date
ORDER BY login_date

select activity_date as login_date, count(distinct user_id) as user_count 
from (select *,
        row_number() over (partition by user_id order by activity_date) as rn
        from (select * from Traffic where activity = 'login')a) A 
where rn = 1 and datediff(day,activity_date,'2019-06-30') <= 90 
group by activity_date
```
we can't use row_number to select first user activity here since there are some users' first activity are not log in.
wrong answer:
```sql
select activity_date as login_date, count(distinct user_id) as user_count 
from (select *,
        row_number() over (partition by user_id order by activity_date) as rn
        from Traffic) A 
where rn = 1 and datediff(day,activity_date,'2019-06-30') <= 90 
group by activity_date
```

# 1112. Highest Grade For Each Student
sql
```sql
#window function
select student_id, course_id, grade
from (select *,
        rank() over (partition by student_id order by grade desc, course_id) as rn
    from Enrollments) S
where rn = 1
order by student_id

#sql
SELECT student_id, MIN(course_id) AS course_id, grade
FROM Enrollments
WHERE (student_id, grade) IN (SELECT student_id, MAX(grade)
                              FROM Enrollments
                              GROUP BY student_id)
GROUP BY student_id
ORDER BY student_id
```

python 
```python
Enrollments['rn'] = Enrollments.groupby(['student_id'])['grade'].rank(method = 'dense', ascending = 0)
indice = Enrollments[Enrollments['rn'] == 1].groupby(['student_id'])['course_id'].idxmin # cannot work
#after grouping to minimum value in pandas, how to display the matching row result entirely along min() value
https://datascience.stackexchange.com/questions/26308/after-grouping-to-minimum-value-in-pandas-how-to-display-the-matching-row-resul
need to figure out why
```

# 1126. Active Businesses
sql
```sql
#v1
select business_id
from(select e1.business_id, e1.event_type, (case when e1.occurences > e2.avg then 1 else 0 end) as count
    from Events e1 left join (select event_type, avg(occurences) as avg from Events group by event_type) e2 
            on e1.event_type = e2.event_type) e
group by business_id
having sum(count) > 1

#v2
select business_id
from events e,
(select event_type,avg(occurences) avg_occ
from events
group by event_type) temp
where e.event_type = temp.event_type and e.occurences > temp.avg_occ
group by e.business_id
having count(*) > 1

#v3
select business_id
from Events e left join (
    select event_type, avg(occurences) as tavg
    from Events
    group by event_type
)t on e.event_type = t.event_type
group by business_id
having sum(case when e.occurences > t.tavg then 1 else 0 end) >1
```

python
```python
```

# 1127. User Purchase Platform
sql
```sql
select t2.spend_date, t2.platform, 
ifnull(sum(amount),0) total_amount, ifnull(count(user_id),0) total_users
from
(select distinct spend_date, "desktop" as platform from Spending
union
select distinct spend_date, "mobile" as platform from Spending
union
select distinct spend_date, "both" as platform from Spending 
) t2
left join
(select spend_date, sum(amount) amount, user_id, 
case when count(*) = 1 then platform else "both" end as platform
from Spending 
group by spend_date, user_id) t1
on t1.spend_date = t2.spend_date
and t1.platform = t2. platform
group by t2.spend_date, t2.platform
```

python
```python
spending[‘mobile_spend’] = spending[spending.channel == ‘mobile’].spend
spending[‘desktop_spend’] = spending[spending.channel == ‘desktop’].spend
member_spend = spending.group_by([‘date’, ‘member_id’]).sum([‘mobile_spend’, ‘desktop_spend’]).to_frame([‘mobile_spend’, ‘desktop_spend’]).reset_index()

member_spend.loc[(member_spend.mobile_spend>0) & (member_spend.desktop_spend==0), ‘channel’] = ‘mobile’
member_spend.loc[member_spend.mobile_spend==0 & member_spend.desktop_spend>0, ‘channel’] = ‘desktop’
member_spend.loc[member_spend.mobile_spend>0 & member_spend.desktop_spend>0, ‘channel’] = ‘both’

tot_members = member_spend.groupby([‘date’, ‘channel’]).size().to_frame(‘tot_members’).reset_index()
tot_spend = member_spend.groupby([‘date’, ‘channel’].agg({‘mobile_spend’:sum, ‘desktop_spend’:sum}).to_frame([‘mobile_spend’, ‘desktop_spend’])
tot_spend[‘tot_spend’] = tot_spend[‘mobile_spend’] + tot_spend[‘desktop_spend’]
output = tot_members.concat(tot_spend[‘tot_spend’])
```
# 1132. Reported Posts II
sql
```sql
# count will not count null values
select round(100*avg(percent),2) as average_daily_percent
from (select a.action_date, count(distinct(r.post_id))/count(distinct(a.post_id))  as percent
        from Removals r right join Actions a on r.post_id = a.post_id
        where extra = 'spam' 
        group by a.action_date) P
	
#the version blow is wrong since there are duplicate post_id in one day. If I count with case when, I cannot count distinct post_id.
select round(100*avg(percent),2) as average_daily_percent
from (select a.action_date, sum(case when r.post_id is null then 0 else 1 end)/count(distinct a.post_id) as percent
        from Removals r right join Actions a on r.post_id = a.post_id
        where extra = 'spam' 
        group by a.action_date) P
```

python
```python
#count distinct with group by: two solutions
df.groupby("date").agg({"duration": np.sum, "user_id": pd.Series.nunique})
df.groupby("date").agg({"duration": np.sum, "user_id": lambda x: x.nunique()})
```

# 1141. User Activity for the Past 30 Days I
sql
```sql
# 一定注意日期差是29天 <30.........包含2019-07-27本天
select activity_date day, count(distinct user_id) active_users
from activity
where datediff('2019-07-27', activity_date) < 30
group by activity_date

select activity_date day, count(distinct user_id) active_users
from activity
where activity_date > date_sub('2019-07-27', interval 30 day)
and activity_date <= '2019-07-27'
group by activity_date
```

python
```python
def daysdiff(d1, d2):
    d1 = datetime.strptime(d1, "%Y-%m-%d")
    d2 = datetime.strptime(d2, "%Y-%m-%d")
    return (d1 - d2).days

import datetime as dt
activity['activity_date']] = pd.to_datetime(activity['activity_date'])
activity['day'] = (dt.strptime('2019-07-27', "%Y-%m-%d") - activity['activity_date']).dt.days
dat = activity[activity['day'] < 30]
dat.groupby(['activity_date'])['user_id'].nunique().to_frame('active_users').reset_index()
```



# 1142. User Activity for the Past 30 Days II
sql
```sql
# 为空的时候一定要显示一个0，注意一下。
select ifnull(round(count(distinct session_id)/count(distinct user_id),2), 0)  average_sessions_per_user
from activity
where datediff('2019-07-27', activity_date) < 30

#v2
select ifnull(round(sum(t.amount)/count(user_id),2), 0)  average_sessions_per_user
from
(select user_id, count(distinct session_id) amount
from activity
where datediff('2019-07-27', activity_date) < 30
group by user_id) t
```

# 1149. Article Views II
oracle sql
```sql
#v1: out of time
select distinct viewer_id as id
from Views 
group by viewer_id, view_date 
having count(distinct article_id) > 1
order by viewer_id
```

python
```python 
Views




