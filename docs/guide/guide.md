# Data Access Service Reference Guide

## Basics
---

### JSON

Plain JSON is chosen over JSON schema for describing the structure of service input, output and object, for a couple of reasons:

- Absolutely zero learning, and
- Sufficient to describe the structure of the data object and the data type of the data fields

Data access service (DAS) does not perform data validation but leaves it to service caller.

### JSON Structure

DAS supports the following data structures inside the service input, output and object, respectively.

| Item | Structure |
| ---- | ---- |
| ***Input*** | value field |
|  | array of values |
|  | object |
| ***Output*** | value field |
|  | array of values |
|  | object |
|  | array of objects |
| ***Object*** | value field |
|  | array of values |
|  | object |
|  | array of objects |

> Note: Array of array is not supported.

#### JSON Data Type

JSON supports three native data types: boolean, number and string. It is extended to support two additional types: datetime and binary. The chart below illustrates how these data types should be encoded in JSON string.

| Data Type     | Encoding |  Examples |
| ------------- | ------------- | ----------------- |
| boolean  | native  | true, false |
| number  | native  | 123, 1.23 |
| string  | native  | "abc" |
| datetime  | ISO 8601 datetime string  | "2019-04-15T15:02 :54.123Z"      |
| binary  | base64 or base64 string prefixed with "base64" | "base64#gghsdh=", "#gghsdh="   |

The above applies both to the JSON data sent in the service request and response and to the sample JSON used to specify the service input, output and object.

### JSON Path

DAS supports a tailed version of JSON path for specifying the data fields or structures of the service input, output and object in various data bindings. Specifically, it uses a dot `"."` to indicate an object structure and a double-dot `".."` to indicate an array structure. Thus for the following sample structure:

```json
{
    "myValueField": "abc",
    "myValueList": ["abc", "xyz"],
    "myObject": {
        "myObjectField": "abc"
    }
    "myArray": [{
        "myArrayField": "abc",
        "myChildArray": [{
            "myChildArrayField": "abc"
        }] 
    }]
}
```

we have

| Field Name | Field Path |
| ---- | ---- |
| myValueFiled | .myValueField |
| myValueList | .myValueList |
| myObjectField | .myObject.myObjectField |
| myArrayField | .myArray..myArrayField |
| myChildArrayField | .myArray..myChildArray..myChildArrayField |

and

| **Object Name** | **Object Path** |
| ---- | ---- |
| root | . |
| myObject | .myObject. |
| myArray | .myArray.. |
| myChildArray | .myArray..myChildArray.. |

The field path ends with the field name. The object path ends with a dot `"."`, if an object, or a double-dot `".."`, if an array.


### Database

To start, DAS only supports MySQL database.

#### MYSQL

The chart below lists the MYSQL data types supported by DAS and how they are mapped to JSON data types.

| JSON Data Type     | MySQL Data Type |
| ------------- | ------------- |
| ***boolean***  | boolean  |
| ***number***  | int  |
|    | tinyint  |
|    | smallint |
|    | mediumint |
|    | bigint |
|    | float |
|    | double |
|    | decimal |
| ***string***  | char  |
|    | varchar  |
|    | text  |
|    | tinytext  |
|    | mediumtext  |
|    | longtext  |
| ***datetime***  | date  |
|    | datetime  |
|    | timestamp  |
| ***binary***  | blob  |
|    | tinyblob  |
|    | mediumblob  |
|    | longblob  |


### Database Transaction

For data access service, database transaction is implicit and abstracted away from developer. Each service request forms a database transaction. 

### Service Request

The data access service is invoked by a HTTP service request for the service. The following is a sample service request for a hypothetic query service:

``` http
Request URL: http://instance:8080/workspace/app/mod/getCustomers
METHOD: POST
Content-Type: application/json
Body:
{
    "state": "CA",
    "city": "Los Angeles"
}
```

The request method is always `POST`. The Content-Type header is `application/json`. The request body is the service input. The request URL is:  

`https://baseUrl/application-name/module-name/service-name`

for query and SQL services, and  

`https://baseUrl/application-name/module-name/service-name/operation`  

for CRUD service, where operation are `read`, `create`, `delete`, `update`, `save` or `merge`.

The baseUrls depends on the workspace for devtime and instance for runtime service.

### Service Response

The data access service responds the service request with a service response. The following is a sample service response for a hypothetic query service:

```http
Status: 200
Body: 
[
    {
        "customerId": 123,
        "customerName": "John",
        "address": {
            "street": "123 main st",
            "city": "Los Angeles",
            "state": "CA"
        }
    },
    {
        "customerId": 456,
        "customerName": "Smith",
        "address": {
            "street": "456 grand ave",
            "city": "Los Angeles",
            "state": "CA"
        }
    }
]
```

The status is `200` if the service is successful, `500` if a server error, or `400` if a service exception.

For `200` status, the response body is the service output. Otherwise, the response body is an exception message, like:

```http
Status: 400
Body: 
{
    "name": "InvalidInput",
    "type": "ServiceException",
    "message": "Input is invalid."
}
```

## Data Access Application
---

### General

A data access application is a collection of data access services organized into modules. It is developed and deployed as one unit. 

#### Database Type

An application is specific to a database type, MySQL for example. The database type is specified when the application is created. It should be consistent with the database type of the data source associated with the application at both dev and run time.

#### Data Source

The data source is created before the application can be deployed, with a data source configuration file shown below:

```
{
    "dbType": "mysql",
    "host": "myHost",
    "port": 3306,
    "database": "myDatabase",
    "username": "myUsername",
    "password": "myPassword"
}
```

Multiple applications may be associated with the same data source. In this case, the applications may use the same or different schemas.

### File Structure

The following shows the source file structure of an application in `Service Builder`. It is automatically created as the application, module, service, and test are created.

``` cm
myApplication/
    src/
        application.json
        myModule/
            module.json
            myQueryService/
                service.json
                input.json
                output.json
                query.sql
                input-bindings.json
                output-bindings.json
                tests/
                    testMyQueryService.json
                    ...
            mySqlService/
                service.json
                input.json
                output.json
                sqls.sql
                query.sql
                input-bindings.json
                output-bindings.json
                tests/
                    testMySqlService.json
                    ...
            MyCrudObject/
                service.json
                object.json
                read/
                    input.json
                    query.sql
                    input-bindings.json
                    output-bindings.json
                write/
                    tables.json
                    my-root-table.columns.json
                    my-child-table.columns.json
                    ...
                tests/
                    testReadMyCrudObject.json
                    testCreateMyCrudObject.json
                    testUpdateMyCrudObject.json
                    testDeleteMyCrudObject.json
                    ...
    README.md
```

### Application File

Each application includes an application file: `application.json`, as shown below:

```json
{
    "name": "myApplication",
    "description": "application. Don't modify this file!",
    "dbType": "mysql",
    "dataSource": "mydb",
    "schema": "classicmodels"
}
```

It is automatically generated when the application is created, and contains information regarding the application. The user is expected to configure a data source, and optionally a database schema, for the application.

### Module File

Each module includes a module file: `module.json`, as shown below

```json
{
    "name": "myModule",
    "description": "module. Don't modify this file!"
}
```

This file is automatically generated when the module is created. The use is not expected to make change to this file.

### Service File

Each service include a service file: `service.json`, along with files for the various components of the service. The content of the service file varies with the type of service. The service file is automatically generated when the service is created. The user is not expected to make change to this file, except when the user wants:

- To enable dynamic query for a query service, by setting its `dynamic` property to `true`; or
- To allow substitution variable for a SQL service, by setting its `variableLength` property to a number greater than zero.


#### Query Service File

```json
{
    "name": "myQueryService",
    "type": "query",
    "description": "query service. Don't modify this file except the dynamic field!",
    "input": "./input.json",
    "output": "./output.json",
    "query": "./query.sql",
    "dynamic": false,
    "inputBindings": "./input-bindings.json",
    "outputBindings": "./output-bindings.json"
}
```

#### SQL Service File

```json
{
    "name": "mySqlService",
    "type": "sql",
    "description": "sql service. Don't modify this file except the variableLength field!",
    "input": "./input.json",
    "output": "./output.json",
    "sqls": "./sqls.sql",
    "variableLength": 0,
    "query": "./query.sql",
    "inputBindings": "./input-bindings.json",
    "outputBindings": "./output-bindings.json"
}}
```

#### CRUD Service File

```json
{
    "name": "MyCrudObject",
    "type": "crud",
    "description": "crud service. Don't modify this file!",
    "object": "./object.json",
    "read": {
        "input": "./read/input.json",
        "query": "./read/query.sql",
        "inputBinding": "./read/input-binding.json",
        "outputBinding": "./read/output-binding.json"
    },
    "write": {
        "tables": "./write/tables.json"
    }
}
```

## Query Service
---

The query service is for retrieving data from database. It comprises

- an input
- an output
- a query
- an input bindings component that maps the input fields to the query parameters, and
- an output bindings component that maps the output fields to the query columns 

The query service supports a single query operation.

### File Structure

The file structure of **query service** is as follows:

``` cmd
myQueryService/
    service.json
    input.json
    output.json
    query.sql
    input-bindings.json
    output-bindings.json
    tests/
        testMyQueryService.json
```

It is generated when the service is created. There is one file corresponding to each of component of the query service. The user is expected to complete the input, output, query, input bindings and output bindings components, with tools provided by Service Builder and third-party, such as Database Client and JSON Grid Viewer, etc., available in the VSCode environment.

### Example

Here is an example of query service to retrieve a list of customer objects by city and state.

#### Input

``` json
{
    "city": "Los Angeles",
    "state": "CA"
}
```

The **input** of the query service is a simple object carrying the query parameters.

#### Output

``` json
[{
    "customerId": 123,
    "customerName": "John",
    "address": {
        "street": "123 main st",
        "city": "Los Angeles",
        "state": "CA"
    }
}]
```

The **output** of the query service may be an object or array of objects. In this example, the output is an array of `customer` objects.

If the query service is to return a single `customer` object by `customerId` instead, the output would be an object, like

``` json
{
    "customerNumber": 123,
    "customerName": "John",
    "address": {
        "address": "123 main st",
        "city": "Los Angeles",
        "state": "CA"
    }
}
```

#### Query

```sql
SELECT customerNumber, customerName,
       addressLine1 as address, city, state    
  FROM classicmodels.customers
 WHERE state = :state
   AND city = :city
```

The **query** component of the query service is a single SELECT statement. The query parameters in the SQL are prefixed with `":"`.

Besides the value parameters like `:state` in the above query, DAS also supports list parameters. For example, the `:customerNames` in the following query is a list parameter:

```sql
SELECT customerNumber, customerName,
       addressLine1 as address, city, state    
  FROM classicmodels.customers
 WHERE customerName in ( :customerNames )
```

The list parameter is to be mapped to a list field in the input. For example, the `.customerNames` field in the following input object:

``` json
{
    "customerNames": ["John", "Smith"]
}
```

#### Input Bindings

``` json
[
    {
        "parameter": "city",
        "field": ".city"
    },
    {
        "parameter": "state",
        "field": ".state"
    }
]
```

The **input bindings** of the query service map the query parameters to the input fields. The `parameter` is the name of the parameter. The `field` is the json path of the data field in the input object.

#### Output Bindings

``` json
[
    {
        "field": "..customerNumber",
        "column": "customerNumber"
    },
    {
        "field": "..customerName",
        "column": "customerName"
    },
    {
        "field": "..address.address",
        "column": "address"
    },
    {
        "field": "..address.city",
        "column": "city"
    }
    {
        "field": "..address.state",
        "column": "state"
    }
]
```

The **output bindings** of the query service map the query columns to the output fields. The column is the alias or the name of the column; the field is the json path of the data field of the output object.

> The user is expected to generate the input and output bindings from the input, output and query with `Service Builder`, and then review and edit them as needed. The `Service Builder` is expected to generate the input and output bindings correctly in normal case, but the user is ultimately responsible for the accuracy of the input and output bindings.

### Dynamic Query

DAS supports dynamic query, or query that changes at runtime with the parameters present in the service input. The dynamic query may be enabled by setting the `dynamic` property to `true` in the service file. For example,

```json
{
    "name": "getCustomers",
    "type": "query",
    "description": "query service. Don't modify this file except the dynamic field!",
    "input": "./input.json",
    "output": "./output.json",
    "query": "./query.sql",

    "dynamic": true,

    "inputBindings": "./input-bindings.json",
    "outputBindings": "./output-bindings.json"
}
```

The dynamic query changes at runtime with the parameters present in the service input. if a parameter is not present in the input, the line(s) containing the parameter is dropped from the query.

For example, if the query service in the example above made dynamic and a service call is initiated with input, like

``` json
{
    "state": "CA"
}
```

The original query

```sql
SELECT customerNumber, customerName,
       addressLine1 as address, city, state    
  FROM classicmodels.customers
 WHERE state = :state
   AND city = :city
```

will change into

```sql
SELECT customerNumber, customerName,
       addressLine1 as address, city, state    
  FROM classicmodels.customers
 WHERE state = :state
```

which is a query by state.

Or if a service call is made with an empty input, like

``` json
{}
```

The query will change into

```sql
SELECT customerNumber, customerName,
       addressLine1 as address, city, state    
  FROM classicmodels.customers
```

which is a query for all customers.

It follows that, if a service call is made with an input, like

``` json
{"city": "Los Angels"}
```

the query will become

```sql
SELECT customerNumber, customerName,
       addressLine1 as address, city, state    
   AND city = :city
```

It is an invalid query, and an error will be thrown for illegal SQL syntax.

> Caution must be exercised when developing dynamic query service.

## SQL Service
---

The SQL service is a command service for manipulating data in the data source. It comprises

- an input
- a sqls component
- an input bindings component that map the input fields to the sql parameters
- an optional output
- an optional query, and
- an optional output bindings component that map the output fields to the query columns, if the query is specified

In SQL service, the query is optional. If no query is specified, the service return nothing. If the query is specified, however, the output and output bindings must also be specified.

The SQL service support a single `execute` operation.

### File Structure

The file structure of the SQL service is as follows:

``` cmd
mySqlService/
    service.json
    input.json
    output.json
    sqls.sql
    query.sql
    input-bindings.json
    output-bindings.json
    tests/
        testMySqlService.json
```

This file structure is generated when the service is created. There is a file for each component of the SQL service. The user is expected to complete the input, sqls,and input bindings components. The output, query, and output bindings components are optional, depending on the requirement of the SQL service.

### Example

The following is a SQL service for cloning a product line, with a query to return the new product line.

#### Input

```json
{
    "newProductLine": "Electric Cars",
    "sourceProductLine": "Classic Cars"
}
```

The **input** is a simple or nested object providing SQL and query parameters. However, no array structure except array of values is allowed in the input object.

#### Output

```json
{
    "productLine": "Electric Cars",
    "description": "The new generation cars",
    "products": {
        "productCode": "N_S10_4757",
        "productName": "Porsche 356-A Roadster"
    }
}
```

The **output** of the SQL service is an object or array of object from the query, if it is specified. In this example, the output is the new product line with the list of the products for the product line.

#### Sqls

``` sql
insert into classicmodels.productlines (
    productLine, textDescription
)
select :newProductLine, textDescription
  from classicmodels.productlines
 where productLine = :sourceProductLine
;

insert into classicmodels.products (
    productLine, productCode, productName
)
select :newProductLine, concat('E-', productCode), productName
  from classicmodels.products
 where productLine = :sourceProductLine
```

The `sqls` component is a list of DML statements separated with a `";"` at the end of the last line of the SQL statement.  The SQL parameters may be a value or a value list. In this example, the `sqls` component includes two `INSERT ... SELECT` statements to copy the source product line and the products associated with the source product line. It is an efficient way to perform this task.

#### Query

```sql
select pl.productLine, pl.textDescription, p.productCode, p.productName
  from classicmodels.productlines pl, classicmodels.products p
 where p.productLine = pl.productLine
   and pl.productLine = :newProductLine
```

The **query** component of the SQL service is a single SELECT statement. The query parameters may be a value or a value list. In this example, the query returns the new product line along with its products.

#### Input Bindings

```json
[
    {
        "parameter": "sourceProductLine",
        "field": ".sourceProductLine"
    },
    {
        "parameter": "newProductLine",
        "field": ".newProductLine"
    }
]
```

The **input bindings** of the SQL service map the input fields to the sql and query parameters.

#### Output Bindings

```json
[
[
    {
        "field": ".productLine",
        "column": "productLine"
    },
    {
        "field": ".description",
        "column": "textDescription"
    },
    {
        "field": ".products..productCode",
        "column": "productCode"
    },
    {
        "field": ".products..productName",
        "column": "productName"
    }
]
```

The output bindings map the output fields to the query columns.

> The user is expected to generate the input and output bindings from the input, output, sqls and query with `Service Builder`, and then review and edit it as needed. The `Service Builder` is expected to generate the input and output bindings accurately in most cases, but the user is ultimately responsible for the accuracy of the input and output bindings.

### DDL and Substitution Variables

SQL service is mainly for data manipulation, but it does support DDL statement in the `sqls` component and substitution variables in the DDL statements. For example, we may have

``` sql
create table  test$testNo
```

in the `sqls` component, where $testNo is a substitution variable that maps to an input data field of the SQL service by name. 

By default, the `variableLength` property in the `service.json` file is set to zero and no substitution variable is allowed in the SQL statements. To enable substitution variables, the `variableLength` property needs to be set to a value greater than zero, based on the needs of the DDL statements. This property sets a limit on the number of characters that the substitution variables may assume, to minimize the risk of SQL injection. The maximum value allowed for the `variableLength` property is 100.

### Batch Input

SQL service supports batch input. That means that, assuming that we have a SQL service to add test score for a student as:

**Input**

```json
{
    "studentId": 1,
    "score": 89
}
```

**Sqls**

```
insert into test_score (student_id, score)
values (:studentId, :score) 
```

we may call this service with an array of input objects, as

```json
[
    {
        "studentId": 1,
        "score": 89
    },
    {
        "studentId": 2,
        "score": 99
    },
    ...
]
```

The SQL service will execute multiple times, once for each input object in the array, as one transaction. For SQL service with query, the query will be executed only once at the end of the service. This is something we need to consider when making service call with batch input.

## CRUD Service
---

The CRUD service is for CRUD operations of aggregate objects. Unlike the query and the SQL service, the CRUD service supports multiple operations, including

- read, for retrieving objects
- create, for creating object in database
- update, for updating object in database in its entirety
- delete, for deleting object from database
- save, for creating or updating object in database, and
- merge, for merging object changes into database

The CRUD service comprises:

- an object
- a read component, and
- a write component

The read component is a dynamic query for the object, comprising:

- an input for read
- a query for read
- an input bindings component that maps the query parameters to the input fields, and
- an output bindings component that maps the query columns to the objects fields

The output of the read operation is always an array of the CRUD objects by definition, even for read by id. In this case, it returns an array of a single object if the object exists, and an empty array if not.

The write components comprises:

- table bindings that map the root object and its child structures to the root and child database tables, respectively, and
- columns bindings that map the columns of a mapped table to the data fields of the CRUD object. 

### File Structure

The following is the file structure of CRUD service:

```bash
myCrudObject
    service.json
    object.json
    read/
        input.json
        query.sql
        input-bindings.json
        output-bindings.json
    write/
        tables.json
        my-root-table.table-alias.columns.json
        my-child-table.table-alias.columns.json
    tests/
        testReadMyCrudObject.json
        testCreateMyCrudObject.json
        testUpdateMyCrudObject.json
        testDeleteMyCrudObject.json
```

This file structure is generated when the service is created. TThere is a file for each component of the CRUD service. The user is expected to complete the object all read and write components.

### Example

Here is an example of completed CRUD service for an `Order` object.

#### Object

```json
{
    "orderNumber": 23,
    "orderDate": "2021-01-01T00:00:00.000Z",
    "customerNumber": 12,
    "customerName": "John",
    "orderLines": [{
        "orderLineNumber": 123,
        "productCode": "PC",
        "productName": "Computer",
        "quantityOrdered": 10,
        "priceEach": 599.00
    }]
}
```

The CRUD object is generally a complex object with nested structure.

#### Input

```json
{
    "orderNumber": 1,
    "customerNumber": 2,
    "startDate": "2021-01-01T00:00:00.000Z",
    "endDate": "2021-02-01T00:00:00.000Z"
}
```

The input is a simple object carrying the query parameters for read operation.

#### Query

```sql
select o.orderNumber, o.orderDate,
       _c.customerNumber, _c.customerName,
       od.orderLineNumber, _p.productCode, _p.productName, 
       od.quantityOrdered, od.priceEach
  from classicmodels.orders o
  join classicmodels.customers _c on o.customerNumber = _c.customerNumber
  left join classicmodels.orderdetails od on od.orderNumber = o.orderNumber
  left join classicmodels.products _p on _p.productCode = od.productCode
 where 1 = 1
   and o.orderNumber = :orderNumber
   and o.customerNumber = :customerNumber
   and o.orderDate between :startDate and :endDate
 order by o.orderNumber, od.orderLineNumber
```

This query is a single SELECT statement for read like the query service. However, it must follow the rules for CRUD query to be discussed below.

#### Input Bindings

```json
[
    {
        "parameter": "orderNumber",
        "field": ".orderNumber"
    },
    {
        "parameter": "customerNumber",
        "field": ".customerNumber"
    },
    {
        "parameter": "startDate",
        "field": ".startDate"
    },
    {
        "parameter": "endDate",
        "field": ".endDate"
    }
]
```

The input bindings map the query parameter to the read input field.

#### Output Bindings

```json
[
    {
        "column": "orderNumber",
        "field": ".orderNumber"
    },
    {
        "column": "orderDate",
        "field": ".orderDate"
    },
    {
        "column": "customerNumber",
        "field": ".customerNumber"
    },
    {
        "column": "customerName",
        "field": ".customerName"
    },
    {
        "column": "orderLineNumber",
        "field": ".orderLines..orderLineNumber"
    },
    {
        "column": "productCode",
        "field": ".orderLines..productCode"
    },
    {
        "column": "productName",
        "field": ".orderLines..productName"
    },
    {
        "column": "quantityOrdered",
        "field": ".orderLines..quantityOrdered"
    },
    {
        "column": "priceEach",
        "field": ".orderLines..priceEach"
    }
]
```

The output bindings map the query column to the object fields.

#### Table Bindings

```json
[
    {
        "name": "classicmodels.orders",
        "alias": "o",
        "object": ".",
        "rootTable": true,
        "mainTable": true,
        "operationIndicator": null,
        "columns": "./classicmodels.orders.columns.json"
    },
    {
        "name": "classicmodels.orderdetails",
        "alias": "od",
        "object": ".orderLines",
        "rootTable": false,
        "mainTable": false,
        "operationIndicator": null,
        "columns": "./classicmodels.orderdetails.columns.json"
    }
]
```

The following summarizes the properties of the table binding.

| Property | Description | Generated by Service Builder  | User Editable |
| --------- | ---------- | ---------- | ----- |
| name | name of table | Yes | No
| alias | alias of table | Yes | No
| object | path of root or child structure | Yes if match found | Yes
| root table | whether the root table | Yes | No |
| main table | whether a main table | Yes | Yes |
| operationIndicator | name of object field indicating merge operation | Yes | No |
| columns | location of column binding file | Yes |No |

The table bindings map the root object and its child structures to the database tables. In this example, the root object ```order``` is mapped to the ```classicmodels.orders``` table, and the child array ```orderLines``` is mapped to the ```classicmodels.orderdetails``` table. The table bindings specifies what tables shall be inserted with what objects/arrays, when the object is created. The order of inserts is dictated by the order of tables in the table bindings.  

#### Column Bindings

There is a column bindings component for each of the mapped tables, the `orders` and the `orderdetails` tables in this case. However, only an abbreviated version of the column bindings for the `orders` table is shown here for brevity. 

```json
[
    {
        "position": 1,
        "column": "orderNumber",
        "field": ".orderNumber",
        "key": true,
        "autoGenerate": false,
        "inputField": ".orderNumber",
        "insertValue": null,
        "updateValue": null,
        "version": false,
        "softDelete": false,
        "dataType": "number",
        "notNull": true,
        "keyEligible": false,
        "versionEligible": true,
        "softDeleteEligible": false
    },
    {
        "position": 2,
        "column": "orderDate",
        "field": ".orderDate",
        "key": false,
        "autoGenerate": false,
        "inputField": null,
        "insertValue": null,
        "updateValue": null,
        "version": false,
        "softDelete": false,
        "dataType": "datetime",
        "notNull": true,
        "keyEligible": false,
        "versionEligible": false,
        "softDeleteEligible": false
    },
    ...
]
```

The following summarizes the properties of the column binding.

| Property | Description | Generated by Service Builder  | User Editable |
| --------- | ---------- | ---------- | ----- |
| column | name of column | Yes | No
| field | path of field mapped to the column | Yes, if match found | Yes
| key | whether a key column | Yes, if primary key defined for table | Yes
| autoGenerate | whether an auto generate column | Yes | No
| inputField | path of input field mapped to the key column. Specify if you want the write operations to read back updated objects | No | Yes
| insertValue | insert value for the column | No | Yes |
| updateValue | default update value for the column | No | Yes |
| version | wether a version control column | No | Yes |
| softDelete | wether a soft delete column | No | Yes |
| Meta Properties: |
| position | column position in the table | Yes | No |
| dataType | target json data type | Yes | No |
| notNull | whether null value allowed | Yes | No |
| keyEligible | whether eligible as a key column | Yes | No |
| versionEligible | whether eligible as a version column | Yes | No |
| softDeleteEligible | whether eligible as a soft delete column | Yes | No |

The columns bindings map the object field to the columns of a table. It specifies how a row should be inserted and updated.

When insert, the value of a column is from:

- database if an auto-generate column
- the insertValue if specified. The insertValue should be a DB expression
- the object field if mapped and the field exists in the input
- otherwise, the column is excluded from the insert statement if none of the above is true.

When update, the value of a column is from:

- the updateValue if specified, the updateValue should be an DB expression
- the object field if mapped and the field exists in the input
- otherwise, the column is excluded from the update statement if none of the above is true.

Both update and delete are by the key. The key is always the primary key of the table. However, if in the rare case where no primary key is defined, the user needs to select one or more columns as the key by setting their ```key``` properties to true.

The CRUD service supports optimistic locking with a version column. The version column is designated by setting its ```version``` property to true. The version column must be a number column. The ```versionEligible``` property indicates if the column is a fit for version control.

The CRUD service supports soft delete. The soft-delete column is designated by setting its ```softDelete``` property to true. The soft-delete column must be a character column. CRUD service updates this column to `'Y'` when delete occurs.

### Read-Write Asymmetry

The read and write operations are generally asymmetric, meaning that the read operation generally reads from more tables than the write operations write into.  

For example, in the order example above, the read operation read from four tables: ```orders```, ```orderdetails```, ```customers``` and ```products```, but the write operations only write into the ```orders``` and ```orderdetails``` table.  

The ```customers``` and ```products``` tables in this case are read-only, and are called reference table, as they are referenced by other tables.  

### Root, Child and Reference Table

The root table stores the root object. The child tables store the data that depends on the root object. The reference tables are read-only, with a different life-cycle than the root object.  Table bindings are not generated for the reference tables.

In the above example, ```orders``` is the root table, ```orderdetails``` is a child table, and ```customers``` and ```products``` are reference tables.

### Main Table

The root table is usually the main table, or the top parent table for the aggregate object, referenced by the child tables. However, there are cases where a child table is referenced by the root table. For example, we may have a root table `customers` referencing the child table `addresses`. When creating the following `Customer` object:

```json
{
    "customerNumber": 123,
    "customerName": "John",
    "address": {
        "addressId": 123,
        "street": "123 Main st",
        "city": "Los Angeles"
    }
}
```

We need to config the `addresses` table to be the main table, so that a record is inserted into this table before the record for the root table.

### CRUD Query

The query for CRUD read operation follows a few rules:

- the root table shall be the only table in the FROM clause, like ```classicmodels.orders``` in the example above;
- the child tables shall be added using ANSI left join, like ```classicmodels.orderDetails``` in the example above, in the order that the records are to be inserted when creating the object
- the reference tables shall be added using ANSI inner or left join as appropriate, like ```classicmodels.customers``` and ```classicmodels.products``` in the example above;
- all table alias should be aliased, like ```o``` for the ```classicmodels.orders``` table in the example above;
- the alias for the reference table should be prefixed with "_", like ```_p``` for the ```classicmodels.products``` table in the example above;
- all table columns from the root and child tables should be prefixed with the respective table alias;
- the WHERE clause may need to include ```1 = 1``` as a base condition to support `readAll` operation, as the query for read is dynamic by definition.

> Database views can only be used as reference tables in CRUD query.

### CRUD Operations

The CRUD service is a multi-operation service. The chart below lists the supported operations, and the input and output for each operation:

| Operation | Input | Output |
| --------- | ----- | ------ |
| read  | input | array of objects  |  
| create  | object or objects | created object or objects |  
| delete  | object or objects | none  |
| update  | object or objects | updated object or objects  |
| save  | object or objects | saved object or objects  |
| merge  | object or objects | updated object or objects  |

For `write` operations, the objects can be batched. For `delete` operation, the object only needs contains the key fields of the root object. The write operation actually returns null by default, unless the `inputField` property is specified for the key columns in the column bindings.

The `inputField` is the field path to the key field in read input. The CRUD service retrieves the update object from the database using the `readByKey` query, which is assumed to be supported by the read component of the service.

#### Read

The read operation is a query for the aggregate object. The input is an instance of the input object; the output is an array of the objects.  

The query is dynamic. In the example above, if the input is

``` json
{}
```

it is a query for all objects;  if the input is

``` json
{ "orderNumber": 123 }
```

it a query by order number; and if the input is

``` json
{ 
    "customerNumber": 123, 
    "startDate": "2021-01-01T00:00:00.000Z",   
    "endDate": "2021-02-01T00:00:00.000Z"
    }
```

it is a query by customer number and date range.

#### Create

The create operation creates and instance of the aggregate object, including the root object and its children, in the database. The input is the object to be created, and the output is the object created. However, the input may also be an array of the objects, and the output will be an array of the objects created in this case.

#### Delete

The delete operation deletes the aggregate, including the root object and its children, from the database. The input is the object or the array of objects to be deleted, the output is none. The delete is by the key of the root object/table, and it is sufficient to include only the key fields of the object in the input of service request.

#### Update

The update operation updates the aggregate, including the root object and its children, in the database. The input is the object to be updated, and the output is the object updated. However, the input may also be an array of the objects, and the output will be an array of the objects updated in this case.

The update is done by the key of the root object/table. The object that is not found and thus not updated will be returned as ```{}``` in the output.

In this operation, the old version of the object will be replaced by the given version in its entirety. As part of this process, the child objects of the aggregate may be created, updated or deleted from the database, to sync the stored object with the given object.

#### Save

The save operation is a create operation if the object does not yet exist or an update operation if the object already exists in the database.

#### Merge

The merge operation merges the changes to an aggregate into the current version in the database. The input is the object that carries the changes, and the output is the merged object. The input may also be an array of objects, and the output will be an array of merged objects in this case.

The input object should only include the fields that are changed. For arrays, an operation indicator fields should be included with the element to indicate the type of operation to be performed for the element. The valid values include: ```create```, ```update``` or ```delete```.

The name of operation indicator field is specified in the table bindings. The operation indicator field itself will not be persisted. Its sole purpose is to indicate the type of change.
