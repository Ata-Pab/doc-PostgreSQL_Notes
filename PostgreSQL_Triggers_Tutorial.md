## Basic PostgreSQL Triggers

- Trigger is a function invoked automatically whenever an event associated with a table occurs. An event could be any of the following: ```INSERT```, ```UPDATE```, ```DELETE``` or ```TRUNCATE```.
- The trigger will be associated with the specified table, view, or foreign table and will execute the **specified function** when certain operations are performed on that table. 
 

### Trigger Types

1. Row Level Trigger: If the trigger is marked ```FOR EACH ROW``` then the trigger function will be called for each row that is getting modified by the event.
2. Statement Level Trigger: The ```FOR EACH STATEMENT``` option will call the trigger function only once for each statement, regardless of the number of the rows getting modified.

### Create a Trigger

```sql
-- Create a test DB as called Company
CREATE TABLE COMPANY(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),
   SALARY         REAL
);

-- To keep audit trial, we will create a new table called AUDIT where log messages will be inserted whenever there is an entry in COMPANY table for a new record
CREATE TABLE AUDIT(
   EMP_ID INT NOT NULL,
   ENTRY_DATE TEXT NOT NULL
);

-- Here, ID is the AUDIT record ID, and EMP_ID is the ID, which will come from COMPANY table, and DATE will keep timestamp when the record will be created in COMPANY table. So now, let us create a trigger on COMPANY table as follows
CREATE TRIGGER example_trigger AFTER INSERT ON COMPANY
FOR EACH ROW EXECUTE PROCEDURE auditlogfunc();

CREATE OR REPLACE FUNCTION auditlogfunc() RETURNS TRIGGER AS $example_table$
   BEGIN
      INSERT INTO AUDIT(EMP_ID, ENTRY_DATE) VALUES (new.ID, current_timestamp);
      RETURN NEW;
   END;
$example_table$ LANGUAGE plpgsql;

-- Insert new company data for testing
INSERT INTO COMPANY (ID,NAME,AGE,ADDRESS,SALARY)
VALUES (1, 'Paul', 32, 'California', 20000.00 );
```
**OUTPUTS**:<br/>
**COMPANY Table**

```txt
id | name | age | address      | salary
----+------+-----+--------------+--------
  1 | Paul |  32 | California   |  20000
```

**AUDIT Table**
```text
 emp_id |          entry_date
--------+-------------------------------
      1 | 2013-05-05 15:49:59.968+05:30
```

### Listing Triggers
```sql
-- List down all the triggers in the current database from pg_trigger table
SELECT * FROM pg_trigger;

-- List the triggers on a particular table, then use AND clause with table name as follows
SELECT tgname FROM pg_trigger, pg_class WHERE tgrelid=pg_class.oid AND relname='company';
```

### Dropping Triggers
```sql
DROP TRIGGER trigger_name;
```





















