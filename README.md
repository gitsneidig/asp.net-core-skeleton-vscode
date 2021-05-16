# asp.net-core-skeleton-vscode
A basic ASP.NET Core skeleton foundation for a project including setup for:
- A React frontend
- Clean Architecture approach with domain layers
  - API
    - ASP.NET Core Web API
  - Application 
    - Class Library
  - Domain
    - Class Library
  - Persistence
    - Class library
    - Entity Framework
    - SQLite Databases

Credit for the skeleton to 
https://www.udemy.com/course/complete-guide-to-building-an-app-with-net-core-and-react/

Documentation below typed in by Scott Neidig.

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
- In the Configure() method comment out app.UseHttpsRedirection()

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
  - Microsoft.EntityFrameworkCore
  - Microsoft.EntityFrameworkCore.Sqlite
- Add to /API
  - Microsoft.EntityFrameworkCore.Design

## DbContext Setup
Delete the default Class1.cs file in /Persistence.

### Add DBContext entity classes
- Create a New C# Class file named DataContext.cs
- Make DataContext class derive from DbContext
- Add a constructor for DataContext
  - Constructor should take parameter DbContextOptions options
  - Also pass options to the base class constructor
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

## Seed Data
### Add Seed class
Inside the /Persistence directory create a New C# Class file named Seed.cs.

To start seeding the database with data create objects for your entity domains. 

Inside the Seed class create a public static async method called SeedData that takes a parameter of type DataContext.

For each domain entity associated with the DataContext object, create a conditional statement that checks to see if there is already data in the database for the entity. 
```
if (context.Activities.Any()) { ... }
```
Inside the conditional create a new List<YOURDOMAIN> that and load it with new objects for your domain.

If there is already data then creating this list will be skipped, if there is no data the value from your list will be used to seed the database.

Underneath the list declaration code add code to the new list to tracked context items and then SaveChanges.
```
await context.Activities.AddRangeAsync(activities);
await context.SaveChangesAsync();
```

### Run SeedData() Method from Program.cs
Call the static method SeedData in the new Seed class from the try/catch block in Program.cs and pass it the context from services.
```
await Seed.SeedData(context);
```

Update the Main method to be async and return a Task.
```
public static async Task Main(string[] args) { ... }
```

In the try/catch block update the call to Migrate the context to use await and MigrateAsync.
```
await context.Database.MigrateAsync();
```

Update the call to Run to use await and RunAsync.
```
await host.RunAsync();
```

There is no real benefit to adding Async to all these methods since the Program class will only be called once at the start of the application, but it does no harm.

Run dotnet watch run in the /API directory and check SQLite to confirm seed rows are present.

## Setup API Controller
### Add a Base API Controller
In /API/Controllers create a BaseApiController class and update the following:
- BaseApiController derives from ControllerBase
- BaseApiController has the [ApiController] attribute
- BaseApiController has a Route("api/[controller]") as a placeholder

All controllers in the project will derive from this controller.

### Add an API Controller for Domain Entities
In /API/Controllers create a YOURDOMAINController class and update the following:
- YOURDOMAINController derives from BaseApiController
- Add constructor
  - Inject DataContext parameter
  - Initialize field from parameter

### Add a Few Basic Test Methods to Handle GET Requests to the New Controller Classes
```
[HttpGet]
public async Task<ActionResult<List<Activity>>> GetActivities() {
  return await _context.Activities.ToListAsync();
}

[HttpGet("{id}")] //activities/id
public async Task<ActionResult<Activity>> GetActivity(System.Guid id) {
  return await _context.Activities.FindAsync(id);
}
```

Restart the application if needed and test the new endpoints.
- http://localhost:5010/api/activities
- http://localhost:5010/api/YOURDOMAIN
- http://localhost:5010/api/activities/8bbf8713-7ada-4a04-ba6d-bf08faa170c5

## Source Control
- Add gitignore file for .net applications
- Add appsettings.json to gitignore (and any other files that may hold sensitive credentials)
