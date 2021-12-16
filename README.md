# Data Governance Project

- The project involves the creation of Master data, Meta data and ensuring high data quality for reporting purposes.


<br/>

## Creation of a new Master Data
The Master data table is basically a better version of the queensland data. The idea is to:

- Remove all duplicate tuples
- Remove all tuples with empty values in the ```ORIG_NAME_NO``` and ```RN``` columns.
- Remove all special characters

To create the Master data, you need access to the old Master table. Run the code below to create the new Master data table.

```sql
/* This statement creates a new table called NEW_MASTER_DATA. It removes all special characters in the ORIG_NAME_NO column. Lastly, it removes all duplicated tuples in the Queensland dataset.
*/
USE `DP_GWDBQLD_Master`;

CREATE TABLE NEW_MASTER_DATA
SELECT DISTINCT ORIG_NAME_NO, LOWER(REGEXP_REPLACE(ORIG_NAME_NO, '[^\\\\x20-\\\\x7E]|<|>|=|:|;|@|\;|\\?|\\[', '')) CLEAN_ORIG_NAME_NO, RN, FACILITY_TYPE, OFFICE, SHIRE_CODE, PARISH, RN_REPLACES, DO_FILE, RO_FILE, HO_FILE, FACILITY_STATUS, DRILLED_DATE, DRILLER_NAME, BASIN, METHOD_CONST, SUB_AREA, LOT, PLAN,
DESCRIPTION, COUNTY, LAT, LNG, EASTING, NORTHING, ZONE, ACCURACY, GPS_ACCURACY, GIS_LAT, GIS_LNG, CHECKED, MAP_SCALE, MAP_SERIES, MAP_NO, PROG_SECT, EQUIPMENT, POLYGON, CONFIDENTIAL,
DATA_OWNER, BORE_LINE_CODE, DRILLER_LICENCE_NUMBER, LOG_RECEIVED_DATE, OBJECTID
FROM MASTER_DATA
WHERE RN != "" AND ORIG_NAME_NO != "";
```

<br/>

## The Arrow Energy table
Simply use the ***Table Import Wizard*** to import the arrow energy data into the database. You may have to convert the original file to CSV format.

<br/>

# Mapping between Queensland and New master data
This part is very tricky. There are several ways one could go about solving this problem. But here's my idea.

1. Create a new column in the ```NEW_MASTER_DATA``` table called ```CLEAN_ORIG_NAME_NO```. This column is a copy of the ```ORIG_NAME_NO``` column however all special characters have been removed. All entries are also made to be lowercase.

2. Repeat Step 1 on the Arrow energy data (```wellsarrow```). This time, the new column will be called ```Clean Well Name```. The ```Clean Well Name``` is a copy of the existing ```Well Name``` column in the Arrow energy data.

3. For each ```CLEAN_ORIG_NAME_NO``` value in the ```NEW_MASTER_DATA``` table, find an exact match in the ```Clean Well Name``` column of the ```wellsarrow``` table.

4. If a match is found, save the details of the facility in a new table called ```storeResults```. The new table must have 0 duplicates of facility names.

<br/>

The code below can be used to complete the steps aforementioned. 

## ```THE RESULT OF ALL DATA MANIPULATION TASKS WERE STORED IN TEMPORARY TABLES```.

<br/>

### Step 1:
```sql
-- Temporary table for the New master data
CREATE TEMPORARY TABLE realRN 
SELECT DISTINCT
    ORIG_NAME_NO, 
    CLEAN_ORIG_NAME_NO,
    RN
FROM NEW_MASTER_DATA;
```
<br/>

### Step 2:
```sql
-- Create a new temporary table for Arrow energy Well data
SET @arrow_number=0; -- dummy primary key
CREATE TEMPORARY TABLE realArrow 
SELECT 
	(@arrow_number:=@arrow_number + 1) AS ARROW_ROW_NUMBER, 
    `Well Name`,
	LOWER(REGEXP_REPLACE(`Well Name`, '[^\\\\x20-\\\\x7E]|<|>|=|:|;|@|\;|\\?|\\[', '')) as `Clean Well Name`
FROM wellsarrow;
```

<br/>

### Step 3:
```sql
-- Create new temporary table to store results
CREATE TEMPORARY TABLE storeResults(
    WellName VARCHAR(50),
	QueenslandID INT,
    ArrowID INT
);

```

<br/>

## The mapping stored procedure
```sql
DELIMITER $$
DROP PROCEDURE IF EXISTS matchProc $$
CREATE PROCEDURE matchProc()
BEGIN
	DECLARE orig_name, clean_orig_name  VARCHAR(50);
    DECLARE queensland_row_num, registration_rn, well_num, was_found, already_added INT;
    DECLARE done INT;
	DECLARE totalRecords INT DEFAULT 5;
    DECLARE record_number INT DEFAULT 0;
    DECLARE new_cur CURSOR FOR SELECT * FROM realRN;
    
    SELECT COUNT(*) INTO totalRecords FROM realRN;

    
    -- OPEN new_cur;
    OPEN new_cur;
    read_loop: LOOP
		IF record_number = totalRecords THEN
			LEAVE read_loop;
		END IF;
        
        FETCH new_cur INTO orig_name, clean_orig_name, registration_rn;
		SELECT Result INTO was_found FROM (SELECT clean_orig_name IN (SELECT `Clean Well Name` FROM realArrow) as Result) as new_table;
	
        
        IF was_found = 1 THEN
			-- Find out we've already recorded the name of the facility
            SELECT already_available INTO already_added FROM (SELECT orig_name IN (SELECT WellName FROM storeResults) as already_available) as t1;
            
            -- Add the facility only if it does not exists
            IF already_added = 0 THEN
				SELECT ARROW_ROW_NUMBER INTO well_num FROM realArrow WHERE `Clean Well Name` = clean_orig_name LIMIT 1;
				INSERT INTO storeResults(WellName, QueenslandID, ArrowID) VALUES (orig_name, registration_rn, well_num);
			END IF;
            
		END IF;
        
        IF was_found = 0 THEN
			-- Check if the facility has already been recorded
			SELECT already_available INTO already_added FROM (SELECT orig_name IN (SELECT WellName FROM storeResults) as already_available) as t2;
            
            -- Add the facility only if it does not exist
            IF already_added = 0 THEN 
				INSERT INTO storeResults(WellName, QueenslandID, ArrowID) VALUES (orig_name, registration_rn, -1);
			END IF;
		END IF;
        
        SET record_number = record_number + 1;
        
        
	END LOOP read_loop;
    SELECT record_number;
    
	-- close the cursor
    CLOSE new_cur;
        
END;
$$

CALL matchProc();
```
<br/>

# Statistics

```sql
-- Unmatched facility names
SELECT COUNT(ArrowID) `Unmatched IDs` FROM storeResults where ArrowID = -1; -- 44268
```

```sql
-- Matched facility names
SELECT COUNT(ArrowID) `Matched IDs` FROM storeResults where ArrowID != -1; -- 195
```


```sql
-- Empty tuples in the old master table
SELECT COUNT(*) Empty_Values FROM MASTER_DATA WHERE ORIG_NAME_NO = ""; -- 108102
```

<br/>

# Results
The results need to be cleaned up before delivery to the client. Use the SQL code below to clean up the ```ORIG_NAME_NO``` column for display purposes only.

```sql
-- Replace all dashes with spaces and remove all special characters
SELECT REGEXP_REPLACE(WellName, "-|_", " ") as WellName, QueenslandID, ArrowID
FROM ( 
	SELECT REGEXP_REPLACE(WellName, "[#?+!&%$\.\\[\\]\*\"\'><=@:;(){}/]", "") as WellName, QueenslandID, ArrowID FROM storeResults
) as t1
```

The data is now clean, and can be delivered to the client for his review.


