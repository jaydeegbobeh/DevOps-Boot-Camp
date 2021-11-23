# KQL (Kusto Query Language) from Scratch

- Uses the Kusto Engine to query big data sets, works with azure
- Where to use KQL
	- Azure log analytics (part of Azure's monitoring solution), monitors cloud environments
		- [Azure log analytics demo](https://aka.ms/LADemo)
	- Azure application insights
	- Windows defender advanced threat protection
	- Azure security center


# Operators
## The search command
```
| search "Memory" 
// Make search case sensitive

search kind=case_sensitive "memory"

// Search in specific tables (Perf,Event, Alert tables)
search in(Perf, Event, Alert) "Contoso"

// Search for a specific column for an exact value

search CounterName=="Available Mbytes"

// Search for value anywhere in the text in the specified column
search CounterName:"MBytes"


// Search for a column whose value exactly matched the word Memory
search "Memory"

// Search across all columns using wildcards
search "*Bytes*"

// Find results that start with search term
search * startswith "Bytes"

// Ends with search 
search * endswith "Bytes"


// Begins with Free, ends with bytes, anything inbetween
search "Free*bytes"

## Logic combine searches
search "Free*bytes and ("c:" or "d:")

```

## The where command

```
// Where limits the result set rather than looking across columns for values, where limits based on conditions
Perf
| where TimeGenerated >= ago(1h)

// ago allows for relative date ranges, Ago says start right now then go back in time N quantity
abreviations:
// d - days
// h - hours
// m - minutes
// s - seconds
// ms - milliseconds
// microseconds - microseconds

where TimeGerated >= ago(1h)
	and (CounterName == "Bytes Received/sec"
		or
		CounterName == "% Processor Time"
	and CounterValue > 0

// Search positional matches

where * hasprefix "Bytes"

where * hassuffix "Bytes"

where  contains "Bytes"

```

## The take command

- Take is used to grab a random number of rows from the input data
```
Perf 
| take 10
// query run a second time may result in different set of rows, or same - depends on varius factors

Perf
| where TimeGenerated >= ago(1h)
	and CounterName == Bytes Received/sec"
	and CounterValue > 0
| take 5

// Limit is a synonym for take
Perf
| limit 10

```

## The count command
- Returns the number of rows in the input dataset
```
Perf
| count
// Use with other filters
Perf
| where TimeGenerated >= ago(1h)
	and CounterName == "Bytes Received/sec"
	and CounterValue > 0
| count

```

## The summarize command
```
// Summarize allows you to count number of rows by column using the count()
// aggregation function
Perf
| summarize count() by CounterName

// Break down by multiple columns
Perf
| summarize count() by ObjectName, CounterName
s
// You can rename the output column for count
Perf
| summarize PerfCount=count()
	by ObjectName, CounterName

// With summarize you can use other aggregation functions
Perf 
| where CounterName == "% Free Space"
| summarize NumberOfEntries=count()
	, AverageFreeSpace=avg(counterValue)
	by CounterName

// Bin allows you to summarize into logical groups, like days
Perf
| summarize NumberOfEntries=count()
	by bin(TimeGenerated, 1d)
```

## The extend command

```
// Extend creates a calculated column and adds to the result set
Perf
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000

// Can extend multiple columns at the same time
Perf
| where CounterValue == "Free Megabytes"
| extend FreeGB = CounterValue / 1000
, FreeKB = CounterValue * 1000


// Can also use with strcat to creat new string columns
Perf
| where TimeGenerated >=(10m)
| extend ObjectCounter = strcat(ObjectName, " - ", CounterName)

```

## The project command

```
// Project allows you to select a subset of columns
Perf
| project ObjectName
		, CounterName
		, InstanceName
		, CounterValue
		, TimeGenerated


| where CounterName == "Free Megabytes"
| project ObjectName
		, CounterName
		, InstanceName
		, CounterValue
		, TimeGenerated
| extend FreeGB = CounterValue / 1000
		, FreeMB = CounterValue 
		, FreeKB = CounterValue * 1000

// There is a variant called project-away. It will project all except the columns listed
Perf
| where TimeGenerated > ago(1h)
| project-away TenantId
		, SourceSystem
		, CounterPath
		, MG

```

// The distinct command
```
// Returns a list of deduplicated values for columns from the input dataset
Perf
| distinct ObjectName, CounterName

// Distinct can be used to limit a result set
// Get a list of all sources that had an error event
Event
| where EventLevelName  == "Error"
| distinct Source
```


## The top command
```
// Top returns the first N rows of the dataset when the dataset is sorted by the "by" clause
Perf
| top 20 by TimeGenerated desc 

// When we combine what we've learned to produce something useful
// Get list of computers that are low on disk space
Perf
| where CounterName == "Free Megabytes" //Get the free mb
	and TimeGenerated >= ago(1h) // within the last hour
| project Computer // for each return the computer name
		, TimeGenerated // when the counter was generated
		, CounterName // and the counter name
		, FreeMegabytes=CounterValue // rename counter value
| distinct Computer // weed out duplicate rows
		, TimeGenerated
		, CounterName
		, FreeMegabytes
| top 25 by FreeMegabytes asc  
```

# Scalar Operators
## The print and now Commands
```
// print can be used to display output to the result grid, IT is primarily a debugging tool.
print "Hello World"

// calculations
print 11 * 3

// You can also name the output column
print TheAnswerToLifeTheUniverseAndEverything = 21 * 2

// current time (GMT)
print now()

// time x amount of time ago
print ago(1d)
print ago(1h)
print ago(1m)
print ago(1s)
print ago(1ms)
print ago (-1d) // tomorrow
```

## The sort command
```
// sort will sort the output of a query
Perf
| where TimeGenerated > ago(15m)
| where CounterName == "Avg. Disk sec/Read"
	and InstanceName == "C:"
| project Computer
		, TimeGenerated
		, ObjectName
		, CounterName
		, InstanceName
		, CounterValue
| sort by Computer
		, TimeGenerated

```

## The extract command

```
// Extract pulls part of a passed in string (the third parameter)
// based on the regular expression placed inside parenthesis
// The second param determines what is returned. A 0 returns
// the whole expression
Perf
| where ObjectName == "LogicalDisk"
	and InstanceName matches regex "[A-Z]:"
| project Computer
		, CounterName
		, extract("[A-Z]:", 0, InstanceName)
```

## The parse command
```
// Evaluates a string expression and parses its value into one or more calculated columns. The calculated columns will have nulls, for unsuccessfully parsed strings. If there is no need to use rows where parsing doesn't succeed, prefer using the parse-where operator.
Event
| where RenderedDescription startswith "Event code"
| parse RenderedDescription with "Event code: " myEventCode
			"Event message: " myEventMessage
			"Event time: " myEventTime
| project myEventCode, myEventMessage, myEventTime

```
## datetime and timespan Arithmetic
```
Determine how long ago a counter was generated
Perf
| where CounterName == "Avg. Disk sec/Read"
| where CounterValue > 0
| take 100
| extend HowLongAgo=( now() - TimeGenerated)
| project Computer
	, CounterName
	, CounterValue
	, TimeGenerated
	, HowLongAgo
```

## The startof command

```
// Useful to know the start of sch as the start of the day, week, month, year
Event
| where TimeGenerated >= ago(7d)
| extend DayGenerated = startofday(TimeGenerated)
| project Source
	, TimeGenerated
	, DayGenerated
```

## The endof command
- Similar to start of, function for end of a time period

## The between command

```
//Between is used to get a range of values.
Perf
| where CounterNmae == "%Free Space"
| where CounterValue between (70.0 .. 100.0)

// Also used with dates
Perf
| where CounterNmae == "%Free Space"
| where CounterValue between (datetime(2018-04-01) .. datetime(2018-04-03))
// omits data before/after midnight of these days => use startof/endof for inclusive results

Perf
| where CounterNmae == "%Free Space"
| where CounterValue between (startofday(datetime(2018-04-01)) .. endofday(datetime(2018-04-03)))

```
## The todynamic command

## The format_Datetime and format_timespan commands

```
Perf
| take 100
| project CounterName
	, CounterValue
	, TimeGenerated
	, format_datetime(TimeGenerated, "y-M-d")
	, format_datetime(TimeGenerated, "yyyy-MM-dd")
	, format_datetime(TimeGenerated, "MM/dd/yyyy hh:mm:ss tt"
// d - day 1-31
// dd - day 01-31
// M - 1-12
// MM - 01-12
// y- 0-9999
// yy - 00-9999
// yyyy - 0000-9999
```

## The datetime_part command

- Extracts the requested date part as an integer value.

```
let dt = datetime(2017-10-30 01:02:03.7654321); 
print 
year = datetime_part("year", dt),
quarter = datetime_part("quarter", dt),
month = datetime_part("month", dt),
weekOfYear = datetime_part("week_of_year", dt),
day = datetime_part("day", dt),
dayOfYear = datetime_part("dayOfYear", dt),
hour = datetime_part("hour", dt),
minute = datetime_part("minute", dt),
second = datetime_part("second", dt),
millisecond = datetime_part("millisecond", dt),
microsecond = datetime_part("microsecond", dt),
nanosecond = datetime_part("nanosecond", dt)

e.g returns 303 under day of year column
```

## iif
- if, then else logic
- Evaluates the first argument (the predicate), and returns the value of either the second or third arguments, depending on whether the predicate evaluated to true (second) or false (third).

```
// iif is a min if/then/else
Perf
| where CounterName == "% Free Space"
| extend FreeState = iif CounterValue < 50
		, You might want to look at this
		, "You're ok!"
```

## The case command
- multilevel if, then else clause
- Evaluates a list of predicates and returns the first result expression whose predicate is satisfied.

```
Perf
| where CounterName == "% Free Space"
| extend FreeLevel = case (CounterValue < 10 "Critical"
			, CounterValue < 30, "Danger"
			, CounterValue < 50, "Look at it"
			, "You're OK!"
			)
| project Computer
	, CounterName
	, CounterValue
	, FreeLevel
```

## isempty and isnull commands
- In KQL strings can be empty and numeric fields can be null, to handle determining when a string is empty or a number is null we have isempty and isnull
```
// isempty matches on empty text strings, more readable rather than an empty Counter
Perf
| where isempty( InstanceName )
| count
Perf
| where TimeGenerated >= ago(1h)
| extend InstName = iif (isempty(InstanceName)
		, "NO INSTANCE NAME"
		, InstanceName
		)

// isnull matches on null columns
Perf
| where isnull( SampleCount )
| count
Perf
| where TimeGenerated >= ago (1h)
|exten SampleCountNull = iff( isnull(SampleCount)
			, "No Sample Count"
			, tostring(SampleCount)
			)
```

## The split command
- Splits a given string according to a given delimiter and returns a string array with the contained substrings.

```
// Use split to break a string into an array based upon a delimited
// Note \ is an escape character so you have to use two of them to denote a single one in the string
Perf
| take 100 // done just to give us a small dataset to demo
| project Computer
	= CounterName
	= CounterValue
	= CounterPath
	=CPSplit = split(CounterPath, "\\")
```

## String operators

```
Perf
| take 100 // done just to give us a small dataset to demo
| where CounterName countains "BYTES"

// With case sensitive
Perf
| take 100 // done just to give us a small dataset to demo
| where CounterName contains_cs "BYTES"

// These also have a "NOT" version
Perf
| take 100
| where CounterName !contains "Bytes"

// in - used to compare a column to a set of values
Perf
| take 1000 // done just to give us a small dataser to demo
| where ConterName in ("Disk Transfers/sec", "Disk Reads/secs", "Avg. Disk sec/Write")

// Also has a NOT version
Perf 
| take 100 // done just to give us a small dataset to demo
| where CounterName !in ( " Disk Transfers/sec"
			, "Disk Reads/sec"
			, "Avg. Disk sec/Write"
			)
```

## The strcat command

- Concatenates between 1 and 64 arguments.

	- If the arguments aren't of string type, they'll be forcibly converted to string.

```
Perf
| take 100 // done just to give us a small dataset to demo
| extend CompObjCounter = strcat((Computer, " - ", ObjectNamw, " - ", CounterName)
| project CompObJCounter
	, TimeGenerated
	, CounterValue

// Use strcat with case and datatime_part to get month names
Perf
| where CounterName == "% Free Space"
| where TimeGenerated between 
```

# Advanced Aggregations
## arg_max and arg_min commands 
- **arg_max** finds the maximum value for the column bein summarized on. and returns the row where that maximum value was found.

```
Perf
| summarize arg_max(CounterValue, *) by CounterName
| sort by CounterName asc
```
- Return all the rows with the highest value in all the columns (*)

- **arg_min** does the same except, finding the min value for the column being summarized on, and returning rows where that min value was found

```
Perf
| project CounterName, CounterValue
| summarize arg_min(CounterValue, *) by CounterName
| sort by CounterName asc
```

## makeset and makelist commands
 - makeset, creates an array of json objects by flattening a hierachy
- gives list computers with less than or equal to 30% free space, makeset removes duplicates
```
Perf
| where CounterName == "% Free Space"
	and CounterValue <= 30
| summarize Computes = makeset(Computer)
```
- make list is similar to makeset but duplicates are not removed
```
Perf
| where CounterName == "% Free Space"
	and CounterValue <= 30
| summarize Computes = makelist(Computer, 256)

// use ,256 to set max size of returned list
```

## mvexpand
- mvexpand takes dynamic value (like a set or list) and converts it back into individual rows

```
SecurityAlert
| exten ExtProps=todynamic(ExtendedProperties)
| mv expand ExtProps
| project TimeGenerated
		, DisplayName
		, AlertName
		, AlertSeverity
		, ExtProps
```
## percentiles
- Calculates the value that is greater than x% of the sampleset.
- Here it shows the CounterValue that is higher than 5% of the other values in the group, then the value that is greater than 50% of the incoming data and finally a value that is greater than 95% of the data

Perf
| where CounterName == "Available MBytes"
| summarize percentiles(CounterValue,5, 50, 95) by computer


## dcount command
- Rerturns an estimate for the number of distinct values that are taken by a scalar expression in the summary group
- Use dcount to get an estimate of these
```
SecurityEvent
| where TimeGenerated >= ago(90d)
| summarize dcount(EventID) by Computer

| sort by Computer asc


## dcountif
- Returns an estimate of the number of distinct values of Expr of rows for which Predicate evaluates to true
- Look for results with specific EventID list
```
Security 
| where TimeGenerated >= ago(90d)
| summarize dcountif ( EventID
					, EventID in (4625, 4688, 4624)
					)

```
## countif
- similar to dcountif, countif allows you to add a filter condition.

```
Perf
| summarize RowCount = countif(CounterName contains "Bytes") by CounterName
| sort by CounterName asc
```

## pivot
- rotates a table by turning the unique values from one column in the input table into multiple columns in the output table, and performs aggregations where they are required on any remaining column values that are wanted in the final output.
```
Event
| project Computer, EventLevelName
| evaluate pivot(EventLevelName)
| sort by Computer asc
```
- Return computer with error and warning count

# top-nested command
- accepts tabular data as input, and one or more aggregation clauses. The first aggregation clause (left-most) subdivides the input records into partitions, according to the unique values of some expression over those records.

```
// e.g get first top 3 ObjectNames as measured by their count
// for each on of those, it gets the top 3 CounterNames, as measured by their count
Perf
| top-nested 3 of ObjectName by ObjectCount = count()
, top-nested 3 of OjectName by CounterNameCount = count
| sort by OjectName asc
		, CounterName asc
```

## max and min
- max returns the maximum value for the column being passed in
```
Perf
| where CounterName == "Free Megabytes"
| summarize max(CounterValue)
```
- min returns minimum value for the column being passed in

```
Perf
| where CounterName == "Free Megabytes"
| summarize min(CounterValue)
```
## sum and sumif
- sum returns the grand total for the indicated column
```
Perf
| where CounterName == "Free Megabytes"
| summarize sum(CounterValue)
```
- sumif allows you to add your condition to the paramerer list
```
Perf
| summarize sumif(CounterValue, CounterName == "Free Megabytes")
```

## any

- any is a randow row generator, when used with an *, it returns a random row from the input dataset
```
Perf
| summarize any(*)
```


- specify column name to return 

```
Perf
| summarize any(Computer)
```

- specify combo of column names to return

```
Perf
| summarize any(ObjectName, CounterName, CounterValue)
```

- use it with by, it returns a random row for each distinct value in the column after the by clause

```
Perf
| summarize any(*) by CounterName
| sort by CounterName asc
```

# Working with Datasets

## let
- let has many uses, first it can be used to represent a constant value

```
let minCounterValue = 300;
let counterName = "Free Megabyes";
Perf
| project Computer
		, TimeGenereated
		, CounterName
		, CounterValue
| where CounterName == counterName
	and CounterValue <= minCounterValue
```
- reusing variable in query
```
let compName = "ContosoSQLSrv1";
Perf
| where Computer == compName
| extend ThisIsFor = strcat ("This data is for computer", compName)
| project ThisIsFor
		, TimeGenerated
		, CounterName
		, CounterValue
```

- let can be used to hold calculated values
```
let startDate = ago(12h);
Perf
| project Computer
		, TimeGenerated
		, CounterName
		, CounterValue
| where TimeGenerated >= startDate
```

- let can hold datasets to make things like union more readable
- let can hold a function
```
let dateDiffInDays = (date1: datetime, date2: datetime = datetime (2018-01-01))
					{ (date1 - date2) / 1d }
;
print dateDiffInDays(now(), todatetime("2018-05-01"))
```
## join
- join two tables together when they have a common column name
```
Perf
| where TimeGenerated >= ago(30d)
| take 1000
| join (Alert) on Computer
```
- when they don't have a common column you can still join them
```
Perf
| where TimeGenerated >= ago(30d)
| take 1000
| join (Alert) on $left.Computer == $right.Computer
```

## union
- takes two or more tables and returns the rows of all of them
- creates an output dataset that is the combination of two tables
```
UpdateSummary
| union Update
```
## Creating a Datatable
- generate a datatable that is define in the code itself
## prev and next
- prev gets the value from a column in previous row
- must use serialize operator in order to enable prev/next functionality
```
let SomeData = datatable ( rowNum:int, rowVal:string )
[
  1, "Value 01"
  2, "Value 02"
  3, "Value 03"	
  4, "Value 04"	
  5, "Value 05"	
  6, "Value 06"	
  7, "Value 07"	
  8, "Value 08"		
  9, "Value 09"	
];
SomeData
| serialize
| extend prvVal = strcat("Previous Value was", prev(rowVal))
```
- generate column called prvVal with value of previous row
- works the same for next(rowVal)
- specify how many rows back/forward `prev(rowVal, 2))

## toscalar 
- returns a scalar constant value of the evaluated expression. This function is useful for queries that require staged calculations e.g calculate a total count of events and then use the result to filter groups that exceed a certain percent of all events.
## row_cumsum
- provies a way to do a cumulative summary for values in a data set
- for each row, it adds a column
- cumulativeSum adding up the value in column 'a' up to that row
```
datatable (a:long)
[
	1, 2, 3, 4, 5, 6, 7, 8, 9, 10
]
| serialize cumulative Sum - row_cumsum(a)
```
## materialize
- allows caching a subquery result during the time of query execution in a way that other subqueries can reference the partial result
# Time series
## range
- produces a table in steps using the boundaries indicated, incrementing by the value in the step parameter
```
range LastWeek from ago(7d) to now() step 1d
```
## make-series
- take a series of values and converts them to an array of values within a column
```
Perf
| where TimeGenerated > ago(3d)
| where CounterName == "Available MBytes"
| make-series avg(CounterValue) default=0
			on TimeGenerated in rage(ago(3d), now(), 1h) by Computer
```
## series_stats
- takes a dynamic series of values and produces a list of all the statistical functions for them in on output
- series_stats_dynamic is used in conjunction with make-series to return statistics for a value

## series_outliers
- The function takes an expression with a dynamic numerical array as input, and generates a dynamic numeric array of the same length. Each value of the array indicates a score of a possible anomaly. A value greater than 1.5 in the same element of the input indicates a rise or decline anomaly. A values less than -1.5, indicates a decline anomaly

## series_fir
- FIR is Finite Impule Response time.
- It is typically used in digital signal processing, such as that used in radio
- the function takes an expression containing a dynamic numerical array as input and applies a FIR filter.
- By speciying the filter coefficients, it can be use for calculating a moving average, change-detection etc
```
// show a moving average of last 5 values
range t from bin(now(), 1h)-23h to bin (now(), 1h) step 1h
| summarize t-makelist(t)
| projecy val-dynamic([10,30,30,40,50,60,100]), t
| extend 5h MovingAvg= series fir(val, dynamic([1,1,1,1,1])),
		5h_MovingAvg_centred=series_fir(val, dynamic([1,1,1,1,1]))
| mvexpand val, t, 5h_MovingAvg, 5h_MovingAvg_centered
```

## series_iir
- Infinite Impluse Response
- The range of values is assumed to for on infinately rather than FIRs assumption it will be declining to zero
- Applies and IIR filter on a series
- The funct takes an expression containing dynamic numerical array as input and applies an Infinite Impulse Response filter
- By specifying the filter coefficients, function can be used:
	- to calculate the cumulative sum of the series
	- to apply smoothing operations

## series_fit_line
- performs a linear regression on a series. It returns multiple values:
- RSquare: stanard measure of the fit of quality, in a range of 0 to 1, the closer to 1, the better the fit
- Slope: slope of the approximated line
- Variance: the variance of the input data
- RVariance: residual variance (the variance between the input data)
- Interception: interception of the approcimated line
- Line_fit: numerical array holding the values of the best fit line. Typically used for charting
```
range x from1 to 1 step 1
| project x = rangebin(now(), 1h), 1h) 
		, y - dynamic([2,5,6,7,8,11,24,30,37])
| extend (RSquare,Slope,Variance,RVariance,Interception,LineFit) = series_fit_line(y)
```
## series_fit_2_lines
- takes the input data and splits the collection (using terms right side/ left side to differentiate)
- it then performs an analysis on each part of the data
- the best split is the one with the maximized RSquare.
- It will return the best values but you can also return the right and left values if you wish

- Function returns these parameters:
	- rsquare	R-square is standard measure of the fit quality. It's a number in the range [0-1], where 1 - is the best possible fit, and 0 means the data is unordered and do not fit any line.
	- split_idx	The index of breaking point to two segments (zero-based).
	- variance	Variance of the input data.
	- rvariance	Residual variance, which is the variance between the input data values the approximated ones (by the two line segments).
	- line_fit	Numerical array holding a series of values of the best fitted line. The series length is equal to the length of the input array. It's mainly used for charting.
	- right_rsquare	R-square of the line on the right side of the split, see series_fit_line().
	- right_slope	Slope of the right approximated line (of the form y=ax+b).
	- right_interception	Interception of the approximated left line (b from y=ax+b).
	- right_variance	Variance of the input data on the right side of the split.
	- right_rvariance	Residual variance of the input data on the right side of the split.
	- left_rsquare	R-square of the line on the left side of the split, see series_fit_line().
	- left_slope	Slope of the left approximated line (of the form y=ax+b).
	- left_interception	Interception of the approximated left line (of the form y=ax+b).
	- left_variance	Variance of the input data on the left side of the split.
	- left_rvariance	Residual variance of the input data on the left side of the split.

# Machine Learning
## basket
- Apiori works by first examining the frew of each distinct value in the list then if an item is not frequent, other combinations with that item wouldn't be considered frequent either and rhus are eliminated from consideration. Once it has a list of attributes that are frequent, it then analzes the combination of attributes for frequency
```
// analysis to see which combo of computer plus performance counters appears the most frequently
Perf
| where TimeGenerated >= ago(10d)
| project Computer
		, ObjectName
		, CounterName
		, InstanceName
| evaluate basket()
```
## autocluster
- looks for common patterns of discrete attributes in the data and reduces it to just a small number of patterns

Event
| where TimeGenerated >= ago(10d)
| project Source
		, EventLog
		, Computer
		, EventLevelName
		, RenderedDescription

| evaluate autocluster()

## diffpatterns
- takes a dataset and splits it into two halves based on two value in a specified column
- it then returns the most common set of attributes, showing how many were associated with the first value(A) and how many for the second value (B)
- here, take a set of attributes from the Event table, and split the data based on the EventLevelName column. Side A will be
- Error events, side b Warning events

```
Event
| where TimeGenerated >= ago(5d)
| project Source, Computer, EventID, EventCategory, EventLevelName
| evaluate diffpatterns(EventLevelName, 'Error, 'Warning')
```
## reduce
- used to determine patterns in string data e.g let's say you have 10 computers whose names all end in .ContosoRetail.com
- reduce will summarize them into the pattern of *.ContosoRetail.com, give you a count of the numer of occurences for this pattern, then in the Representative column show one example of this pattern

```
Perf
| where TimeGenerated >= ago(12h)
| project Computer
| reduce by Computer
```
- reduce has a threshold value, although it's implemented slightly different from the other functions.
- it should be in the range of 0 to 1, the default being 0.1.
- for larger inputs use a small value

```
Perf
| where TimeGenerated >= ago(12h)
| project Computer
| reduce by Computer with threshold = 0.6
```

# Exporting Data

