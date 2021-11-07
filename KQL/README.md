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
