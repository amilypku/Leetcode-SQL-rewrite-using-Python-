# Leetcode-SQL-rewrite-using-Python-
This is a repository created by Lei Huang to record Leetcode SQL practice.

# 175. Combine Two Tables

## sql
...
select FirstName, LastName, City, State
from Person P left join Address A on P.PersonId = A.PersonId
...

## Python
...
Example (Optional)
// code away!

let generateProject = project => {
  let code = [];
  for (let js = 0; js < project.length; js++) {
    code.push(js);
  }
};



