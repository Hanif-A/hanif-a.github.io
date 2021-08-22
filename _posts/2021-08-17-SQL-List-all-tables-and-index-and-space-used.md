List all tables, and indexes in a SQL Server database, and bring back their Used Space (in GB).

Uses dynamic SQL, elevated rights required.

```sql

DECLARE @Counter INT = 1
DECLARE @DBName VARCHAR(MAX)

-- SQL, saved in a variable, so we can run it dynamically while changing databases
DECLARE @SpaceSQL VARCHAR(MAX) = '
	INSERT INTO ##Space
	SELECT 
		DB_NAME(),
		t.NAME AS TableName,
		s.Name AS SchemaName,
		p.rows,
		CAST(ROUND(((SUM(a.total_pages) * 8) / 1024.00/1024.00), 2) AS NUMERIC(36, 2)) AS TotalSpaceGB,
		CAST(ROUND(((SUM(a.used_pages) * 8) / 1024.00/1024.00), 2) AS NUMERIC(36, 2)) AS UsedSpaceGB, 
		CAST(ROUND(((SUM(a.total_pages) - SUM(a.used_pages)) * 8) / 1024.00/1024.00, 2) AS NUMERIC(36, 2)) AS UnusedSpaceGB
	FROM 
		sys.tables t
	INNER JOIN      
		sys.indexes i ON t.OBJECT_ID = i.object_id
	INNER JOIN 
		sys.partitions p ON i.object_id = p.OBJECT_ID AND i.index_id = p.index_id
	INNER JOIN 
		sys.allocation_units a ON p.partition_id = a.container_id
	LEFT OUTER JOIN 
		sys.schemas s ON t.schema_id = s.schema_id
	WHERE 
		t.NAME NOT LIKE ''dt%''
		AND t.is_ms_shipped = 0
		AND i.OBJECT_ID > 255 
	GROUP BY 
		t.Name, s.Name, p.Rows
	ORDER BY 
		TotalSpaceGB DESC, t.Name'

DECLARE @IndexSQL VARCHAR(MAX) = '
INSERT INTO ##SpaceIndex
SELECT
		DB_NAME() [DBName]
	,	ix.[name] AS [Indexname]
	,	s.Name	[SchemaName]
	,	tn.Name [TableName]
	,	SUM(sz.[used_page_count]) * 8.00 / 1024.00 / 1024.00

FROM sys.dm_db_partition_stats AS sz

INNER JOIN sys.indexes AS ix
	ON sz.[object_id] = ix.[object_id] 
	AND sz.[index_id] = ix.[index_id]

INNER JOIN sys.tables tn
	ON tn.OBJECT_ID = ix.object_id

LEFT OUTER JOIN sys.schemas s
	ON tn.schema_id = s.schema_id

GROUP BY tn.[name], ix.[name], s.Name, tn.Name
ORDER BY tn.[name]
'

-- Output table - use two ## for table, so it becomes global (therefore accessible across scopes)
IF OBJECT_ID('tempdb.dbo.##Space', 'U') IS NOT NULL DROP TABLE ##Space
CREATE TABLE ##Space (
	DBName	VARCHAR(MAX)
,	TableName VARCHAR(MAX)
,	SchemaName VARCHAR(MAX)
,	[Rows]	int
,	[TotalSpaceGB]	decimal(8,2)
,	[UsedSpaceGB]	decimal(8, 2)
,	[UnusedSpaceGB]	decimal (8, 2)
)

IF OBJECT_ID('tempdb.dbo.##SpaceIndex', 'U') IS NOT NULL DROP TABLE ##SpaceIndex
CREATE TABLE ##SpaceIndex (
	DBName	VARCHAR(MAX)
,	IndexName VARCHAR(MAX)
,	SchemaName VARCHAR(MAX)
,	TableName	VARCHAR(MAX)
,	[UsedSpaceGB]	decimal(8, 2)
)

-- Get a list of all databases in this server
IF OBJECT_ID('tempdb.dbo.#DB', 'U') IS NOT NULL DROP TABLE #DB
SELECT 
		Name
	,	ROW_NUMBER() OVER(ORDER by NAME) AS RowNum
INTO #DB
FROM master.dbo.sysdatabases

-- Loop through every DB
WHILE @Counter < (SELECT MAX(RowNum) FROM #DB) BEGIN

	-- Connect to every database
	SET @DBName = (SELECT [Name] FROM #DB WHERE [RowNum] = @Counter)
	PRINT('Using: '+ @DBName)

	-- Execute everything into a single scope
	EXEC ('USE ' + @DBName + '; ' + @SpaceSQL)

	PRINT('Getting Index...')
	EXEC ('USE ' + @DBName + '; ' +@IndexSQL)

	SET @Counter = @Counter + 1

END

-- Return Data
SELECT 'Table'[IDX/TBL], DBName, TableName = SchemaName + '.' + TableName , IndexName = NULL, Rows, TotalSpaceGB, UsedSpaceGB, UnusedSpaceGB FROM ##Space
UNION ALL
SELECT 'Index', DBName, TableName = SchemaName + '.' + TableName, SI.IndexName, NULL, NULL, SI.UsedSpaceGB, NULL from ##SpaceIndex SI
ORDER BY UsedSpaceGB DESC

```
