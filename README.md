# social-dataset-query-engine
Dynamic Query Engine on the Social Dataset 

The Social Data Dynamic Query Engine is a java module allowing to perform dynamic (search) queries and extractions from the Social dataset.

This social dataset is build upon the:
 + Twitter tweet [Twitter Parser](https://github.com/faucon-consortium/twitter-tweet-parser)
 + Linkedin Profiles (on work)
 + Facebook Graph (on study)

Those various datasets are stored (indexed) in MongoDb/ElastiSearch. By this way, we can perform many and various data manipulating and analysis operations.

-----------------------------

The query request is formalized through a JSON message and look like the SQL syntax  
Here is the JSON Schema:

```
 root
 |-- select: array (nullable = false)
 |    |-- element: string (containsNull = false)
 |-- from: string (nullable = false)
 |-- where: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- clause: struct (nullable = true)
 |    |    |    |-- parenthesis: string (nullable = true)
 |    |    |    |-- condition: struct (nullable = true)
 |    |    |    |    |-- index: string (nullable = true)
 |    |    |    |    |-- field: string (nullable = true)
 |    |    |    |    |-- operator: string (nullable = true)
 |    |    |    |    |-- value: string (nullable = true)
 |    |    |    |-- logicalOp: string (nullable = true)
 |-- groupBy: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- missing: boolean (nullable = true)
 |    |    |-- name: string (nullable = true)
 |    |    |-- order: string (nullable = true)
 |-- orderBy: array (nullable = true)
 |    |-- element: string (containsNull = true)
 |-- prettyPrint: boolean (nullable = true)
 
```
-----------------------------
### Fields Description:
#### -- select: array of String
This field, list the elements data attributes (fields related to a dataset and based on his schema JSON) to retrieve the suitable data. The attrbute name data is expressed by its **full JSON name**. This means that the name of the field corresponds to the **full path access** to the field with *each tree level* separated by a point.  

##### Example: #####
```
"select": ["id", "user.id"]
```
Wildcards can be used to build the field name:
```
"select": ["user.*"]
or
"select": ["*"]
```
The fisrt example aims to retrieve **all recursive childs elements** from the *root element*: ***server*** and the last one means retrieving all data from the JSON root.  


There is a special syntax that is used to express data aggregations. The supported aggregations are: COUNT, SUM, AVG, MAX and MIN

Here are some examples:

- **```"select" : ["MAX(user.followers_count) AS Nbfollowers"]```**
   Get the maximum followers_count value. The "AS" clause (case sensitive) allow to define the name of the field that will be reported for this aggregation in the result JSON document.
- **```"select" : ["MIN(user.followers_count)"]```**
   Get the minimum followers count value. As there is no "AS" clause, the name of the field returned will be ```"MIN_"<full field name>```.  
   In the example above, the name of the field returned will be: ```"MIN_followers.count"```
- **```"select" : ["SUM(user.followers_count) AS TotaFollowers"]```**
   Get the summation of all users Followers  
   
- **```"select" : ["COUNT(user.followers_count)"]```**
   Get the number of user who have followers
    
- **```"select" : ["AVG(user.followers_count)"]```**
   Get the average followers count   


#### -- from: String
This field define the collection name (look like a Table in SQL) on wich the requests are performed  


#### -- where: List<Clause>
This field define a list of filters, applied to the data retreival. 
##### Parameters definition: #####
  + <ins>parenthesis</ins>: STRING
  This field take two values, **"START"** to open a parenthesis "(" and **"END"** to close the parenthesis ")"
  + <ins>condition</ins>
  This field define the filter conditions applied to the data fields
    + <ins>index</ins>: STRING
    The name of the index on wich perform the filter
    + <ins>field</ins>: STRING
    The full name of the field on wich the condition applied
    + <ins>operator</ins>: STRING
    Define the operator condition. The following operators are supported:
      + "="
      + "!="
      + "<"
      + "<="
      + ">"
      + ">="
      + "[]" (range of data with bounds included)
      + "{}" (range of data with bounds excluded)
      + <ins>value</ins>
    Define the field value of the condition. There is a special value: "NULL" to express the nullity of a field.
    + <ins>logicalOp</ins>
  Define the logical operator used to *link* two conditions. Two values are supported: "AND" and "OR" logical operator.  
  
  
For example, the below sample can be expressed in *SQL Syntax* as : ```"where customer.dob IS NOT NULL AND (customer.civility="Mme" OR customer.civility="M")```
```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "customers",
        "field" : "customer.dob",
        "operator" : "!=",
        "value" : "NULL"
      },
      "logicalOp" : "AND"
    }
  }, {
    "clause" : {
      "parenthesis" : "START",
      "condition" : {
        "index" : "customers",
        "field" : "customer.civility",
        "operator" : "=",
        "value" : "Mme"
      },
      "logicalOp" : "OR"
    }
  }, {
    "clause" : {
      "parenthesis" : "END",
      "condition" : {
        "index" : "customers",
        "field" : "customer.civility",
        "operator" : "=",
        "value" : "M"
      }
    }
  } ]
```
##### Dates condition: #####
All the dates are stored in format **"yyyy-MM-dd'T'HH:mm:ss"** with:
+ ```yyyy```: Year in four digits (ex: 2020)
+ ```MM```: The month number from the year, in two digits (ex: 09, ex:11)
+ ```dd```: The day number from the month, in two digits (ex: 02, ex:31)
+ ```HH```: Hour in 24h format (ex:09, ex:17)
+ ```mm```: Minutes (ex:23, ex:47)
+ ```ss```: Seconds (ex:05, ex:33)

Example:
2020-02-10T06:26:10
  
You can query the date fields in different format:
+ By Year (```yyyy```): Get all documents from the same year
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020}"
      }
    }
  } ]
  ```
+ By Month (```yyyy-MM```): Get all documents from the same month and year
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020-02"
      }
    }
  } ]
  ```
+ By Day (```yyyy-MM-dd```): Get all documents from the same day, month and year
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020-02-29"
      }
    }
  } ]
  ```
+ By Hour (```yyyy-MM-ddTHH```): Get all documents from the same hour, day, month and year
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020-02-29T06"
      }
    }
  } ]
  ```
+ By Minutes (```yyyy-MM-ddTHH:mm```): Get all documents from the same minute, hour, day, month and year
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020-02-29T06:25"
      }
    }
  } ]
  ```

+ By Seconds (```yyyy-MM-ddTHH:mm:ss```): Get all documents from the same second, minute, hour, day, month and year
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020-02-29T06:25:17"
      }
    }
  } ]
  ```
+ By Milliseconds (```yyyy-MM-ddTHH:mm:ss.SSSSSS```): Get all documents from the same DateTime
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020-02-29T06:25:17.213070"
      }
    }
  } ]
  ```

+ By Range:
  You can specify a date range like that:
   ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "[2020-02-29T06,2020-02-29T08}"
      }
    }
  } ]
  ```
The **[** and **]** characters defines **inclusive bounds**. On the other side, the **{** and **}** characters defines **exclusive bounds**.


#### -- groupBy: List<GroupBy>
Defines the fields to be grouped for the calculation of aggregations.
##### Paremeters Definition: #####

  + <ins>name</ins>: String
  The name of the field to be grouped
  + <ins>missing</ins>: Boolean
  If true an explicit null bucket will represent documents with missing values.
  + <ins>order</ins>: String
  Define the *groupBy* sort order: "ASC" for ascending sort and "DESC" for descending sort

##### Example: #####
 Group by customer civility in ascending order (first:"M" and second:"Mme") and for each civility, group by descending customer age
 ```
 "groupBy" : [ {
    "name" : "customer.civility",
    "order" : "ASC",
    "missing" : true
  }, {
    "name" : "cutomer.age",
    "order" : "DESC",
    "missing" : true
  } ],
```

Can be expressed in *SQL Syntax*  as : ```"GROUP BY customer.civility,customer.age```


#### -- orderBy: List<String>
Define the *general (or extended)* sort order.
The syntax is: "<*Sort Order*> (*data field fullname*)". "Sort Order" can take two values: "ASC" for ascending sort and "DESC" for descending sort.

##### Example: #####
```
"orderBy" : [ "ASC (customer.nbVisit)" ]
```

#### -- prettyPrint: Boolean
If set to **true**, the JSON results message is retuned as a formated string whith indent and line break. If set to **false**, the JSON message is compacted in one line.
