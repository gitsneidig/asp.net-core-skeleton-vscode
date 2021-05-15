# asp.net-core-skeleton-vscode
A basic ASP.NET Core skeleton foundation for a project including setup for:
- A React frontend
- Clean Architecture approach with domain layers
-- API
--- ASP.NET Core Web API
-- Application 
--- Class Library
-- Domain
--- Class Library
-- Persistence
--- Class library
--- Entity Framework
--- SQLite Databases

## .NET CLI Commands
Check .NET SDK and runtime version.
```
dotnet --info
```
Review available commands.
```
dotnet --help
```
Review list of available project and file templates.
```
dotnet new -l
```

## Inital Setup
Make a directory for project files.
```
mkdir Reactivities
dotnet new sln
```
The dotnet new command will create a solution file with the name of the containing directory.

Create projects
```
dotnet new webapi -n API
dotnet new classlib -n Application
dotnet new classlib -n Domain
dotnet new classlib -n Persistence
```

Add newly created projects to the solution
```
dotnet sln add API/API.csproj
dotnet sln add Application
dotnet sln add Domain
dotnet sln Add Persistence
```

Setup dependencies between projects
```
cd API
dotnet add reference ../Application
cd ../Application
dotnet add reference ../Persistence
dotnet add reference ../Domain
cd ../Persistence
dotnet add reference ../Domain
```

Open the solution in VSCode and review newly created project.
From project root
```
code .
```

Press yes to add assets to build and debug if prompted.

The API project should be the startup project.

## Configure Project Files
Make the following updates inside /API directory.
```
cd API
```
#### Properties/launchSettings.json
Optional settings in /API/Properties/launchSettings.json
- Set launchBrowser: false
- Remove https from applicationUrl if not needed for dev

#### appsettings.Development.json
Get more robust info in the console.
- Set Logging -> LogLevel -> Microsoft: Information 

#### Startup.cs
-- In the Configure() method comment out app.UseHttpsRedirection()

#### VSCode
- Toggle File > Auto Save if needed

### Run Project and Confirm Swagger Output is Available
Open terminal in VSCode and run. 
```
dotnet watch run
```
Use dotnet watch run instead of dotnet run to get hot reload. 

In browser visit URL (check launchSettings.json if port has changed).
http://localhost:5010/swagger/index.html

## Create Entity Domains
Make the following updates inside /Domain directory.
```
cd ../Domain
```
Delete the default Class1.cs file in /Domain.

### Add domain entity classes
- Create a New C# Class file named for each domain entity
- Add properties as needed
- Use System.Guid as type for primary keys

## Entity Framework
Make the following updates inside /Persistence directory.
```
cd ../Persistence
```

### Add NuGet Packages
Make sure NuGet package versions match the version of .NET Core you are using.
- Add to /Persistence
-- Microsoft.EntityFrameworkCore
-- Microsoft.EntityFrameworkCore.Sqlite
- Add to /API
-- Microsoft.EntityFrameworkCore.Design

## DbContext Setup
Delete the default Class1.cs file in /Persistence.

### Add DBContext entity classes
- Create a New C# Class file named DataContext.cs
- Make DataContext class derive from DbContext
- Add a constructor for DataContext
-- Constructor should take parameter DbContextOptions options
-- Also pass options to the base class constructor
- Add a property of type DbSet<YOURDOMAIN> to DataContext for each domain entity

### /API/Startup.cs
In ConfigureServices method add a service for the newly created DBContext.
- AddDbContext types should match DbContext class name
- Set options to use SqLite database
```
services.AddDbContext<DataContext>(opt => { opt.UseSqlite() })
```

#### Get Connection String from Config
Change default setup for how IConfiguration is used by Startup. 

Remove Configuration assignment from constructor.
```
Configuration = configuration;
```

Remove default Configuration property from class.
```
public IConfiguration Configuration { get; }
```

Rename configuratin parameter passed in to startup to config.
```
public Startup(IConfiguration config) { }
```

Initialize a field from the Startup parmeter.
```
private readonly IConfiguration _config;
```

Assign value to new field in Startup constructor.
```
public Startup(IConfiguration config) {
  _config = config
 }
```

In ConfigureServices method set UseSqlite parameter to value from config field.
```
services.AddDbContext<DataContext>(opt => { opt.UseSqlite(_config.GetConnectionString("DefaultConnection")) })
```

In appsettings.Development.json add property and value for Sqlite ConnectionStrings. 
```
"ConnectionStrings": {
  "DefaultConnection": "Data source=reactivities.db"
}
```

## Create Entity Framework Code First Migration

### Setup
Check to make sure dotnet-ef is installed and install if needed.
```
dotnet tool list --global
```

Download from nuget.org if needed.
[https://www.nuget.org/packages/dotnet-ef](https://www.nuget.org/packages/dotnet-ef)

If you run into version issues try using the following commands to switch to the needed version.
```
dotnet tool install --global dotnet-ef --version 5.0.1
dotnet tool update --global dotnet-ef --version 5.0.1
```

Consult MS documentation for more info on installing dotnet-ef.
[https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-tool-uninstall](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-tool-uninstall)

Review the list of available dotnet-ef commands
```
dotnet ef -h
```

### Add the Initial Migration

Note the use of switches
- -p specifies which project has DbContext file
- -s specifies startup project
```
dotnet ef migrations add InitialCreate -p Persistence/ -s API/
```

A new Migrations directory containing should appear in /Persistence containing associated migration files.

### Update Program.cs to Create Database on Startup if it is Missing

This code provides access to the DataContext using the ServicesProvider. Using the DataContext service context.Database.Migrate() can be run on startup to apply any pending migrations for the context to the database, and create the database if it does not already exist. 

```
public static void Main(string[] args)
  {
    // CreateHostBuilder(args).Build().Run();
    var host = CreateHostBuilder(args).Build();

    using var scope = host.Services.CreateScope();

    var services = scope.ServiceProvider;

    try {
      var context = services.GetRequiredService<DataContext>();
      context.Database.Migrate();
    } 
      catch (Exception ex) 
    {
      var logger = services.GetRequiredService<ILogger<Program>>();
      logger.LogError(ex, "An error occured during migration");
    }

    host.Run();
  }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
          webBuilder.UseStartup<Startup>();
        });
```

Run the application from the terminal and afterwards you should see a new Sqlite database file in reactivites.db in /API.

Use Sqlite extension in VSCode to view database context and confirm tables exist for domain entities.



