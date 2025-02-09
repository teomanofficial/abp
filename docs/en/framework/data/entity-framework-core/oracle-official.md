# Switch to EF Core Oracle Provider

This document explains how to switch to the **Oracle** database provider for **[the application startup template](../../../solution-templates/layered-web-application)** which comes with SQL Server provider pre-configured.

> Before switching your provider, please ensure your Oracle version is **v12.2+**. In the earlier versions of Oracle, there were long identifier limitations that prevents creating a database table, column or index longer than 30 bytes. With [v12.2](https://docs.oracle.com/en/database/oracle/oracle-database/12.2/newft/new-features.html#GUID-64283AD6-0939-47B0-856E-5E9255D7246B) "The maximum length of identifiers is increased to 128 bytes". **v12.2** and later versions, you can use the database tables, columns and indexes provided by ABP without any problems. 

## Replace the Volo.Abp.EntityFrameworkCore.SqlServer Package

`.EntityFrameworkCore` project in the solution depends on the [Volo.Abp.EntityFrameworkCore.SqlServer](https://www.nuget.org/packages/Volo.Abp.EntityFrameworkCore.SqlServer) NuGet package. Remove this package and add the same version of the [Volo.Abp.EntityFrameworkCore.Oracle](https://www.nuget.org/packages/Volo.Abp.EntityFrameworkCore.Oracle) package.

## Replace the Module Dependency

Find ***YourProjectName*EntityFrameworkCoreModule** class inside the `.EntityFrameworkCore` project, remove `typeof(AbpEntityFrameworkCoreSqlServerModule)` from the `DependsOn` attribute, add `typeof(AbpEntityFrameworkCoreOracleModule)`

Also replace `using Volo.Abp.EntityFrameworkCore.SqlServer;` with `using Volo.Abp.EntityFrameworkCore.Oracle;`.

## UseOracle()

Find `UseSqlServer()` calls in your solution, replace with `UseOracle()`. Check the following files:

* *YourProjectName*EntityFrameworkCoreModule.cs inside the `.EntityFrameworkCore` project.
* *YourProjectName*DbContextFactory.cs inside the `.EntityFrameworkCore` project.


In the `CreateDbContext()` method of the *YourProjectName*DbContextFactory.cs, replace the following code block

```csharp
var builder = new DbContextOptionsBuilder<YourProjectNameDbContext>()
                .UseSqlServer(configuration.GetConnectionString("Default"));
```

with this one (just changes `UseSqlServer(...)` to `UseOracle(...)`)
```csharp
var builder = new DbContextOptionsBuilder<YourProjectNameDbContext>()
                .UseOracle(configuration.GetConnectionString("Default"));
```

> Depending on your solution structure, you may find more code files need to be changed.

## Change the Connection Strings

Oracle connection strings are different than SQL Server connection strings. So, check all `appsettings.json` files in your solution and replace the connection strings inside them. See the [connectionstrings.com](https://www.connectionstrings.com/oracle/) for details of Oracle connection string options.

You typically will change the `appsettings.json` inside the `.DbMigrator` and `.Web` projects, but it depends on your solution structure.

## Re-Generate the Migrations

The startup template uses [Entity Framework Core's Code First Migrations](https://docs.microsoft.com/en-us/ef/core/managing-schemas/migrations/) by default. 
EF Core Migrations depend on the selected DBMS provider. Changing the DBMS provider, may not work with the existing migrations.

* Delete the `Migrations` folder under the `.EntityFrameworkCore` project and re-build the solution.
* Run `Add-Migration "Initial"` on the Package Manager Console window (select the `.DbMigrator`  (or `.Web`) project as the startup project in the Solution Explorer and select the `.EntityFrameworkCore` project as the default project in the Package Manager Console).

This will scaffold a new migration for Oracle.

Run the `.DbMigrator` project to create the database, apply the changes and seed the initial data.

## ORA-12899: value too large for column fix

Oracle limits strings to `NVARCHAR2(2000)` when the migration is created. Some of the entity properties may extend it. You can check the known and reported properties of ABP modules entities that can extend this limit. To prevent this problem, you need to convert the `string` type to `long`  type first and generate a new migration. Then convert the `long` type to `clob` type with maximum length. 

### First Migration

Update you application DbContext `OnModelCreating` method:

```csharp
protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        /* Include modules to your migration db context */

        builder.Entity<OpenIddictToken>(b =>
        {
            b.Property(q => q.Payload).HasColumnType("long").HasMaxLength(int.MaxValue);
        });
        
        builder.Entity<AuditLog>(b =>
        {
            b.Property(x => x.Exceptions).HasColumnType("long").HasMaxLength(int.MaxValue);
        });
        
        builder.Entity<AuditLogAction>(b =>
        {
            b.Property(x => x.Parameters).HasColumnType("long").HasMaxLength(int.MaxValue);
        });
        
        /* Configure your own tables/entities inside here */
    }
```

Create a new migration using dotnet tooling under EntityFrameworkCore layer of your solution: `dotnet ef migrations add Oracle_Long_Conversion`

### Second Migration

Update you application DbContext `OnModelCreating` method:

```csharp
protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        /* Include modules to your migration db context */

        builder.Entity<OpenIddictToken>(b =>
        {
            b.Property(q => q.Payload).HasColumnType("clob").HasMaxLength(4000);
        });
        
        builder.Entity<AuditLog>(b =>
        {
            b.Property(x => x.Exceptions).HasColumnType("clob").HasMaxLength(4000);
        });
        
        builder.Entity<AuditLogAction>(b =>
        {
            b.Property(x => x.Parameters).HasColumnType("clob").HasMaxLength(4000);
        });
        
        /* Configure your own tables/entities inside here */
    }
```

Create a new migration using dotnet tooling under EntityFrameworkCore layer of your solution: `dotnet ef migrations add Oracle_Clob_Conversion`

Run DbMigrator (or use dotnet tooling) to migrate the oracle database.

## Run the Application

It is ready. Just run the application and enjoy coding.
