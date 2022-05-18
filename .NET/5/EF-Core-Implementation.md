# EF Core Setup
## NuGet Installation
- Microsoft.EntityFrameworkCore - *5.0.17*
- Microsoft.EntityFrameworkCore.SqlServer - *5.0.17*
> *Warning*  
> More recent versions might turn out incompatible with .NET 5. If this is the case and an error shows up at installation, make sure that you are downloading the correct version.

## Entity classes creation
The amazing part about EF Core is that it automatically creates relations among data entities based on their C# definition, using their *table* and *navigation* properties. Imagine you need a simple database with **Cities** with **Restaurants** in them, for a stablishment review searching app:

```cs
public class City 
{
    // Table properties
    public int CityID { get; set; }
    public int PostalCodePrefix { get; set; }

    // Navigation properties
    public ICollection<Restaurant> Restaurants { get; set;}
}

public class Restaurant {
    // Table properties
    public int RestaurantID { get; set; }
    public int CityID { get; set; }
    public string Name { get; set; }

    // Navigation properties
    public City City { get; set; }
}
```
Some requirements for class and property naming (not case sensitive) in EF Core are: 
- PKs must be `Id` or `_classname_Id`
- FKs must be `_refClass_Id`
- There must be a navigation property for every table relation. In this example, a ``Restaurant`` can only be from one ``City``, and in a single ``City`` there can be a lot of `Restaurant` instances. Note how it's named after the class they reference, and it's using an `ICollection` instance and the plural form of it's name for one-to-many relations.

> Although it's permitted to begin properties with a lower-case letter, I'ts recommended to follow C#'s naming convention and do so with an upper-case one.

## Data Context Creation
Entity Framework uses `DbContext` class as a repository to access our data in the database. A data context to hold both `Restaurants` and `Cities` looks like this:

```cs
public class DiningContext : DbContext // Inherit from DbContext class
{
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer("your-connection-string");
                // or UseSQLite() if your're using it instead.

    public DbSet<City> Cities { get; set; }
    public DbSet<Restaurant> Restaurants { get; set; }
}
```

In that `OnConfiguring` method, we're setting up our context to use *SQL Server* mode to connect to our database. It can be also done in `Startup.cs`, changing a  few things, but this approach seems more straight-forward to me.

Note where we are declaring both `Players` and `Rooms` properties. They will contain a repository for the whole table in our database later.

Now we can set custom relations and extra features for our tables and properties, overriding the virtual method `OnModelCreating`. This time, we'll make both tables PKs autoincrement themselves when inserting a new item.

```cs
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Restaurant>()
        .Property(o => o.RestaurantID)
        .ValueGeneratedOnAdd();// Sets the property to be auto-generated

    modelBuilder.Entity<City>()
        .Property(o => o.RestaurantID)
        .ValueGeneratedOnAdd();
}
```
`OnModelCreating` overrides EF's automatic setup and sets our custom realtions and property features. It's very useful for a larger database, and there is a lot of documentation about it on [MS Docs](docs.microsoft.com).

With this, our context looks like this:

```cs
public class DiningContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer("your-connection-string");
                // or UseSQLite() if your're using it instead.

    public DbSet<City> Cities { get; set; }
    public DbSet<Restaurant> Restaurants { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
    modelBuilder.Entity<Restaurant>()
        .Property(o => o.RestaurantID)
        .ValueGeneratedOnAdd();// Sets the property to be auto-generated

    modelBuilder.Entity<City>()
        .Property(o => o.RestaurantID)
        .ValueGeneratedOnAdd();// Sets the property to be auto-generated
    }
}
```

## `Startup.cs` setup
For this step, we'll need to tell the compiler that we'll be using EF Core's services, adding these lines to the `ConfigureServices` method:

```cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();
    // ... other services
    services.AddDbContext<DiningContext>();
    services.AddEntityFrameworkSqlServer();
}
```

## Database Initialization
Now, we need to tell the main program to initialize our database. This is done in `Program.cs` modificating our `Main` method like this:

```cs
// Custom method to initialize our DB if it doesn't exist yet, 
// and raise an error if anything happens
private static void CreateDbIfNotExists(IHost host)
{
    using (var scope = host.Services.CreateScope())
    {
        var services = scope.ServiceProvider;
        try
        {
            using (var context = services.GetRequiredService<DiningContext>()) 
            {
                context.Database.EnsureCreated();
            }
        }
        catch (Exception ex)
        {
            var logger = services.GetRequiredService<ILogger<Program>>();
            logger.LogError(ex, "An error occurred creating the DB.");
        }
    }
}

public static void Main(string[] args)
{
    var host = CreateHostBuilder(args).Build();
    // Separate Build() and Run() methods to fit our method between them
    // if you need to
    CreateDbIfNotExists(host);
    host.Run();
}
```

# EF Core data access and creation
Okay, you have your data structure all set up and ready. Now all what's left to do is store and retrieve the data. For this we'll use C# exclusively,
using our data context and *LINQ*.

## Store data
To **store** your data you'll just create them as normal C# objects of your entity classes. Then, you can use the DBSet's `AddAsync` method to asynchronichally add new items.
```cs
// Add a new City to our DB
using (var context = new DiningContext()) 
{
    var cities = context.Cities;
    var newCity = new City();
    newCity.PostalCodePrefix = 21;
    // Set other properties
    // ...
    await cities.AddAsync(newCity);
    await context.SaveChangesAsync(); // ALWAYS save changes to update the DB
}
```
Our context is merely a **repository** for our data. That `SaveChangesAsync` method updates our database to be up to date to our context repository.

## Retrieve data
To **retrieve** data you use *LINQ* statements. LINQ is a data access library made by Microsoft that makes accessing datasets from C# a lot easier.  
Here go some quick examples using our existing context.
```cs
using (var context = new DiningContext())
{
    var restaurants = context.Restaurants;

    // Get all restaurants
    var allRestaurants = restaurants.ToList();
        
    // Get the restaurant with 1 as ID
    var singleRestaurant = restaurants
        .Single(r => r.RestaurantID == 1);
        
    // All the restaurants that are located in city #1
    var filteredRestaurants = restaurants
        .Where(r => r.CityID == 1)
        .ToList();
}
```
As you can see, to access the data we use chained LINQ methods to select and filter it. The lambda statements inside each LINQ method  
indicate how to to this selection. In the example, `r` is a reference to the Restaurant object which LINQ is currently iterating on. Knowing this,  
on the right side of `=>`, we can now perform any boolean expression to filter or select the data you need. 

> It's LINQ convention to write each chained LINQ expression under the previous one, like in the example.
> ```cs
>  var result = table
>     .Where(expression)
>     .Single(expression)
>     .ToList();
> ```




## Afterword
Now you're all set! Later updates will contain information about more complex LINQ expression.
