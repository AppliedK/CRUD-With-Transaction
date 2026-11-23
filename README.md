# CRUD-With-Transaction
ASP.NET Core Web API CRUD Operation with Transaction using EF Core

Create application using ASP.NET Core Web API template in Visual Studio 2022.

- Solution name: ApiDemo1
- Project name: ApiDemo1

Note. We create single project for simplicity.

**Create folders structure:**
```
├───Controllers
├───Core
│   ├───Data
│   ├───Domains
│   └───Migrations
├───Properties
└───Services
    └───Models
```

**Install Packages.**
- dotnet add package Microsoft.EntityFrameworkCore -v 8.0.23
- dotnet add package Microsoft.EntityFrameworkCore.Design -v 8.0.23
- dotnet add package Microsoft.EntityFrameworkCore.Tools -v 8.0.23
- dotnet add package Microsoft.EntityFrameworkCore.SqlServer -v 8.0.23
- dotnet add package microsoft.visualstudio.web.codegeneration.design -v 8.0.23

**Update appsettings.json file.**
```
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=DESKTOP-PUI1MDR\\SQLEXPRESS;Database=Pascal;Integrated Security=True;TrustServerCertificate=True;"
  }
}
```
**Create Employee Domain class.**

```
namespace ApiDemo1.Core.Domains
{
    public class Employee
    {
        public int EmployeeId { get; set; }
        public string Name { get; set; }
        public string Phone { get; set; }
        public string Email { get; set; }
        public string Country { get; set; }
        public int Age { get; set; }
        public DateTime CreatedOn { get; set; }
        public DateTime UpdatedOn { get; set; }
    }
}
```

**Create AppDbContext class.**
```
using ApiDemo1.Core.Domains;
using Microsoft.EntityFrameworkCore;

namespace ApiDemo1.Core.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions options) : base(options)
        {
        }
        public DbSet<Employee> Employees { get; set; }
    }
}
```
**Add services to the DI container in Program file.**
```
builder.Services.AddDbContext<AppDbContext>(options => options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```
**Create EmployeeModel model class.**
```
namespace ApiDemo1.Services.Models
{
    public class EmployeeModel
    {
        public string Name { get; set; }
        public string Phone { get; set; }
        public string Email { get; set; }
        public string Country { get; set; }
        public int Age { get; set; }
    }
}
```
**Create AppDbContext class.**
```
using ApiDemo1.Core.Domains;
using Microsoft.EntityFrameworkCore;

namespace ApiDemo1.Core.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions options) : base(options)
        {
        }
        public DbSet<Employee> Employees { get; set; }
    }
}

```

**Create IEmployeeService interface.**
```
using ApiDemo1.Services.Models;

namespace ApiDemo1.Services
{
    public interface IEmployeeService
    {
        Task<List<EmployeeModel>> GetEmployees();
        Task<EmployeeModel?> GetEmployeeById(int id);
        Task<int> CreateEmployee(EmployeeModel employeeModel);
        Task<int> EditEmployee(int id, EmployeeModel employeeModel);
        Task<int> DeleteEmployeeById(int id);
    }
}

```
**Create EmployeeService class.**

```
using ApiDemo1.Core.Data;
using ApiDemo1.Core.Domains;
using ApiDemo1.Services.Models;
using Microsoft.EntityFrameworkCore;

namespace ApiDemo1.Services
{
    public class EmployeeService : IEmployeeService
    {
        private readonly AppDbContext _db;
        public EmployeeService(AppDbContext dbContext)
        {
            _db = dbContext;
        }

        public async Task<int> CreateEmployee(EmployeeModel model)
        {
            using (var transaction = await _db.Database.BeginTransactionAsync())
            {
                try
                {
                    Employee employee = new Employee()
                    {
                        Name = model.Name,
                        Phone = model.Phone,
                        Email = model.Email,
                        Country = model.Country,
                        Age = model.Age,
                        CreatedOn = DateTime.UtcNow,
                        UpdatedOn = DateTime.UtcNow,
                    };
                    _db.Employees.Add(employee);
                    await _db.SaveChangesAsync();
                    transaction.Commit();
                    return employee.EmployeeId;
                }

                catch (Exception)
                {
                    transaction.Rollback();
                    throw;
                }
            }
        }

        public async Task<int> DeleteEmployeeById(int id)
        {
            using (var transaction = await _db.Database.BeginTransactionAsync())
            {
                try
                {
                    var emp = _db.Employees.FirstOrDefault(x => x.EmployeeId == id);

                    if (emp != null)
                    {
                        _db.Remove(emp);
                        await _db.SaveChangesAsync();
                        transaction.Commit();
                        return emp.EmployeeId;
                    }
                }
                catch (Exception)
                {
                    transaction.Rollback();
                    throw;
                }
            }
            return 0;
        }

        public async Task<int> EditEmployee(int id, EmployeeModel model)
        {
            var empRecord = _db.Employees.FirstOrDefault(x => x.EmployeeId == id);

            if (empRecord != null)
            {
                empRecord.Name = model.Name;
                empRecord.Phone = model.Phone;
                empRecord.Email = model.Email;
                empRecord.Country = model.Country;
                empRecord.Age = model.Age;
                empRecord.UpdatedOn = DateTime.UtcNow;
                using (var transaction = await _db.Database.BeginTransactionAsync())
                {
                    try
                    {
                        _db.Employees.Update(empRecord);
                        await _db.SaveChangesAsync();
                        transaction.Commit();
                        return empRecord.EmployeeId;
                    }
                    catch (Exception)
                    {
                        transaction.Rollback();
                        throw;
                    }
                }
            }
            return 0;
        }
        public async Task<EmployeeModel?> GetEmployeeById(int id)
        {
            var emp = await _db.Employees.FirstOrDefaultAsync(x => x.EmployeeId == id);

            if (emp != null)
            {
                EmployeeModel model = new EmployeeModel
                {
                    Name = emp.Name,
                    Phone = emp.Phone,
                    Email = emp.Email,
                    Country = emp.Country,
                    Age = emp.Age
                };
                return model;
            }
            return null;
        }

        public async Task<List<EmployeeModel>> GetEmployees()
        {
            List<EmployeeModel> employeesModel = new List<EmployeeModel>();
            var employees = await _db.Employees.ToListAsync();
            foreach (var emp in employees)
            {
                EmployeeModel model = new EmployeeModel
                {
                    Name = emp.Name,
                    Phone = emp.Phone,
                    Email = emp.Email,
                    Country = emp.Country,
                    Age = emp.Age
                };
                employeesModel.Add(model);
            }
            return employeesModel;
        }
    }
}

```
**Update Program class:**
```
using ApiDemo1.Core.Data;
using ApiDemo1.Services;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.

builder.Services.AddControllers();
builder.Services.AddScoped<IEmployeeService, EmployeeService>();
builder.Services.AddDbContext<AppDbContext>(options => options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

// Configure the HTTP request pipeline.

app.UseAuthorization();

app.MapControllers();

app.Run();

```
**Create EmployeeController class.**
```
using ApiDemo1.Services;
using ApiDemo1.Services.Models;
using Microsoft.AspNetCore.Mvc;

namespace ApiDemo1.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class EmployeesController : ControllerBase
    {
        private readonly IEmployeeService _service;
        public EmployeesController(IEmployeeService service)
        {
            _service = service;
        }
        [HttpGet]
        public async Task<ActionResult<List<EmployeeModel>>> GetEmployees()
        {
            var employees = await _service.GetEmployees();
            if (employees.Count >= 0)
            {
                return Ok(employees);
            }
            else
            {
                return NotFound();
            }

        }
        [HttpGet("{id}", Name = "employeeById")]
        public async Task<ActionResult<EmployeeModel>> GetEmployeeById(int id)
        {
            var employee = await _service.GetEmployeeById(id);
            if (employee != null)
            {
                return Ok(employee);
            }
            else
            {
                return NotFound();
            }
        }

        [HttpDelete]
        public async Task<ActionResult<EmployeeModel>> DeleteEmployeeById(int id)
        {
            var empId = await _service.DeleteEmployeeById(id);
            if (empId > 0)
            {
                return Ok("Employee record was deleted successfully with ID " + empId);
            }
            else
            {
                return BadRequest();
            }
        }

        [HttpPost]
        public async Task<ActionResult<EmployeeModel>> PostEmployee(EmployeeModel model)
        {
            var empId = await _service.CreateEmployee(model);
            if (empId > 0)
            {
                return CreatedAtRoute("employeeById", new { id = empId });
            }
            else
            {
                return BadRequest();
            }
        }

        [HttpPut]
        public async Task<ActionResult<EmployeeModel>> EditEmployee(int id, EmployeeModel model)
        {
            var empId = await _service.EditEmployee(id, model);
            if (empId > 0)
            {
                return Ok("Employee was updated for ID " + empId);
            }
            else
            {
                return BadRequest();
            }
        }
    }
}
```
Add Migrations

```
dotnet ef migrations add First -o ./Core/Migrations
```
Update Database
```
dotnet ef database update
```
Test Data:
```
[
    {
        "name": "Ajeet",
        "phone": "6766112345",
        "email": "akgupta@gmail.com",
        "country": "India",
        "age": 32
    },
    {
        "name": "Rina",
        "phone": "7766112345",
        "email": "rina@gmail.com",
        "country": "India",
        "age": 22
    },
    {
        "name": "Mira",
        "phone": "7766112225",
        "email": "mira@gmail.com",
        "country": "India",
        "age": 24
    },
    {
        "name": "Shyam",
        "phone": "7766552345",
        "email": "shayam@gmail.com",
        "country": "France",
        "age": 43
    }
]
```
