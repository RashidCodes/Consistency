# How to enforce consistency in Facility Names in the Arrow Energy and Queensland GW Database

Charles Chaaya suggested that we replace all dashes ```-``` with spaces ```" "```. Steve also suggested that I replace all underscores ```_``` with spaces, and lastly, upon visual inspection, I replaced ```???``` with spaces and removed unnecessary asterisks ```*```.

<br/>

# Enforcing data consistency in the queensland GW database (MASTER_DATA table)

I have quite a substantial amount of experience with data manipulation in python, so I decided that instead of using SQL, I'll use python instead. The solution to the problem aforementioned is outlined in the steps below;

1. Create a new table called NEW_MASTER_TABLE using the SQL code below. The code replaces all dashes (```-```) with spaces (```" "```) in the **ORIG_NAME_NO** column as suggested by Charles Chaaya. The **ORIG_NAME_NO** column is renamed to **CONSISTENT_ORIG_NAME_NO**.

```sql
USE `DP_GWDBQLD_Master`;

-- MODIFYING THE DRILLED_DATE AND LOG_RECEVIED_DATE COLUMN; Because it contains some invalid datetime values
ALTER TABLE MASTER_DATA MODIFY COLUMN DRILLED_DATE VARCHAR(30), MODIFY COLUMN LOG_RECEIVED_DATE VARCHAR(30);

CREATE TABLE NEW_MASTER_DATA 
SELECT RN, FACILITY_TYPE, OFFICE, SHIRE_CODE, PARISH, RN_REPLACES, DO_FILE, RO_FILE, HO_FILE, FACILITY_STATUS, DRILLED_DATE, DRILLER_NAME, BASIN, METHOD_CONST, SUB_AREA, LOT, PLAN, DESCRIPTION, COUNTY, LAT, LNG, EASTING, NORTHING, ZONE, ACCURACY, GPS_ACCURACY, GIS_LAT, GIS_LNG, CHECKED, MAP_SCALE, MAP_SERIES, MAP_NO, PROG_SECT, EQUIPMENT, REPLACE(ORIG_NAME_NO, "-", " ") as CONSISTENT_ORIG_NAME_NO, POLYGON, CONFIDENTIAL, DATA_OWNER, BORE_LINE_CODE, DRILLER_LICENCE_NUMBER, LOG_RECEIVED_DATE, OBJECTID
FROM MASTER_DATA;
``` 

<br/>

2. To the extent of my knowledge in SQL, replacing ```??```, ```*```, etc. requires a few subqueries. Since I'm more productive with python, I decided to use that instead. 

Firstly, the data in the ```NEW_MASTER_DATA``` table is exported in a CSV format using the following steps;
- Remove the limit on the number of rows that can be displayed in the workbench
<img src="show limit.png" />

<img src="Remove limit.png">

<br/>

- Run the following SQL code. It simply selects all the data in the ```NEW_MASTER_DATA``` table.

```sql
-- This database should now contain the New Master data table
USE `DP_GWDBQLD_Master`;

-- Select everything from the New Master data table
SELECT * FROM NEW_MASTER_DATA;
```

<br/>

- Export the selection in a CSV format.
<img src="Export.png" />

Make sure to name the CSV file as ```Consistent Names.csv```.

<br/>

- The ```NEW_MASTER_DATA``` table is now in CSV format. Use the following python script to remove all unnecessary characters.

```python
import pandas as pd 
pd.set_option('display.max_columns', None)

def preprocess_text(df: pd.DataFrame, column_name: str) -> pd.DataFrame:
    """Preprocesses Well names
    

    Parameters
    ----------
    df: pd.DataFrame
        Arrow energy or Queensland data

    column_name: 
        The column that contains facility names


    Returns
    -------
    df: pd.DataFrame
        The modified dataframe
    """

    df[column_name] = df[column_name].str.replace('\?|\*|"', "", regex=True)
    df[column_name] = df[column_name].str.replace('_|-', " ", regex=True)
    df[column_name] = df[column_name].str.rstrip()
    df[column_name] = df[column_name].str.lstrip()

    return df 


consistent_names = pd.read_csv("Consistent Names.csv")
consistent_data = preprocess_text(consistent_names, "CONSISTENT_ORIG_NAME_NO")
consistent_data.to_csv("Consistent Names.csv", index=False)
```

<br/><br/>

# Enforcing data consistency in the Arrow Energy wells data

1. The Arrow Energy wells data has to be imported into the ```DP_GWDBQLD_Master``` database. This is done using the **Table Data Import Wizard** as shown in the screen capture below.

<img src="table wizard.png" />

The import wizard does not work with excel files, so the Arrow energy excel file will have to be converted into CSV format. In this case, the name of the newly created CSV file is ```WELLSArrow.csv```.

<br/>

2. Remove all unecessary characters using the python script below.

```python
arrow_wells = pd.read_csv("WELLSArrow.csv")
consistent_arrow_data = preprocess_text(arrow_wells, "Well Name")
consistent_arrow_data.to_csv("WELLSArrow.csv", index=False)
```

<br/><br/>

# Summary

1. Create a new table called **NEW_MASTER_DATA** on mysql workbench.
2. Export the content of this table in CSV format.
3. Remove all unnecessary characters using python (Creates clean version of **NEW_MASTER_DATA**).
4. Replace the existing **NEW_MASTER_DATA** table with the clean version. This involves dropping the existing **NEW_MASTER_DATA** table and uploading the clean version to the server.
5. Remove all unnecessary characters in the Arrow energy wells data.
6. Upload it to Mysql server using the **Table Data Import Wizard**.