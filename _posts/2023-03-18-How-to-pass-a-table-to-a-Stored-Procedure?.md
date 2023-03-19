---
layout: post
title:  "How to pass a table to a Stored Procedure?"
---
This can be useful when we want to get data in bulk from a Store Procedure. 
For example uploding a file with a few columns and we need the values of the columns to be the parameters of the query.

We will explain here the answer number 3 mentioned in the stackoverflow answer. [C# SQL Server - Passing a list to a stored procedure](https://stackoverflow.com/a/7097418/3290276)

In the SQL Server Management Studio(SSMS) Create a new type as table. This will be the shape of the data that you will be passing to the Store Procedure(SP)
```sql
 CREATE TYPE [dbo].[myExternalTablev2] AS TABLE(
    [OrganizationLevel] [int] NOT NULL,
	  [BusinessEntityID ] [int] NOT NULL
);
```

Then create the SP where the input will be of the type that we previously created. Note it has to be READONLY
We can then use that parameter as a table and join it to the tables that contain the data that we are looking for.
```sql
USE [AdventureWorks2019]
GO

SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

CREATE OR ALTER   PROCEDURE [dbo].[uspGetEmployeeManagersFromTable]
    @ExternalValues myExternalTablev2 READONLY
AS
BEGIN
    SET NOCOUNT ON;

		Select e.BusinessEntityID, e.OrganizationLevel, e.JobTitle
		FROM [HumanResources].[Employee] e 
		INNER JOIN [Person].[Person] as p
		ON p.[BusinessEntityID] = e.[BusinessEntityID]
		INNER JOIN @ExternalValues ex
		ON ex.OrganizationLevel = e.OrganizationLevel
		AND ex.BusinessEntityID = e.BusinessEntityID
END;
GO

```

Pass the table and execute the SP
```csharp
        private static void ExecSPfromTable(SqlConnection connection)
        {
            using (SqlCommand command = new SqlCommand("uspGetEmployeeManagersFromTable", connection))
            {
                var table = new DataTable();

                table.Columns.Add("OrganizationLevel", typeof(string));
                table.Columns.Add("BusinessEntityID", typeof(string));

                // loop on the data to add the rows with the values
                table.Rows.Add(3, 4); //(OrganizationLevel, BusinessEntityID)
                table.Rows.Add(3, 5);

                command.CommandType = CommandType.StoredProcedure;
                command.Parameters.Add(new SqlParameter("@ExternalValues", table));

                using (SqlDataReader reader = command.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        Console.WriteLine("{0}\t{1}\t{2}",
                            reader.GetInt32(0),
                            reader.GetInt16(1),
                            reader.GetString(2)
                            );
                    }
                }
            }
        }
```

Main:
```csharp
         static void Main(string[] args)
        {
            Console.WriteLine("Start");
            SqlConnectionStringBuilder sConnB = new SqlConnectionStringBuilder();
            sConnB.DataSource = "YourServer";
            sConnB.InitialCatalog = "AdventureWorks2019";
            sConnB.UserID = "YourTestUser";
            sConnB.Password = "YourTestPassWord";

            using (var connection = new SqlConnection(sConnB.ConnectionString))
            {
                connection.Open();
                ExecSPfromTable(connection);
            }
            Console.WriteLine("End - enter any key to exit");
            Console.ReadKey(true);
        }
```
Results:

![image](https://user-images.githubusercontent.com/4723976/226143187-9c6329dc-b531-4075-a877-f14189fbe64e.png)

References: 
- For this example we are using ```using System.Data.SqlClient``` which can be added to the project as a Nuget Package
- Using the public available database [AdventureWorks2019](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver16&tabs=ssms)
