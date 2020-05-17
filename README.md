# Leetcode-SQL-rewrite-using-Python-
This is a repository created by Lei Huang to record Leetcode SQL practice.

# 175. Combine Two Tables

sql

```sql
select FirstName, LastName, City, State
from Person P left join Address A on P.PersonId = A.PersonId
```
```python
table = Person.merge(Address, left_on = ['PersonId'], right_on = ['PersonId'], how = 'left')
table['FirstName', 'LastName', 'City', 'State']
```





