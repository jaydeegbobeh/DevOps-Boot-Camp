# SQL Server Fundamentals

- Implementation of a relational database, pile of data that a lot of people can access at once 
- SSMS is a GUI to manage your databases, but often use unix shells/powershell
- SQL server profiler: allows you to make a trace of all commands being sent to SQL server
- Database Engine Tuning Advisor: used to optimise the performance of SQL server by analysing commands being sent to SQL server

- example
	- Table names
	-insert into names values ('dan') 
	- select * from names

## T-SQL (Transact-SQL) example
- Join clause is used to combine rows from two or more tables, based on a related column between them
	- select emp.EmployeeID, per.FirstName, per.LastName
	- from HumanResources.Employee as emp: setting aliases
	- join
	- Person.Contact as per
	- on emp.ContactID = per.ContactID

- e.g 2
```
use People;
create table stuff
(
id int identity,
name varchar(50)
);

insert into stuff values ('Bob')
insert into stuff values ('Jim')
insert into stuff values ('Kim')


select * from stuff
```

## OLTP vs OLAP
- Both online processing systems.
- OLTP = transactional processing, manages transaction-oriented applications on the internet e.g ATM
- OLAP = analytical processing system, online system that reports to multidimensional analytical queries e.g financial reporting

## SQL Server Integration Services
- Transform data from OLTP DB to OLAP
- Moving data is easy, controlling it takes a package
- What you got - a source e.g excel spreadsheet -> SSIS -> what you want (OLAP database)

## SQL Server Reporting Services
- Devlopment framework
- Runtime service

## Group By and Having
- Group By: breaks up table
- Having: group predicate

### example for franchise
- Three tables: franchisees- ID, Name, city, state; stores- ID, Franchisee, zipcode, years in business; unified receipts - register ID, store ID, sequence,date, cashier ID

```
SELECT zipcode from stores
GROUP BY zipcode;

SELECT zipcode, COUNT(*) as count from stores
GROUP by zipcode;
--> output number of instances for each zipcode


SELECT zipcode, COUNT(*) as count from stores
AVG([Years in business]) As 'Average' from Stores
GROUP by zipcode;
--> average years in business of stores in each zipcode

SELECT S.Zipcode,
AVG(UR.Amount) AD 'Average', SUM(UR.Amount) AS 'Total'
FROM [Unified Receipts] AS UR JOIN Stores AS S
ON UR.[Store ID] - S.[ID]
Group By S.Zipcode
HAVING avg(UR.Amount) >=60
--> show average receipts greater than 60 for each zipcode


AVG(UR.Amount) AD 'Average', SUM(UR.Amount) AS 'Total'
FROM [Unified Receipts] AS UR JOIN Stores AS S
ON UR.[Store ID] - S.[ID]
WHERE UR.Amount >50
Group By S.Zipcode
HAVING avg(UR.Amount) >=60
--> only include receipts in avg > 50
```

## CTE's (Common Table Expressions) and Ranking
- Extension to syntax insert, select, update, delete statements
- Temporary named result set that you can reference within one of these statements ^
- *Top* clause can be used in a SELECT statement to prune down the size of the result set to the first N rows in the sequence specified by an ORDER BY clause, use with UPDATE/DELETE to limit number of rows that it processes
- *Ranking* rows numerically ranked by a sort criterion you specify, multiple rankings of a row, each with a different criterion are allowed
- *Partition windows* breaks the result set into multiple windows according to criterion you set, each set processed as separate window

- A syntax, use sub queries
	- name
	- subquery-like syntax
	- use in operation like table
```
with MyName as (query)
MyName 2 as
(query2)
select MyName.length, MyName2.STATE, employees.Name
from MyName join MyName2
on MyName.id = MyName2.id
join Employees
on MyName2.id = Employees.id
```

- example1 salary rates
```
select * from shifts
select * from (plant locations)
select * from employees

with locations
as
(select id, state, premium, (business sector) from (plant locations))
select * from locations
```
- Returns table with id, state, premium as columns

```
select * from shifts
select * from (plant locations)
select * from employees

with locations(plant, st, premium, sector)
as
(select id, state, premium, (business sector) from (plant locations))
select * from locations
```
- Overide column names (must change all columns)

- find the net premium by shift, by plant
```
with [shift premiums]
as 
(select id, premium from shifts),
[local premiums](shift,plant, premium)
as 
(select sp.id, p.id,
sp.premium * pl.premium
from [shift premiums as sp cross join [plant locations] as pl)

select pl.name, lp.shift, lp.premium
from [plant locations] as pl join [local premiums] as lp
on pl.id =lp.plant
order by pl.name, lp.shift
```

- example2 calculate each employee pay relative to the average pay of all employees in terms of standard deviations
```
with [locations] (shift_id, plant_id, premium)
as
(select shifts.id, [plant locations].id,
shifts.premium * [plant locations].premium
from shifts cross join [plant locations])

[employee rates] (id, rate) as
(select employees.id, [base rate] * [location shifts].premium
from employees join [location shifts]
on employees.location = [location shifts].plant_id
and employees.shift = [location shifts].plant_id
),
stats (stdev_rate, avg_rate) as
(select, stdev(rate), avg(rate) from [employee rates])
select employees.id, employees.[first name], employees.[last name],
([employee rates].rate - stats.avg_rate)/stats.stdev_rate
from employees join [employee rates] on employees.id = [employee rates].id
cross join stats
order by [last name], [first name]

## Top
- Uses first N as ORDERED
- Traditional
	- literal number select, order by
- Current
	- (expression) select, insert, update, delete
- SELECT TOP clause is used to specify the number of records to return
- SELECT TOP clause is useful on large tables with thousands of records
	- SELECT TOP 10 * from table
```
DELETE TOP (top_value) [ PERCENT ] 
FROM table
[WHERE conditions];
```

## Ranking
- Assigns a number to a row
	- Ranking function indicates numeric rank relative to other rows
	- ordered value used to calculate
	- may be unique, depending on ranking function used


## Row_Number

- Ranks rows ordered by value calculated from row
	- unique rank
	- select name, row_number() over (order by column_name)

## Dense_Rank

- Ranks rows ordered by value calculated from row
	- same value means same rank

## NTile

- Ranks rows by tile membership
	- result set ordered, broken into about equal size sequences, i.e tiles
	- ranked as memeber of 1st, 2nd, 3rd, 4th etc title
```
quartiles
select name, ntile(4) over (order by weight)
from [mechanical parts]
```

## Partitioning

- Ranking function applied by partition
	- logical window defined by predicate applied to resultset
	- default is entire resultset
```
select name,
rank() over (partition by right(name, len(name) - charindex('-', name))
order by weight) from [mechanical parts]
```

## Aggregate Partitions 

- Aggregates can be applied to partitions
	- no column restrictions
	- mixed partitions

```
select right(name, len(name) - charindex('-', name)).
sum(price) over (partition by right(name, len(name) - charindex('-', name)))
aggregate                         window       
```


## Hierachies

- SQL server is relational
	-	 trees are in the mind of the questioner
- Hierarchyid
	- non-recursive queries
	- early 90's

### Hierarchyid's are for Tree Oriented Questions

- Table is set of entities of same type
	- entity ~ row described by column values
- Tree oriented questions

### Why use hierarchyid?

-Tree oriented questions
	- converted to range queries or simple lookups
- Size
	-  parent foreign key typically smaller
- Maintenance
	- integrity
	- moving a node

### Hierarchyid properties

- Tree coordinates
	- like cartesian but for trees
	- supported by functions
  
### Finding Descendants

-CLR based udt
- IsDescendantOf (or self)
- GetLevel

```
select name from personnel
where node.IsDescendantOf('/2/') = 1 and
node.GetLevel() = 2
``` 
### Adding nodes

- GetDescendant
	- just generates a hierarchyid, returns a child node of the parent

```
parent.GetDescendant (child1, child2)
```

### GetReparentedValue

- Calculates new hierarchyid on move
	- walk down the tree


### Depth & Breadth

-IsDescendantOf -> rangequery

```
select node from personnel
where node >= Mary and node < Jack
```

## Managing query plans

- To be able to execute queries, the SQL Server Database Engine must analyze the statement to determine the most efficient way to access the required data. This analysis is handled by a component called the Query Optimizer. The input to the Query Optimizer consists of the query, the database schema (table and index definitions), and the database statistics. The output of the Query Optimizer is a query execution plan, sometimes referred to as a query plan, or execution plan.

### Parameterization
- Literal values make statements look different
- Automatic
	- simple
	- forced
```
select * from sales where agent = @agent
```


### Plan guides

- Control plan generation
	- adds/removes hints
	- specific plan
	- changes parameterization

### Kinds of plan guides

- SQL
- Object
- Template
