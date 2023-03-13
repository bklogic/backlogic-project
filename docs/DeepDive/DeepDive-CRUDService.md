# CRUD Service Deep Dive

In this article, we do a deep dive on the CRUD service. We will start with an overview of CRUD service, followed by an examination of the various components of CRUD service. We will then present a set of simple and complex examples of CRUD service.

We assume that you have had some hands-on experience with Service Builder, the development tool for data access service, and with the data access service. If not, please take a look at these tutorial first:

[Get Started with Service Builder](/docs/GetStarted/getStartedWithServiceBuilder.md)  
[Get Started with Data Access Service](/docs/GetStarted/getStartedWithDataAccessService.md)


## Overview
---

### Use

CRUD service is a repository for CRUD operations of aggregate object. It aims to provide the developer the flexibility to create efficient repository for any DDD aggregate that is needed by the application.

### Components

Conceptually, the CRUD service is composed of the following components:

- object
- read
  - input
  - query
  - input bindings
  - output bindings
- write
  - table bindings
  - column bindings

The object component is a JSON object specifying the shape and fields of the aggregate object.

The read component specifies a dynamic query service to retrieve the aggregate objects. Its output is always an array of the aggregate objects.

The write component comprises table bindings and column bindings. The table bindings specifies the tables to write and maps the tables to the root and child structures of the aggregate object. The column bindings map the columns of each table to the data fields of the object or structure.

The read and write components are specified separately, and thus changing a potential bi-directional mapping into two uni-direction mappings. This simplifies the difficult object-relational mapping problem, and supports asymmetric read-write as we will see in the example below.

### Operations

Unlike the single-operation SQL and query services, the CRUD service supports a number of read and write operations on the aggregate object, including
- read, for retrieving object
- create, for creating object
- update, for updating object in their entirety, equivalent to replacement
- delete, for removing object
- save, for updating or creating (if not exist) object, and
- merge, for merging changes into object

### Development

Service Builder provides a streamlined process for developing the CRUD service in a few simple steps:
- create the service
- specify the object in JSON
- develop the SQL query for read
- generate, review and edit, if needed, the input and output bindings for read, and
- generate, review and edit, if needed, the table and column bindings for write
- generate, edit and run the tests for the service

For `read`, the focus is on developing the SQL query to return the data set for populating the object. For `write`, the focus is on the column bindings to add record(s) for each table. As the SQL query for read is also responsible for providing information needed to generate the table and column bindings, it must follow a number of rules for CRUD query. 

As such, it is important to create a SQL development environment in VS Code with the various database extensions available, so that we have the visibility of database in VS Code and have the help of SQL syntax linting, SQL intellisense, SQL test, etc. Otherwise, we will need to compose and test the SQL in a specialized SQL editor, such as Oracle SQL developer, and then copy and paste them into the `query.sql` file.

As soon as the SQL service is completed, we should try to deploy it into the remote workspace, which helps validate the service. We can then fix the issue, if any, and repeat the process until it is clean. Thereafter, we can proceed to adding the test.

For CRUD service, at minimum, we will need one test for each operation. For read, we are expected to add one test for each read scenarios. For example, we may have one test for `readAll` and one test for `readById`.

The test for write operations may be run with or without a commit. A rollback will be performed at the end of the test if it is run without commit.


### Deployment

The CRUD service is deployed on the data access server as a set of HTTP APIs, one for each operation. At development time, it is deployed into the remote workspace as it is developed. At runtime, it is deployed as part of the data access application.

### Service Calls

Since the CRUD service is deployed as a set of HTTP APIs, it may be invoked by a HTTP request for read or write, like

```http
POST: https://{baseUrl}/application/module/service/operation
Content Type: application/json

{
    "json data": 123
}
```

The HTTP method is always `POST`. The `baseUrl` depends on the workspace or runtime instance. The operation is `read`, `create`, `update`, `delete`, `save` or `merge`. The body of the request is the input for read or the object for write. The `delete` operation only needs to include the key field(s) in the object. The input for the write operations may be an array of the objects. In this case, the objects will be processed in a single transaction.

## Anatomy
---

Lets take a look at the CRUD service through an example: `Order`. The service is to provide CRUD operations for the `Order` object.

### Files

Upon creation, we will get a set of service files as below:

```sh
- Order
    - service.json
    - object.json
    - read
        - input.json
        - query.sql
        - input-bindings.json
        - output-bindings.json
    - write
        - tables.json
        - {schema}.{table}.{alias}.columns.json
```

The `service.json` files is a service descriptor generated by Service Builder and is not supposed to be modified. The rest of files are the various components of CRUD service and are what are to be developed by the developer.

### Object

The object define what the target of the CRUD service, and the problem for the developer to solve.

For our example, we have

``` json
{
    "orderNumber": 1,
    "orderDate": "2020-12-10T00:00:00.000",
    "customerNumber": 103,
    "customerName": "John",
    "requiredDate": "2020-12-10T00:00:00.000",
    "shippedDate": "2020-12-10T00:00:00.000",
    "status": "shipped",
    "comments": "shipped on time",
    "total": "345.60",
    "lines": [{
        "orderLineNumber": 10,
        "productCode": "S12_1099",
        "productName": "1968 Ford Mustang",
        "qty": 2,
        "price": 35.45,
        "subtotal": 70.90
    }]
}
```

### Read

#### Input

The input defines the parameters that are to be used for read. As `read` is a dynamic query, not all parameters defined in the input need to be supplied in the service request, depending on the read request. 

For our example, we have:

```json
{
    "orderNumber": 1,
    "customerNumber": 3,
    "startDate": "2020-12-01T00:00:00.000",
    "endDate": "2020-12-31T00:00:00.000"
}
```

It is to support `readByOrderNumber`, `readByCustomerNumber` and `readByOrderDateRange`. Note that the `startDate` and `endDate` are not from the `Order` object. The separate `input` for `read` operation provides a great amount of flexibility.

#### Query

This SQL query is responsible for retrieving the data set for populating all data fields of the object. It is the developer's answer to the CRUD read problem.

For our example, we have

```sql
select o.orderNumber, o.orderDate,
       o.customerNumber, _c.customerName,
       o.requiredDate, o.shippedDate, o.status, o.comments,
       od.orderLineNumber, od.productCode, _p.productName,
       od.quantityOrdered as qty, od.priceEach, od.quantityOrdered*od.priceEach as subtotal,
       (select sum(quantityOrdered*priceEach) from orderdetails 
         where orderNumber = o.orderNumber) as total
  from orders o
  join customers _c on o.customerNumber = _c.customerNumber
  left join orderdetails od on od.orderNumber = o.orderNumber
  left join  products _p on _p.productCode = od.productCode
 where ( 0 = 1
         or :orderNumber is not null
         or :customerNumber is not null
         or :startDate is not null 
         or :endDate is not null
       )
   and o.orderNumber = :orderNumber
   and o.customerNumber = :customerNumber
   and o.orderDate between :startDate and :endDate
```

This read query is dynamic by design. Therefore, if we invoke the read operation by passing an `orderNumber`, we will get the order identified by the orderNumber; if we the read operation by passing a `customerNumber`, we will get the list of orders associated with the customer; if we invoke the read operation without passing any parameter, we will get empty list from this query. The `WHERE` clause of the read query shall be formulated to support all query needs for the object.

From this query, we see that the data for populating the `Order` object is read not only from the `orders` and `order_lines` tables, but also from the `customers` and `products` tables. However, when writing the `Order` object back to the database, we will only write the data into the `orders` and `order_lines` tables. The `total` and `subtotal` fields within the `Order` object are also read-only fields summed up from the order details, and are not to be written back to the database. This is the so-called read-write asymmetry, and is supported by the CRUD service by design.

The `customers` and `products` tables here are called reference tables, as data is only read from them. From DDD (domain driven design) perspective, they are not part of the domain of concern. The `orders` table is called root table, as it is mapped to the root object. The `order_lines` table is called child table as it is mapped to a child structure of the root object. The write operations write data to the root and child tables.

In order for Service Builder to be able to generate proper table and column bindings from the read components, there are a few rules that the CRUD query must follow:
- The query must be in ANSI format, with the root table appearing in the `FROM` clause and the child and reference tables in the `JOIN` clauses;
- All tables must be aliased and all columns must be prefixed with the table alias, like `o.orderNumber`;
- The alias for the reference tables should start with an underscore, like `_c` and `_p` in above query for the `customers` and `products` tables, respectively.
- The child tables must be joined in the order that the records are to be inserted when creating the object.

#### Input and Output Bindings

The input bindings indicate the source of data for the query parameters. The output bindings provide the critical information to construct the aggregate object(s) from the relational data set returned by the query.

For our example, we have

*Input bindings:*

|  | parameter | field |
| - | - | - |
| 0 | orderNumber | .orderNumber |
| 1 | endDate | .endDate |
| 2 | customerNumber | .customerNumber |
| 3 | startDate | .startDate |


*Output Bindings:*

| | field | column |
| - | - | - |
| 0 | .orderNumber | orderNumber |
| 1 | .orderDate | orderDate |
| 2 | .customerNumber | customerNumber |
| 3 | .customerName | customerName |
| 4 | .requiredDate | requiredDate |
| 5 | .shippedDate | shippedDate |
| 6 | .status | status |
| 7 | .comments | comments |
| 8 | .total | total |
| 9 | .lines..orderLineNumber | orderLineNumber |
| 10 | .lines..productCode | productCode |
| 11 | .lines..productName | productName |
| 12 | .lines..qty | qty |
| 13 | .lines..price | priceEach |
| 14 | .lines..subtotal | subtotal |

Like the query service, the input and output bindings are not expected to be hand-coded from scratch, but generated with Service Builder at first, and then reviewed and edited, if necessary, by the developer.

### Write

#### Table Bindings

The table bindings specify the tables to write and map them to the root and child structures of the aggregate object.

For our example, we have

```json
[
    {
        "name": "orders",
        "alias": "o",
        "object": ".",
        "rootTable": true,
        "mainTable": true,
        "operationIndicator": null,
        "columns": "./orders.o.columns.json"
    },
    {
        "name": "orderdetails",
        "alias": "od",
        "object": ".lines..",
        "rootTable": false,
        "mainTable": true,
        "operationIndicator": null,
        "columns": "./orderdetails.od.columns.json"
    }
]
```

There are two entries, for the root `orders` table and the child `orderDetails` table, respectively. The table bindings component is generated from the read query, and is not to be edited except for the `mainTable` and `operationIndicator` attributes that we do not discuss here.

#### Column Bindings

There is one column bindings component for each mapped table. The column bindings specify how the record should be inserted, updated and deleted for the relevant write operations.

For the `orders` table, we have

| position | column | field | key | autoGenerate | inputField | insertValue | updateValue | version | softDelete | dataType | notNull | keyEligible | versionEligible | softDeleteEligible | 
| - | - | - | - | - | - | - | - | - | - | - | - | - | - | - |
| 1 | orderNumber | .orderNumber | true | false | .orderNumber |  |  | false | false | number | true | true | true | false | 
| 2 | orderDate | .orderDate | false | false |  | CURRENT_TIMESTAMP |  | false | false | datetime | true | false | false | false | 
| 3 | requiredDate | .requiredDate | false | false |  |  |  | false | false | datetime | true | false | false | false | 
| 4 | shippedDate | .shippedDate | false | false |  |  |  | false | false | datetime | false | false | false | false | 
| 5 | status | .status | false | false |  |  |  | false | false | string | true | false | false | true | 
| 6 | comments | .comments | false | false |  | 'no comments' |  | false | false | string | false | false | false | false | 
| 7 | customerNumber | .customerNumber | false | false |  |  |  | false | false | number | true | false | true | false | 

Here is a brief explanation of the various attributes in the column bindings:
- column position in the table
- column: the column name
- field: the data field mapped to the column
- key: whether the column is part of the key
- autoGenerate: whether an auto generated column
- input field: path to input data field, controls whether to return updated object
- insertValue: SQL expression to generate value for insert
- updateValue: SQL expression to generate value for update
- version: whether a version control column
- softDelete: whether a column for soft delete
- dataType: generic data type of the column, for reference
- notNull: whether a not null column, for reference
- keyEligible: whether eligible to be a key column, for reference
- versionEligible: whether eligible to be a version control column, for reference
- softDeleteEligible: whether whether eligible to be a column for soft delete, for reference

These attributes are extracted from the table meta data. The attribute we may edit include:

- key: if primary key is defined for the table
- input field: if not populated bu we want to return updated object for write operations
- inputValue: if we want to use this value for insert, regardless whether the column is mapped to an object field
- updateValue if you want to use this value for update, regardless whether the column is mapped to an object field
- version: to true, if we want to enable version control on the table
- softDelete: to true, if we want to enable soft delete for the table

For INSERT, `autoGenerate` takes precedence over `insert value`; `insert value` takes precedence over mapped `field`. If neither is available, the column will be excluded from the INSERT statement, and the value will be `null` or the database default. For UPDATE, `update value` takes precedence over mapped `field`. If neither is available, the column will be excluded from the UPDATE statement, and the column will not be updated.

The `insert value` and `update value` shall be a SQL expression, like the `insert value`: `CURRENT_TIMESTAMP` and `'no comments'` for columns `orderDate` and `comments`, respectively, in our example. 

## Examples
---

The examples in this tutorial demonstrates what we can do with CRUD and how we could do it. The sample MySQL database used for the examples is the [`classicmodels`](https://www.mysqltutorial.org/mysql-sample-database.aspx). The source code for these example CRUD services can be found [here](https://github.com/bklogic/data-access-service-example).

For the seek of brevity, the input and output bindings are omitted from presentations of all examples.

### Example #1 - Office

#### Problem

Repository for `Office`. The simplest CRUD service for a simple object mapped to a single table. It may be directly generated from the table.

##### Object

```json
{
    "officeCode": "10",
    "address": "100 Market Street",
    "phone": "+1 650 219 4782",
    "city": "San Francisco",
    "state": "CA",
    "country": "USA",
    "postalCode": "94080",
    "territory": "NA"
}
```

#### Read

##### Input

```json
{
    "officeCode": "10",
    "country": "USA"
}
```

##### Query

```sql
SELECT o.officeCode, o.addressLine1 as address, o.phone,
       o.city, o.state, o.country, o.postalCode, o.territory
  FROM offices o
 WHERE 1 = 1
   AND o.officeCode = :officeCode
   AND o.country = :country
```

This service supports `readByOfficeCode`, `readByCountry` and `readAllOffices`. 

#### Write

##### Table Bindings

```json
[
    {
        "name": "offices",
        "alias": "o",
        "object": ".",
        "rootTable": true,
        "mainTable": true,
        "columns": "./offices.columns.json"
    }
]
```

##### Column Bindings

| position | column | field | key | autoGenerate | inputField | insertValue | updateValue | version | softDelete | dataType | notNull | keyEligible | versionEligible | softDeleteEligible |
| - | - | - | - | - | - | - | - | - | - | - | - | - | - | - |
| 1 | officeCode | .officeCode | true | false | .officeCode |  |  | false | false | string | true | true | false | true |
| 2 | city | .city | false | false |  |  |  | false | false | string | true | false | false | true |
| 3 | phone | .phone | false | false |  |  |  | false | false | string | true | false | false | true |
| 4 | addressLine1 | .address | false | false |  |  |  | false | false | string | true | false | false | true |
| 5 | addressLine2 |  | false | false |  |  |  | false | false | string | false | false | false | true |
| 6 | state | .state | false | false |  |  |  | false | false | string | false | false | false | true |
| 7 | country | .country | false | false |  |  |  | false | false | string | true | false | false | true |
| 8 | postalCode | .postalCode | false | false |  |  |  | false | false | string | true | false | false | true |
| 9 | territory | .territory | false | false |  |  |  | false | false | string | true | false | false | true |

Note that the `addressLine2` column is not mapped to any data field.

### Example #2 - Customer

#### Problem

Repository for `Customer`. A CRUD service for a structured object mapped to a single table.

##### Object

```json
{
    "customerNumber": 1,
    "customerName": "Land of Toys Inc.",
    "contact": {
        "firstName": "Joe",
        "LastName": "Williamson",
        "phone": "(171) 555-2282"
    },
    "address": {
        "addressLine1": "5557 North Pendale Street",
        "addressLine2": "",
        "city": "NYC",
        "state": "NY",
        "country": "USA"
    }
}
```

#### Read

##### Input

```json
{
    "customerNumber": 1,
    "city": "Los Angeles",
    "postalCode": "83030"
}
```

##### Query

```sql
select c.customerNumber, c.customerName, 
       c.contactLastName, c.contactFirstName, c.phone,
       c.addressLine1, c.addressLine2, c.city, c.state, 
       c.country , c.postalCode
  from customers c
 where 1 = 1
   and c.customerNumber = :customerNumber
   and c.city = :city
   and c.postalCode = :postalCode
```

This service supports `readByCustomerNumber`, `readByCity`, `readByPostalCode` and `readAllCustomers`.

#### Write

##### Table Bindings

```json
[
    {
        "name": "customers",
        "alias": "c",
        "object": ".",
        "rootTable": true,
        "mainTable": true,
        "operationIndicator": null,
        "columns": "./customers.c.columns.json"
    }
]
```

##### Column Bindings

| position |column |field |key |autoGenerate |inputField |insertValue |updateValue |version |softDelete |dataType |notNull |keyEligible |versionEligible |softDeleteEligible |
| - | - | - | - | - | - | - | - | - | - | - | - | - | - | - |
| 1 |customerNumber |.customerNumber |true |false |.customerNumber | | |false |false |number |true |true |true |false |
| 2 |customerName |.customerName |false |false | | | |false |false |string |true |false |false |true |
| 3 |contactLastName |.contact.LastName |false |false | | | |false |false |string |true |false |false |true |
| 4 |contactFirstName |.contact.firstName |false |false | | | |false |false |string |true |false |false |true |
| 5 |phone |.contact.phone |false |false | | | |false |false |string |true |false |false |true |
| 6 |addressLine1 |.address.addressLine1 |false |false | | | |false |false |string |true |false |false |true |
| 7 |addressLine2 |.address.addressLine2 |false |false | | | |false |false |string |false |false |false |true |
| 8 |city |.address.city |false |false | | | |false |false |string |true |false |false |true |
| 9 |state |.address.state |false |false | | | |false |false |string |false |false |false |true |
| 10 |postalCode | |false |false | | | |false |false |string |false |false |false |true |
| 11 |country |.address.country |false |false | | | |false |false |string |true |false |false |true |
| 12 |salesRepEmployeeNumber | |false |false | | | |false |false |number |false |false |true |false |
| 13 |creditLimit | |false |false | | | |false |false |number |false |false |false |false |


### Example #3 - ProductLine

#### Problem

Repository for maintaining product line. A CRUD service for object with array structure.

##### Object

```json
{
    "productLine": "Classic Cars",
    "description": "Land of Toys Inc.",
    "image": "base64YmFzZTY0ZW5jb2RlZA==",
    "products": [{
        "productCode": "S12_1099",
        "productName": "1968 Ford Mustang",
        "productVendor": "Motor City Art Classics",
        "productDescription": "Hood, doors and trunk all open.",
        "productScale": "1:100",
        "quantityInStock": 100,
        "buyPrice": "15.45",
        "MSRP": "46.00"
    }]
}
```

#### Read

##### Input

```json
{
    "productLine": "Classic Cars",
    "productName": "1968 Ford Mustang"
}
```

##### Query

```sql
select pl.productLine, pl.textDescription, pl.image,
       p.productCode, p.productName, p.productVendor, p.productDescription,
       p.productScale, p.quantityInStock, p.buyPrice, p.MSRP
  from productlines pl
  left join products p on p.productLine = pl.productLine
 where 1 = 1
   and pl.productLine = :productLine
   and pl.productLine in (select productLine from products where productName = :productName)
```

This service supports `readAllProductLines`,  `readByProductLineName`,  and `readProductLinesWithProductName` that returns product lines including product with a given name. From this example, it is seen that we have a fair amount of flexibility in formulating the `WHERE` clause of the query, in order to satisfy the various query needs for the object.

#### Write

##### Table Bindings

```json
[
    {
        "name": "productlines",
        "alias": "pl",
        "object": ".",
        "rootTable": true,
        "mainTable": true,
        "operationIndicator": null,
        "columns": "./productlines.pl.columns.json"
    },
    {
        "name": "products",
        "alias": "p",
        "object": ".products..",
        "rootTable": false,
        "mainTable": true,
        "operationIndicator": null,
        "columns": "./products.p.columns.json"
    }
]
```

***Column Bindings - productlines:***

| position |column |field |key |autoGenerate |inputField |insertValue |updateValue |version |softDelete |dataType |notNull |keyEligible |versionEligible |softDeleteEligible |
| - | - | - | - | - | - | - | - | - | - | - | - | - | - | - |
| 1 |productLine |.productLine |true |false |.productLine | | |false |false |string |true |true |false |true |
| 2 |textDescription |.description |false |false | | | |false |false |string |false |false |false |true |
| 3 |htmlDescription | |false |false | | | |false |false |string |false |false |false |false |
| 4 |image |.image |false |false | | | |false |false |binary |false |false |false |false |

***Column Bindings - products:***

| position |column |field |key |autoGenerate |inputField |insertValue |updateValue |version |softDelete |dataType |notNull |keyEligible |versionEligible |softDeleteEligible |
| - | - | - | - | - | - | - | - | - | - | - | - | - | - | - |
| 1 |productCode |.products..productCode |true |false |.products..productCode | | |false |false |string |true |true |false |true |
| 2 |productName |.products..productName |false |false | | | |false |false |string |true |false |false |true |
| 3 |productLine |.productLine |false |false | | | |false |false |string |true |false |false |true |
| 4 |productScale |.products..productScale |false |false | | | |false |false |string |true |false |false |true |
| 5 |productVendor |.products..productVendor |false |false | | | |false |false |string |true |false |false |true |
| 6 |productDescription |.products..productDescription |false |false | | | |false |false |string |true |false |false |false |
| 7 |quantityInStock |.products..quantityInStock |false |false | | | |false |false |number |true |false |true |false |
| 8 |buyPrice |.products..buyPrice |false |false | | | |false |false |number |true |false |false |false |
| 9 |MSRP |.products..MSRP |false |false | | | |false |false |number |true |false |false |false |


### Example #4 - Order

#### Problem

Repository for `Order`. A CRUD service for object with array structure, featuring read-write asymmetry. This is the example used in the Anatomy section and is relisted here.

##### Object

``` json
{
    "orderNumber": 1,
    "orderDate": "2020-12-10T00:00:00.000",
    "customerNumber": 103,
    "customerName": "John",
    "requiredDate": "2020-12-10T00:00:00.000",
    "shippedDate": "2020-12-10T00:00:00.000",
    "status": "shipped",
    "comments": "shipped on time",
    "total": "345.60",
    "lines": [{
        "orderLineNumber": 10,
        "productCode": "S12_1099",
        "productName": "1968 Ford Mustang",
        "qty": 2,
        "price": 35.45,
        "subtotal": 70.90
    }]
}
```

#### Read

##### Input

```json
{
    "orderNumber": 1,
    "customerNumber": 103,
    "startDate": "2004-12-01T00:00:00.000",
    "endDate": "2004-12-31T00:00:00.000"
}
```

##### Query

```sql
select o.orderNumber, o.orderDate,
       o.customerNumber, _c.customerName,
       o.requiredDate, o.shippedDate, o.status, o.comments,
       od.orderLineNumber, od.productCode, _p.productName,
       od.quantityOrdered as qty, od.priceEach, od.quantityOrdered*od.priceEach as subtotal,
       (select sum(quantityOrdered*priceEach) from orderdetails 
         where orderNumber = o.orderNumber) as total
  from orders o
  join customers _c on o.customerNumber = _c.customerNumber
  left join orderdetails od on od.orderNumber = o.orderNumber
  left join  products _p on _p.productCode = od.productCode
 where ( 0 = 1
         or :orderNumber is not null
         or :customerNumber is not null
         or :startDate is not null 
         or :endDate is not null
       )
   and o.orderNumber = :orderNumber
   and o.customerNumber = :customerNumber
   and o.orderDate between :startDate and :endDate
```

This service supports `readByOrderNumber`, `readByCustomerNumber` and `readByOrderDateRange`, but does not support `readAllOrders`. Point worth noting include:
- The `total` and `subtotal` column are calculated fields for read purpose only
- The query parameters `startDate` and `endDate` are not mapped to any table column.

#### Write

```json
[
    {
        "name": "orders",
        "alias": "o",
        "object": ".",
        "rootTable": true,
        "mainTable": true,
        "operationIndicator": null,
        "columns": "./orders.o.columns.json"
    },
    {
        "name": "orderdetails",
        "alias": "od",
        "object": ".lines..",
        "rootTable": false,
        "mainTable": true,
        "operationIndicator": null,
        "columns": "./orderdetails.od.columns.json"
    }
]
```

*Column Bindings - orders:*

| position | column | field | key | autoGenerate | inputField | insertValue | updateValue | version | softDelete | dataType | notNull | keyEligible | versionEligible | softDeleteEligible | 
| - | - | - | - | - | - | - | - | - | - | - | - | - | - | - |
| 1 | orderNumber | .orderNumber | true | false | .orderNumber |  |  | false | false | number | true | true | true | false | 
| 2 | orderDate | .orderDate | false | false |  | CURRENT_TIMESTAMP |  | false | false | datetime | true | false | false | false | 
| 3 | requiredDate | .requiredDate | false | false |  |  |  | false | false | datetime | true | false | false | false | 
| 4 | shippedDate | .shippedDate | false | false |  |  |  | false | false | datetime | false | false | false | false | 
| 5 | status | .status | false | false |  |  |  | false | false | string | true | false | false | true | 
| 6 | comments | .comments | false | false |  | 'no comments' |  | false | false | string | false | false | false | false | 
| 7 | customerNumber | .customerNumber | false | false |  |  |  | false | false | number | true | false | true | false | 

*Column Bindings - orderdetails:*

| position |column |field |key |autoGenerate |inputField |insertValue |updateValue |version |softDelete |dataType |notNull |keyEligible |versionEligible |softDeleteEligible |
| - | - | - | - | - | - | - | - | - | - | - | - | - | - | - |
| 1 |orderNumber |.orderNumber |true |false | | | |false |false |number |true |true |true |false |
| 2 |productCode |.lines..productCode |true |false | | | |false |false |string |true |true |false |true |
| 3 |quantityOrdered |.lines..qty |false |false | | | |false |false |number |true |false |true |false |
| 4 |priceEach |.lines..price |false |false | | | |false |false |number |true |false |false |false |
| 5 |orderLineNumber |.lines..orderLineNumber |false |false | | | |false |false |number |true |false |true |false |

Note that the `orderdetails` table has a composite key in `orderNumber` and `productCode`.

### Example #5 - OfficeAggregate

#### Problem

Repository for OfficeAggregate. A CRUD service for a complex object.

##### Object

```json
{
    "officeCode": "100",
    "name": "San Francisco",
    "address": {
        "city": "San Francisco",
        "phone": "312-243-0567",
        "address": "123 Main st.",
        "state": "CA",
        "country": "USA",
        "postalCode": "81000"    
    },
    "manager": {
        "employeeNumber": 1,
        "firstName": "John",
        "lastName": "Smith",
        "jobTitle": "Office Manager",
        "extension": "1234",
        "email": ""
    },
    "employees": [{
        "employeeNumber": 2,
        "firstName": "John",
        "lastName": "Smith",
        "jobTitle": "Developer",
        "extension": "1234",
        "email": "",
        "reportsTo": 1
    }]
}
```

#### Read

##### Input

```json
{
    "officeCode": "100"
}
```

##### Query

```sql
select e.employeeNumber as eEmployeeNumber, e.lastName as eLastName, 
       e.firstName as eFirstName, e.jobTitle as eJobTitle, o.officeCode, o.city, o.phone, o.addressLine1, 
       o.state, o.country, o.postalCode,
       e.employeeNumber as eEmployeeNumber, e.lastName as eLastName, 
       e.firstName as eFirstName, e.jobTitle as eJobTitle,
       e.extension as eExtension, e.email as eEmail, e.reportsTo,
       m.employeeNumber as mEmployeeNumber, m.lastName as mLastName, 
       m.firstName as mFirstName, m.jobTitle as mJobTitle,
       m.extension as mExtension, m.email as mEmail
  from offices o
  left join employees m on m.officeCode = o.officeCode and m.jobTitle = 'office manager'
  left join employees e on e.officeCode = o.officeCode and e.jobTitle != 'office manager'
 where 1 = 1
   and o.officeCode = :officeCode
```

##### Output Bindings

| field | column |
| - | - |
| .officeCode | officeCode |
| .name | city |
| .address.city | city |
| .address.state | state |
| ... |

Point worth noting:
- The `name` of office does not exist in the table or SQL query. We have it mapped to the `city` column to have `city` as the office name;
- Column `city` is also mapped to the `city` field within `OfficeAggregate`.

#### Write

##### Table Bindings

| name |alias |object |rootTable |mainTable |operationIndicator |columns |
| - | - | - | - | - | - | - |
| offices |o |. |true |true | |./offices.columns.json |
| employees |m |.manager. |false |false | |./employees.columns.json |
| employees |e |.employees.. |false |true | |./employees.columns.json |

Point worth noting:
- The `employees` table appeared twice in the table bindings, one for the `"manager"` record and one for the `"employees"` records. The two are differentiated by table aliases;
- When the `Office` object is created in the database, there will be one record inserted into the `offices` table, one record inserted into the `employees` table for `"manager"`, and multiple records inserted into the `employees` table for `"employees"`.

##### Column Bindings

***Office:***

| position |column |field |key |autoGenerate |inputField |insertValue |updateValue |version |softDelete |dataType |notNull |keyEligible |versionEligible |softDeleteEligible |
| - | - | - | - | - | - | - | - | - | - | - | - | - | - | - |
| 1 |officeCode |.officeCode |true |false |.officeCode | | |false |false |string |true |true |false |true |
| 2 |city |.address.city |false |false | | | |false |false |string |true |false |false |true |
| 3 |phone |.address.phone |false |false | | | |false |false |string |true |false |false |true |
| 4 |addressLine1 |.address.address |false |false | | | |false |false |string |true |false |false |true |
| 5 |addressLine2 | |false |false | | | |false |false |string |false |false |false |true |
| 6 |state |.address.state |false |false | | | |false |false |string |false |false |false |true |
| 7 |country |.address.country |false |false | | | |false |false |string |true |false |false |true |
| 8 |postalCode |.address.postalCode |false |false | | | |false |false |string |true |false |false |true |
| 9 |territory | |false |false | |'NA' | |false |false |string |true |false |false |true |

Point worth noting:
- The `territory` column is a `NOT NULL` column and is not mapped to any data field. We set a `insert value` for it to avoid INSERT error.

***Manager:***

| position |column |field |key |autoGenerate |inputField |insertValue |updateValue |version |softDelete |dataType |notNull |keyEligible |versionEligible |softDeleteEligible |
| - | - | - | - | - | - | - | - | - | - | - | - | - | - | - |
| 1| employeeNumber| .manager.employeeNumber| true| false| | | | false| false| number| true| true| true| false| 
| 2| lastName| .manager.lastName| false| false| | | | false| false| string| true| false| false| true| 
| 3| firstName| .manager.firstName| false| false| | | | false| false| string| true| false| false| true| 
| 4| extension| .manager.extension| false| false| | | | false| false| string| true| false| false| true| 
| 5| email| .manager.email| false| false| | | | false| false| string| true| false| false| true| 
| 6| officeCode| .officeCode| false| false| | | | false| false| string| true| false| false| true| 
| 7| reportsTo| | false| false| | | | false| false| number| false| false| true| false| 
| 8| jobTitle| .manager.jobTitle| false| false| | 'office manager'| | false| false| string| true| false| false| true| 

Point worth noting:
- The `job title` for the `"manager"` is `"office manager"`. We set the `insert value` to be `'office manager'`.


***Employee:***

| position |column |field |key |autoGenerate |inputField |insertValue |updateValue |version |softDelete |dataType |notNull |keyEligible |versionEligible |softDeleteEligible |
| - | - | - | - | - | - | - | - | - | - | - | - | - | - | - |
| 1| employeeNumber| .employees..employeeNumber| true| false| | | | false| false| number| true| true| true| false| 
| 2| lastName| .employees..lastName| false| false| | | | false| false| string| true| false| false| true| 
| 3| firstName| .employees..firstName| false| false| | | | false| false| string| true| false| false| true| 
| 4| extension| .employees..extension| false| false| | | | false| false| string| true| false| false| true| 
| 5| email| .employees..email| false| false| | | | false| false| string| true| false| false| true| 
| 6| officeCode| .officeCode| false| false| | | | false| false| string| true| false| false| true| 
| 7| reportsTo| .manager.employeeNumber| false| false| | | | false| false| number| false| false| true| false| 
| 8| jobTitle| .employees..jobTitle| false| false| | | | false| false| string| true| false| false| true| 

Point worth noting:
- We mapped the `reportsTo` to `.manager.employeeNumber` manually.


## Optimistic Locking
---

CRUD service supports optimistic locking through a version control column. We can designate an integer column to be the version control column by setting its `version` attribute to `true` in the column bindings.

For example, we can hypothetically add a `version` column to the `orders` table as

```sql
alter table orders add version integer
```

and enable optimistic locking on this table by designating this new column as the version control column. To do this, we set the `version` attribute for this new column to `true` in the column bindings, as

```json
    {
        "position": 1,
        "column": "version",
        "field": ".version",
        "key": false,
        "autoGenerate": false,
        "inputField": null,
        "insertValue": null,
        "updateValue": null,
        "version": true,
        "softDelete": false,
        "dataType": "number",
        "notNull": false,
        "keyEligible": false,
        "versionEligible": false,
        "softDeleteEligible": true
    },
```

This column needs to be mapped to a version field of the object. It will be automatically set to `1` at creation, and incremented by each update operation. The service engine will make sure the updated version matches the version in database.  

## Soft Delete
---

CRUD service supports soft delete through a soft delete column. We can designate a `char` column to be the soft delete column by setting its `softDelete` attribute to `true` in the column bindings.

For example, we can hypothetically add a `deleted` column to the `orders` table as

```sql
alter table orders add deleted char(1)
```

and enable optimistic locking on this table by designating this new column as the soft delete column. To do this, we set the `softDelete` attribute for this new column to `true` in the column bindings , as

```json
    {
        "position": 1,
        "column": "deleted",
        "field": null,
        "key": false,
        "autoGenerate": false,
        "inputField": null,
        "insertValue": null,
        "updateValue": null,
        "version": false,
        "softDelete": true,
        "dataType": "string",
        "notNull": false,
        "keyEligible": false,
        "versionEligible": false,
        "softDeleteEligible": true
    },
```

This column is not mapped to any data field. However, it will be automatically populated with 'N' by the create operation and updated to 'Y' by the delete operation.

## Conclusion
---

CRUD service may be used to create repository for simple object mapped to a single table to complex object mapped to many tables, with or without read-write asymmetry. The key of the CRUD service development is in writing the SQL query for read. Then the `write` component can be generated from the `read` component and edited by the developer.  
