## Fetch Json from the IBM i with an RPG program call

This little spike shows how to call an IBM i RPG program object that returns Json through its parameter list with ASNA Visual RPG.

This is a very performant way to fetch data from the IBM i. It is especially well-suited for getting paged sets of data for Web or Web services work. It has a few drawbacks:

* The RPG parameter list must declare a string large enough to hold the Json result. The maximum length of a *Char field in this context is 32,766. This technique is intended for fetching well-scoped chunks of data and this limitation is probably not a big deal. This little spike is using a 10K field for the Json result. 

* This technique uses the IBM i SQL's JSON_OBJECT and JSON_ARRAYAGG functions with SQL's INTO clause. I've not been able to make SELECT statement with those functions and the INTO clause work with dynamic SQL. Unless there is a solution to that challenge this technique requires a specific RPG program object for each file you need to work with. There is less than 12 lines of SQL that need to be unique, so it's not an enormous constraint, but it is a constraint. 

* The JSON_OBJECT and JSON_ARRAYAGG functions don't work directly with ORDER BY and several other clauses. Therefore, you need to use a subquery to fetch the data. For this spike, the called RPG program first uses dynamic SQL to write a result set table in QTEMP and then queries that temporary table to generate the Json. See this SQL below. 

* This spike uses the [NewtonSoft.Json](https://www.nuget.org/packages/Newtonsoft.Json/) nuget package to deserialize the Json string to a .NET object. 

This SQL is passed to the RPG program to fetch the data needed (with clauses as needed) and write it to a temporary table:

```
CREATE TABLE qtemp/pager as
(
    WITH result AS 
    (
      SELECT  cmcustno, cmname
      FROM  examples/cmastnewL2 
      ORDER BY  cmname, cmcustno
      LIMIT 16
      OFFSET 0
    ) 
    SELECT * FROM result 
) 
WITH DATA 
```

After the temporary table is created, this static SQL queries that table and puts the Json result in the `Json` variable RPG (which in the RPG program's parameter list). This static SQL is what must be created for each file. 

```
SELECT JSON_OBJECT
(
    customers' value JSON_ARRAYAGG
    (
      JSON_OBJECT
        (
          'CMCustNo' value cmcustno,
          'CMName' value TRIM(cmname)
        )
    )
)
INTO :json
FROM 
(
  SELECT cmname, cmcustno 
  FROM qtemp/pager
  ORDER BY cmname, cmcustno
);
```


The code below shows how the Json returned by the RPG program is deserialized into a .NET object. The `Json` variable referenced in the code below is the character string of Json the RPG program returned. 


```
DclFld jo Type(JsonObject) 

Try 
    CallProgramObject(DGDB, ProgramLibrary, RPGProgramToCall, Sql)  
Catch ex Type(Exception) 
    ...
EndTry 

jo = NewtonSoft.Json.JsonConvert. +
               DeserializeObject(Json, +
                                 *TypeOf(JsonObject)) *As JsonObject
LeaveSr jo
```

These two small classes are the target of the Json deserialization:

```
BegClass JsonObject
    DclProp customers Type(Customer) Rank(1) Access(*Public)
EndClass 

BegClass Customer Access(*Public) 
    DclProp CMName Type(*Char) Len(40) Access(*Public) 
    DclProp CMCustNo Type(*Packed) Len(9,0) Access(*Public) 
EndClass                 
```


