## CachingWebApi


### Types of Caching
Caching has 2 categories-  
- **In Memory Cache**
    - Lives within the application server itself
    - Chunks of ram will be utilized for caching the information
    - Reduces the capabilities of the application as we are using the same application server
    - Much more faster if we access from the ram itself than different server   
- **Distributed Cache**
    - Lives in separate individual server with dedicated purpose
    - Does not take any resources from our application server
    - As it lives outside the application server, once our application scales up, we will be actually able to utilize cache with all of the different instances(multiple different applications will use the same sigle cache for multiple use)


---
### Implementatins Steps by Steps:
1. Create a folder for the project and go to that directory
    ```
    mkdir CachingWebApi
    cd CachingWebApi
    ```
2. Create a webApi project 
    ```
    dotnet new webapi -n CachingWebApi -f net6.0
    ```
3. Open the project in visual studio code
    ```
    code .
    ```
4. Add a gitignore file into the solution
    ```
    dotnet new gitignore
    ```
5. Add a new file named README.md
    ```
    touch README.md
    ```
6. We will be using distributed cache with Redis. For that we wiil use **Docker** to make an instance of redis and run it in the docker
    ```
    docker run --name my-redis -p 6379:6379 -d redis
    ```
    This will download the docker image and run it
    ```
    docker ps
    ```
    This will show you the list of running images including redis one we created
7. Now add the required packages
    ```
    dotnet add package Microsoft.EntityFrameWorkCore -v 6.0.28
    dotnet add package Microsoft.EntityFrameworkCore.SqlServer -v 6.0.28
    dotnet add package Microsoft.EntityFrameworkCore.Design -v 6.0.28
    dotnet add package Microsoft.EntityFrameworkCore.Tools -v 6.0.28
    dotnet add package StackExchange.Redis
    dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis -v 6.0.28
    ```
8. Create a folder named **Models** and create a class named **Driver**
    ```
    mkdir Models
    dotnet new class -n Driver
    ```
    ```
    public class Driver
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public int DriverNb { get; set; }
    }
    ```
9. Create a folder named **Data** and add a class **AppDbContext**
    ```
    mkdir Data
    dotnet new class -n AppDbContext
    ```
    ```
    public class AppDbContext : DbContext
    {
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
        
    }

    public DbSet<Driver> Drivers { get; set; }
    }  
    ```
10. Add connectionstring into appSettings.json
    ```
      "ConnectionStrings" : 
      {
        "SampleDbConnection" : "Server=localhost;Database=SampleDb;User=sa;Password=Docker@123"
      }
    ```
11. Inject the AppDbContext into program file
    ```
    //Add Database service
    builder.Services.AddDbContext<AppDbContext>(opt => opt.UseSqlServer(builder.Configuration.GetConnectionString("SampleDbConnection")));
    ```
12. Now add a migration and update database
    ```
    dotnet ef migrations add initial_migration
    dotnet ef database update
    ```
13. Create a folder and create an interface **ICacheService**
    ```
    dotnet new interface -n ICacheService
    ```
    ```
    public interface ICacheService
    {
        T GetData<T>(string key);
        bool SetData<T>(string key, T value, DateTimeOffset expirationTime);
        object RemoveData(string key);
    }
    ```
14. Create a class named **CacheService** and inherit **ICacheService**
    ```
    dotnet new class -n CacheService
    ```
    ```
    using System.Text.Json;
    using StackExchange.Redis;

    namespace CachingWebApi;

    public class CacheService : ICacheService
    {
        private IDatabase _cacheDb;
        public CacheService()
        {
            var redis = ConnectionMultiplexer.Connect("localhost:6379");
            _cacheDb = redis.GetDatabase();
        }
        public T GetData<T>(string key)
        {
            var value = _cacheDb.StringGet(key);
            if(!string.IsNullOrEmpty(value))
            {
                return JsonSerializer.Deserialize<T>(value);
            }
            return default;
        }

        public object RemoveData(string key)
        {
            var exists = _cacheDb.KeyExists(key);
            if(exists)
            {
                return _cacheDb.KeyDelete(key);
            }
            return false;
        }

        public bool SetData<T>(string key, T value, DateTimeOffset expirationTime)
        {
            var expiryTime = expirationTime.DateTime.Subtract(DateTime.Now);
            return _cacheDb.StringSet(key, JsonSerializer.Serialize(value),expiryTime);
        }
    }

    ```
15. Now inject the ICacheService and its implementatin into program file
    ```
    builder.Services.AddScoped<ICacheService, CacheService>();
    ```
16. Add a new apiController named **DriversController** and write accordingly
    ```
    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Threading.Tasks;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.EntityFrameworkCore;

    namespace CachingWebApi.Controllers; 

    [ApiController]
    [Route("api/[controller]")]
    public class DriversController : ControllerBase
    {
        private readonly ILogger<DriversController> _logger;
        private readonly ICacheService _cacheService;
        private readonly AppDbContext _context;

        public DriversController(ILogger<DriversController> logger, ICacheService cacheService, AppDbContext context)
        {
            _logger = logger;
            _cacheService = cacheService;
            _context = context;
        }

        [HttpGet("drivers")]
        public async Task<ActionResult> Get()
        {
            //check cache data
            var cacheData = _cacheService.GetData<IEnumerable<Driver>>("drivers");

            if(cacheData != null && cacheData.Count() > 0)
            {
                return Ok(cacheData);
            }
            cacheData = await _context.Drivers.ToListAsync();

            //Set Expiry time
            var expiryTime = DateTimeOffset.Now.AddSeconds(30);
            _cacheService.SetData<IEnumerable<Driver>>("drivers",cacheData, expiryTime);

            return Ok(cacheData);
        }

        [HttpPost("AddDriver")]
        public async Task<IActionResult> Post(Driver value)
        {
            var addedObj = await _context.Drivers.AddAsync(value);

            var expiryTime = DateTimeOffset.Now.AddSeconds(30);
            _cacheService.SetData<Driver>($"driver{value.Id}", addedObj.Entity, expiryTime);

            await _context.SaveChangesAsync();

            return Ok(addedObj.Entity);
        }

        [HttpDelete("DeleteDriver")]
        public async Task<IActionResult> Delete(int Id)
        {
            var exists = await _context.Drivers.FirstOrDefaultAsync(x => x.Id == Id);

            if(exists != null)
            {
                _context.Remove(exists);
                _cacheService.RemoveData($"driver{Id}");
                _context.SaveChangesAsync();

                return NoContent();
            }

            return NotFound();
        }
    }

    ```
17. Now build the project and then run
    ``` 
    dotnet build
    dotnet run
    ```
18. Now test the apis and check if the data is fetched from the cache or the db. 2 things to remember for testing-
    - run the image for redis in docker
    - run the image for db if db runs in docker also


#### Reference: https://www.youtube.com/watch?v=6HZVu3kGOrg&ab_channel=MohamadLawand

