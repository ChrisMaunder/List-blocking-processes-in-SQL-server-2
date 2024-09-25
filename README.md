# List blocking processes in SQL server

Here's an alternate version that doesn't use sp\_who yet provides a little more information. It also provides the option to kill the blocking processes themself.IF NOT EXISTS (SELECT \* FROM sys.objects WHERE object\_id = OBJECT\_ID(N'[dbo].[ListBlocking]') AND type in (N'P', N'PC'))EXEC...
Here's an alternate version that doesn't use sp\_who yet provides a little more information. It also provides the option to kill the blocking processes themself.

```cpp
IF NOT EXISTS (SELECT * FROM sys.objects 
WHERE object_id = OBJECT_ID(N'[dbo].[ListBlocking]') 
AND type in (N'P', N'PC'))
EXEC dbo.sp_executesql @statement = N'CREATE PROCEDURE [dbo].[ListBlocking] AS'
GO

/*===================================================================

Description:    Wrapper to sp_who2 to show only those processes that 
                are blocking. 
                
                Please see http://support.microsoft.com/kb/224453 for
                more info on locking.

===================================================================*/ 

ALTER procedure ListBlocking
(
    @KillOrphanedProcesses bit = 0, -- Kill any orphaned (SPID = -2) processes
    @KillBlockingProcesses bit = 0  -- Kill all blocking processes 
)
as

-- 0. Setup

SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED

IF OBJECT_ID('tempdb..#BlockingProcess') IS NOT NULL drop table #BlockingProcess

-- More items can be added to this if you wish
CREATE TABLE #BlockingProcess 
(
    BlockingProcessID int IDENTITY(1,1),
    ProcessID varchar(20),
    EventType varchar(100),
    Parameters varchar(100),
    EventInfo varchar(500),
    CPUTime int,
    DiskIO int,
    TransactionCount int,
    Command varchar(max),
    ObjectName varchar(100),
    TransactionIsolation varchar(50)
)

DECLARE @BlockingSPID           int,
        @CPUTime                int, 
        @DiskIO                 int,
        @TransactionCount       int,
        @Command                varchar(max), 
        @ObjectName             varchar(100), 
        @TransactionIsolation   varchar(50)

SET NOCOUNT ON

-- 1. Get a list of blocking processes. 
SELECT * 
INTO #ProcessRaw
FROM
(
    -- This SELECT based on Derek Dieter's sp_who3 code from  
    -- http://sqlserverplanet.com/dmv-queries/a-better-sp_who2-using-dmvs-sp_who3/
    SELECT SPID             = BlockingRequest.session_id, 
           Status           = Session.status,
           BlockedBy        = BlockingRequest.blocking_session_id,
           Command          = SUBSTRING(SqlText.text, BlockingRequest.statement_start_offset/2,
                                    (CASE WHEN BlockingRequest.statement_end_offset = -1
                                        THEN LEN(CONVERT(nvarchar(MAX), SqlText.text)) * 2
                                        ELSE BlockingRequest.statement_end_offset
                                        END - BlockingRequest.statement_start_offset)/2),
           ObjectName       = OBJECT_SCHEMA_NAME(SqlText.objectid,dbid) + '.' 
                            + OBJECT_NAME(SqlText.objectid, SqlText.dbid),
           StartTime        = BlockingRequest.start_time,
           ElapsedMS        = BlockingRequest.total_elapsed_time,
           CPUTime          = BlockingRequest.cpu_time,
           IOReads          = BlockingRequest.logical_reads + BlockingRequest.reads,
           IOWrites         = BlockingRequest.writes,
           LastWaitType     = BlockingRequest.last_wait_type,
           Protocol         = Connection.net_transport,
           TransactionIsolation =
                CASE Session.transaction_isolation_level
                    WHEN 0 THEN 'Unspecified'
                    WHEN 1 THEN 'Read Uncommitted'
                    WHEN 2 THEN 'Read Committed'
                    WHEN 3 THEN 'Repeatable'
                    WHEN 4 THEN 'Serializable'
                    WHEN 5 THEN 'Snapshot'
                END,
           ConnectionWrites = Connection.num_writes,
           ConnectionReads  = Connection.num_reads,
           ClientAddress    = Connection.client_net_address,
           Authentication   = Connection.auth_scheme,
           [Login]          = Session.login_name,
           Host             = Session.host_name,
           DBName           = DB_Name(BlockingRequest.database_id),
           CommandType      = BlockingRequest.command
        FROM sys.dm_exec_requests Request
        LEFT JOIN sys.dm_exec_requests BlockingRequest 
               ON BlockingRequest.session_id = Request.blocking_session_id
        LEFT JOIN sys.dm_exec_sessions Session 
               ON Session.session_id = BlockingRequest.session_id
        LEFT JOIN sys.dm_exec_connections Connection 
               ON Connection.session_id = Session.session_id
        OUTER APPLY sys.dm_exec_sql_text(BlockingRequest.sql_handle) as SqlText
        WHERE Request.session_id > 50
          AND Request.blocking_session_id <> 0
        --AND Session.login_name <> 'sa'
) as BlockingProcess
ORDER BY BlockingProcess.BlockedBy DESC, BlockingProcess.SPID

-- 2. Loop through this list either killing them or getting more information on them

DECLARE BlockingProcessCursor CURSOR FAST_FORWARD FOR 
    SELECT SPID, CPUTime, IOWrites + IOReads as DiskIO,
           Command, ObjectName, TransactionIsolation
    FROM #ProcessRaw
    GROUP BY SPID, CPUTime, IOWrites + IOReads, Command, ObjectName, TransactionIsolation
OPEN BlockingProcessCursor

FETCH NEXT FROM BlockingProcessCursor 
INTO @BlockingSPID, @CPUTime, @DiskIO, @Command, @ObjectName, @TransactionIsolation

WHILE @@FETCH_STATUS = 0
BEGIN

    -- For each blocked process get what information we can on the process and what's blocking it
    SET @TransactionCount = 0

    --  -2 = The blocking resource is owned by an orphaned distributed transaction.
    IF @BlockingSPID = '-2' 
    BEGIN
    
        -- For orphaned processes we need to get the Unit of work if we're to do anything with it
        DECLARE @UnitOfWork varchar(50)
        select top 1 @UnitOfWork = ISNULL(req_transactionUOW, '')
        from master..syslockinfo
        where req_spid = -2
            
        if @KillOrphanedProcesses = 1
        BEGIN
            if @UnitOfWork <> '' exec('KILL ''' + @UnitOfWork + '''')
            
            INSERT INTO #BlockingProcess (EventType, Parameters, EventInfo)
            VALUES('', '', '- Killed UOW ' + @UnitOfWork + ' -')
        END
        ELSE
        BEGIN
            INSERT INTO #BlockingProcess (EventType, Parameters, EventInfo)
            VALUES('', '', '- Orphaned. UOW = ' + @UnitOfWork + ' -')
        END
    END
    
    -- -3 = The blocking resource is owned by a deferred recovery transaction.
    ELSE IF @BlockingSPID = '-3' 
    BEGIN
        INSERT INTO #BlockingProcess (EventType, Parameters, EventInfo)
        VALUES('', '', '- deferred recovery transaction -')
    END
    
    -- -4 = Session ID of the blocking latch owner could not be determined at this 
    --      time because of internal latch state transitions.
    ELSE IF @BlockingSPID = '-4' 
    BEGIN
        INSERT INTO #BlockingProcess (EventType, Parameters, EventInfo)
        VALUES('', '', '- Latch owner could not be determined -')
    END

    -- Nothing unusual here. A process is being blocked by another processes, so let's get the
    -- info on the process that's doing the blocking
    ELSE
    BEGIN

        SELECT @TransactionCount = open_tran FROM master.sys.sysprocesses WHERE SPID=@BlockingSPID

        IF @BlockingSPID <> 0
        BEGIN
            if @KillBlockingProcesses = 1
            BEGIN
                exec('KILL ' + @BlockingSPID)

                INSERT INTO #BlockingProcess (EventType, Parameters, EventInfo)
                VALUES('', '', '- Killed Process ' + @BlockingSPID + ' -')
            END
            ELSE
            BEGIN
                INSERT INTO #BlockingProcess (EventType, Parameters, EventInfo)
                EXEC ('DBCC INPUTBUFFER(' + @BlockingSPID + ') WITH NO_INFOMSGS')
            END
        END
        ELSE
        BEGIN
            INSERT INTO #BlockingProcess (EventType, Parameters, EventInfo)
            VALUES ('', '', ' - Unable to determin PID - ')
        END
    END         
    
    -- Add some aggregate info on the blocking processes
    UPDATE #BlockingProcess 
    SET ProcessID            = @BlockingSPID,
        CPUTime              = @CPUTime,
        DiskIO               = @DiskIO,
        TransactionCount     = @TransactionCount,
        Command              = @Command,
        ObjectName           = @ObjectName, 
        TransactionIsolation = @TransactionIsolation
    WHERE BlockingProcessID = SCOPE_IDENTITY()
    
    FETCH NEXT FROM BlockingProcessCursor 
    INTO @BlockingSPID, @CPUTime, @DiskIO, @Command, @ObjectName, @TransactionIsolation
END

CLOSE BlockingProcessCursor
DEALLOCATE BlockingProcessCursor

-- 3. Display the results
SELECT ProcessID, EventInfo, Command, ObjectName, TransactionCount, TransactionIsolation, 
       CPUTime, DiskIO 
FROM #BlockingProcess

SET NOCOUNT OFF
```
