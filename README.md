# ASPNETCORE 2.2 - API com Entity Framework Core

**Criar as Pasta com o comando abaixo:**

**mkdir** dojo

**cd** dojo

**mkdir** src

**cd** src

    dotnet new sln --name Api

    dotnet new webapi -n Application -o Api.Application --no-https

    dotnet sln add Api.Application

    dotnet build 

**cd** ..

**Abrir o VS-Code**
**Code .**

Ao entrar no VS-Code a ide irá pedir para criar pasta .vscode

	Required assets to build and debug are missing from 'dojo'
	Add them?
			Don't Ask Again     Not Now     Yes

**YES**

Acessar os endereço abaixo:

http://localhost:5000/api/values  
http://localhost:5000/api/values/1

**cd** SRC

**Criar a Camada de Dominio**

    dotnet new classlib -n Domain -f netcoreapp2.2  -o Api.Domain
    dotnet sln add Api.Domain
    dotnet build

**Criar a Camada de Data**

    dotnet new classlib -n Data -f netcoreapp2.2  -o Api.Data
    dotnet sln add Api.Data

**Criar a Camada de Service**

    dotnet new classlib -n Service -f netcoreapp2.2  -o Api.Service
    dotnet sln add Api.Service 


**Referências**

    dotnet add Api.Application reference Api.Domain
    dotnet add Api.Application reference Api.Service

    dotnet add Api.Data reference Api.Domain  
  
    dotnet add Api.Service reference Api.Domain
    dotnet add Api.Service reference Api.Data
	
###### Api.Data - Instalação Pacotes Entity Framework

**Para instalar precisar estar dentro da Pasta Api.Data**

    dotnet add package Microsoft.EntityFrameworkCore.Tools --version 2.2.4
    dotnet add package Microsoft.EntityFrameworkCore.Design --version 2.2.4
    dotnet add package Pomelo.EntityFrameworkCore.MySql --version 2.2.0

    dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 2.2.4
    dotnet add package Microsoft.EntityFrameworkCore.SqlServer.Design --version 1.1.6
	

###### Api.Domain -> Criar a pasta Entities -> 

**BaseEntity.cs**

    using System;
    using System.ComponentModel.DataAnnotations;

    namespace Api.Domain.Entities
    {
        public abstract class BaseEntity
        {
            [Key]
            public Guid Id { get; set; }
            private DateTime? _createAt;
            public DateTime? CreateAt
            {
                get { return _createAt; }
                set { _createAt = (value == null ? DateTime.UtcNow : value); }
            }
            public DateTime? UpdateAt { get; set; }
        }
    }

**UserEntity.cs**

    namespace Api.Domain.Entities
    {
        public class UserEntity : BaseEntity
        {
            public string Name { get; set; }
            public string Email { get; set; }
        }
    }


###### Api.Data
Criar três pastas: 

*.Context
*.Mapping 
*.Repository


###### Api.Data -> Mapping
** UserMap.cs **

	using Api.Domain.Entities;
	using Microsoft.EntityFrameworkCore;
	using Microsoft.EntityFrameworkCore.Metadata.Builders;

	namespace Api.Data.Mapping
	{
		public class UserMap : IEntityTypeConfiguration<UserEntity>
		{
			public void Configure(EntityTypeBuilder<UserEntity> builder)
			{
				builder.ToTable("User");

				builder.HasKey(p => p.Id);

				builder.HasIndex(p => p.Email)
					   .IsUnique();

				builder.Property(c => c.Name)
					.IsRequired()
					.HasMaxLength(60);

				builder.Property(c => c.Email)
					.HasMaxLength(100);

			}
		}
	}


###### Api.Data -> Pasta Contexto
**MyContext.cs**
 
	using Api.Data.Mapping;
	using Api.Domain.Entities;
	using Microsoft.EntityFrameworkCore;

	namespace Api.Data.Context
	{
		public class MyContext : DbContext
		{
			public DbSet<UserEntity> Users { get; set; }
			
			public MyContext(DbContextOptions options) : base(options)
			{

			}

		    protected override void OnModelCreating(ModelBuilder modelBuilder)
		    {
			   base.OnModelCreating(modelBuilder);
			   modelBuilder.Entity<UserEntity> (new UserMap().Configure);
		    }

		}
	}


###### Criar um Arquivo na Raiz do Projeto

**docker-compose.yml**

    version: '3.1'
    services:
    Mysql:
    image: mysql:8.0.16
    restart: always
    environment:
        MYSQL_ROOT_PASSWORD: admin
    ports:
        - 3306:3306
    container_name: mysql8

    MsSql:
    image:  microsoft/mssql-server-linux:2017-latest
    environment:
        MSSQL_SA_PASSWORD: mudar@123
        ACCEPT_EULA: "Y"
    ports:
        - 11433:1433
    container_name: mssql

   
** Para subir o Containers Docker **

    sudo docker-compose up -d
    sudo docker stats

###### Api.Data 
** Acessar a pasta para executar os comando do Entity Framework **
    dotnet ef --version

    dotnet ef --help

    dotnet ef migrations add Initials


*Vai gerar um problema pois não tem a conexao*

**ContextFactory.cs**

	using Microsoft.EntityFrameworkCore;
	using Microsoft.EntityFrameworkCore.Design;

	na Pasta Contexto
	namespace Api.Data.Context
	{
		public class ContextFactory : IDesignTimeDbContextFactory<MyContext>

		{
			public MyContext CreateDbContext(string[] args)
			{
			   //Usado para Criar as Migrações
			   var connectionString ="Server=localhost;Port=3306;Database=dojo;Uid=root;Pwd=admin";
			   var optionsBuilder = new DbContextOptionsBuilder<MyContext>();
			   optionsBuilder.UseMySql(connectionString);
			   return new MyContext(optionsBuilder.Options);            
			}
		}
	}   

**Entity Framework Migrations**

    dotnet ef migrations add Initials

    dotnet ef database update


###### Api.Domain - Implementação da Interface de Repositório

**Criar uma pasta**
*.Interfaces


**IRepository.cs**

	using System;
	using System.Collections.Generic;
	using System.Threading.Tasks;
	using Api.Domain.Entities;

	namespace Api.Domain.Interfaces
	{
		public interface IRepository<T> where T : BaseEntity
		{
		   Task<T> InsertAsync(T item);
		   Task<T> UpdateAsync(T item);
		   Task<bool> DeleteAsync(Guid id);
		   Task<T> SelectAsync(Guid id);
		   Task<IEnumerable<T>> SelectAsync();
		}
	}


###### Api.Data - Repositório
**Repository.cs**

	using System;
	using System.Collections.Generic;
	using System.Threading.Tasks;
	using Api.Data.Context;
	using Api.Domain.Entities;
	using Api.Domain.Interfaces;
	using Microsoft.EntityFrameworkCore;

	namespace Api.Data.Repository
	{
		public class BaseRepository<T> : IRepository<T> where T : BaseEntity
		{
			protected readonly MyContext _context;
			private DbSet<T> _dataset;

			public BaseRepository(MyContext context)
			{
				_context = context;
				_dataset = context.Set<T>();
			}

			public async Task<bool> DeleteAsync(Guid id)
			{
				var result = await _dataset.SingleOrDefaultAsync(p => p.Id.Equals(id));
				try
				{
					if (result != null)
					{
						_dataset.Remove(result);
						await _context.SaveChangesAsync();
						return true;
					}
					else
					{
						return false;
					}
				}
				catch (Exception ex)
				{
					throw ex;
				}
			}

			public async Task<T> InsertAsync(T item)
			{
				try
				{
					if (item.Id == Guid.Empty)
					{
						item.Id = Guid.NewGuid();
					}

					item.CreateAt = DateTime.UtcNow;
					_dataset.Add(item);
					await _context.SaveChangesAsync();
				}
				catch (Exception ex)
				{
					throw ex;
				}
				return item;
			}

			public async Task<T> SelectAsync(Guid id)
			{
				try
				{
					return await _dataset.SingleOrDefaultAsync(p => p.Id.Equals(id));
				}
				catch (Exception ex)
				{
					throw ex;
				}
			}

			public async Task<IEnumerable<T>> SelectAsync()
			{
				try
				{
					return await _dataset.ToListAsync();
				}
				catch (Exception ex)
				{
					throw ex;
				}
			}

			public async Task<T> UpdateAsync(T item)
			{
				item.UpdateAt = DateTime.Now;
				var result = await _dataset.SingleOrDefaultAsync(p => p.Id.Equals(item.Id));

				if (result == null)
					return null;

				item.CreateAt = result.CreateAt;
				try
				{
					_context.Entry(result).CurrentValues.SetValues(item);
					await _context.SaveChangesAsync();
				}
				catch (Exception ex)
				{
					throw ex;
				}
				return item;
			}
		}
	}
	
	
###### Api.Domain - Interface para Services

Dentro da Pasta Interfaces 
Criar uma nova pasta
*.Services

Criar uma Interface

**IUserService.cs**

	using System;
	using System.Collections.Generic;
	using System.Threading.Tasks;
	using Api.Domain.Entities;

	namespace Api.Domain.Interfaces.Services
	{
		public interface IUserService
		{
		   Task<UserEntity> Get(Guid id);
		   Task<IEnumerable<UserEntity>> GetAll();
		   Task<UserEntity> Post(UserEntity user);
		   Task<UserEntity> Put(UserEntity user);
		   Task<bool> Delete(Guid id);           
		}
	}

###### Api.Service - Regras de Negócio

**UserService.cs**
	
	using System;
	using System.Collections.Generic;
	using System.Threading.Tasks;
	using Api.Domain.Entities;
	using Api.Domain.Interfaces;
	using Api.Domain.Interfaces.Services;

	namespace Api.Domain.Services
	{
		public class UserService : IUserService
		{
			private IRepository<UserEntity> _repository;

			public UserService(IRepository<UserEntity> repository)
			{
				_repository = repository;

			}

			public async Task<bool> Delete(Guid id)
			{
				return await _repository.DeleteAsync(id);
			}

			public async Task<UserEntity> Get(Guid id)
			{
				return await _repository.SelectAsync(id);
			}

			public async Task<IEnumerable<UserEntity>> GetAll()
			{
				return await _repository.SelectAsync();
			}

			public async Task<UserEntity> Post(UserEntity user)
			{
				return await _repository.InsertAsync(user);
			}

			public async Task<UserEntity> Put(UserEntity user)
			{
				return await _repository.UpdateAsync(user);
			}
		}
	}
	
###### Api.Application

    using System;
    using System.Net;
    using System.Threading.Tasks;
    using Api.Domain.Interfaces.Services;
    using Microsoft.AspNetCore.Mvc;

    namespace Api.Application.Controllers
    {
        [Route("api/[controller]")]
        [ApiController]
        public class UsersController : ControllerBase
        {

            [HttpGet]
            public async Task<ActionResult> GetAll([FromServices] IUserService service)
            {
                if (!ModelState.IsValid)
                {
                    return BadRequest(ModelState);
                }

                try
                {
                    return Ok(await service.GetAll());
                }
                catch (ArgumentException e)
                {
                    return StatusCode((int)HttpStatusCode.InternalServerError, e.Message);
                }

            }

        }
    }	


http://localhost:5000/api/users

	
**Adicionar em Startup**

        public void ConfigureServices(IServiceCollection services)
        {
		    services.AddDbContext<MyContext>(
                options => options.UseMySql("Server=localhost;Port=3306;Database=dojo;Uid=root;Pwd=admin")
            );
			
            services.AddTransient<IUserService, UserService>();
			
			services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
			
		    ......
        }	
		
		
**Controller Inteira**

    using System;
    using System.Net;
    using System.Threading.Tasks;
    using Api.Domain.Entities;
    using Api.Domain.Interfaces.Services;
    using Microsoft.AspNetCore.Mvc;

    namespace Api.Application.Controllers
    {
        [Route("api/[controller]")]
        [ApiController]
        public class UsersController : ControllerBase
        {

            protected readonly IUserService _service;
            public UsersController(IUserService service)
            {
                _service = service;
            }

            [HttpGet]
            public async Task<ActionResult> GetAll()
            {
                if (!ModelState.IsValid)
                {
                    return BadRequest(ModelState);
                }

                try
                {
                    return Ok(await _service.GetAll());
                }
                catch (ArgumentException e)
                {
                    return StatusCode((int)HttpStatusCode.InternalServerError, e.Message);
                }

            }


            [HttpGet]
            [Route("{id}", Name = "GetWithId")]
            public async Task<ActionResult> Get(Guid id)
            {
                if (!ModelState.IsValid)
                {
                    return BadRequest(ModelState);
                }

                try
                {
                    return Ok(await _service.Get(id));
                }
                catch (ArgumentException e)
                {
                    return StatusCode((int)HttpStatusCode.InternalServerError, e.Message);
                }
            }

            [HttpPost]
            public async Task<ActionResult> Post([FromBody] UserEntity user,
                                                [FromServices] IUserService service)
            {
                if (!ModelState.IsValid)
                {
                    return BadRequest(ModelState);
                }

                try
                {
                    var result = await service.Post(user);
                    if (result != null)
                    {
                        return Created(
                            new Uri(Url.Link("GetWithId", new { id = result.Id })), result);
                    }
                    else
                    {
                        return BadRequest();
                    }
                }
                catch (ArgumentException e)
                {
                    return StatusCode((int)HttpStatusCode.InternalServerError, e.Message);
                }
            }

            [HttpPut]
            public async Task<ActionResult> Put([FromBody] UserEntity user)
            {
                if (!ModelState.IsValid)
                {
                    return BadRequest(ModelState);
                }

                try
                {
                    var result = await _service.Put(user);
                    if (result != null)
                    {
                        return Ok(result);
                    }
                    else
                    {
                        return BadRequest();
                    }
                }
                catch (ArgumentException e)
                {
                    return StatusCode((int)HttpStatusCode.InternalServerError, e.Message);
                }
            }

            [HttpDelete]
            public async Task<ActionResult> Delete(Guid id)
            {
                if (!ModelState.IsValid)
                {
                    return BadRequest(ModelState);
                }

                try
                {
                    return Ok(await _service.Delete(id));
                }
                catch (ArgumentException e)
                {
                    return StatusCode((int)HttpStatusCode.InternalServerError, e.Message);
                }
            }

        }
    }
		

**MySQL**
*. options => options.UseMySql("Server=localhost;Port=3306;Database=dojo;Uid=root;Pwd=admin")

**MS-SQL-Server**
*. options => options.UseSqlServer("Server=127.0.0.1,11433;Database=dojo;User Id=sa;Password=mudar@123")	






