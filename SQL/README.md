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