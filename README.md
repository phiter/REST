inicialização back end (abrindo swagger)

    ctrl + shift + b
    inicializar na DevIO.Api
  
inicialização front end ( abrindo aplicação)
  
    npm install
    npm update 
    npm start  
    npm s 
  
soluções para possiveis erro na depedencia do front 

    npm cache clean --force
    npm pkill node
    npm install
  
  
 app em : https://github.com/alfredo1995/SPA7

 <br><br>

criar solution blank como projeto
	
	project type> other > blank solution

blank solution
	
	MinhaApiCompleta
	C:\Users\Alfredo\source\repos
	place solution and project in the same diretory

criar as pastas no setup da aplicação (raiz do projeto contenndo a sln e sln.DotSeting.user)

	.vs
	sql
	src
	teste

dentro da pasta src>

	DevIo.Bunsiness  (camada de negocio)
	DevIo.Data		 (camada de acesso ao dados)

Solution 'MinhaApiCompleta' 

	criar a camada DevIo.Bunsiness
	criar a camada DevIo.Data

DevIo.Bunsiness
	
	cria a pa pasta Models
	
criar a class Fornecedor.cs

	using System;
	using System.Collections.Generic;

	namespace DevIO.Business.Models
	{
		public class Fornecedor : Entity
		{
			public string Nome { get; set; }
			public string Documento { get; set; }
			public TipoFornecedor TipoFornecedor { get; set; }
			public Endereco Endereco { get; set; }
			public bool Ativo { get; set; }

			/* EF Relations */
			public IEnumerable<Produto> Produtos { get; set; }
		}
	}
		
criar a class Endereco.cs

	using System;

	namespace DevIO.Business.Models
	{
		public class Endereco : Entity
		{
			public Guid FornecedorId { get; set; }
			public string Logradouro { get; set; }
			public string Numero { get; set; }
			public string Complemento { get; set; }
			public string Cep { get; set; }
			public string Bairro { get; set; }
			public string Cidade { get; set; }
			public string Estado { get; set; }

			/* EF Relation */
			public Fornecedor Fornecedor { get; set; }
		}
	}

	
Cria a class Produto.cs

	using System;

	namespace DevIO.Business.Models
	{
		public class Produto : Entity
		{
			public Guid FornecedorId { get; set; }

			public string Nome { get; set; }
			public string Descricao { get; set; }
			public string Imagem { get; set; }
			public decimal Valor { get; set; }
			public DateTime DataCadastro { get; set; }
			public bool Ativo { get; set; }

			/* EF Relations */
			public Fornecedor Fornecedor { get; set; }
		}
	}

criar a class TipoFornecedor.cs

	namespace DevIO.Business.Models
	{
		public enum TipoFornecedor
		{
			PessoaFisica = 1,
			PessoaJuridica
		}
	}

Complementando classa Entity.cs

	using System;

	namespace DevIO.Business.Models
	{
		public abstract class Entity
		{
			protected Entity()
			{
				Id = Guid.NewGuid();
			}

			public Guid Id { get; set; }
		}
	}

DevIo.Bunsiness
	
	cria a pa pasta Notificacoes

criar a class Notificacoes.cs

	namespace DevIO.Business.Notificacoes
	{
		public class Notificacao
		{
			public Notificacao(string mensagem)
			{
				Mensagem = mensagem;
			}

			public string Mensagem { get; }
		}
	}

criar a class Notificador.cs

	using System.Collections.Generic;
	using System.Linq;
	using DevIO.Business.Intefaces;

	namespace DevIO.Business.Notificacoes
	{
		public class Notificador : INotificador
		{
			private List<Notificacao> _notificacoes;

			public Notificador()
			{
				_notificacoes = new List<Notificacao>();
			}

			public void Handle(Notificacao notificacao)
			{
				_notificacoes.Add(notificacao);
			}

			public List<Notificacao> ObterNotificacoes()
			{
				return _notificacoes;
			}

			public bool TemNotificacao()
			{
				return _notificacoes.Any();
			}
		}
	}

DevIo.Bunsiness
	
	cria a pa pasta Services

criar a class FornecedorService.cs

		using System;
		using System.Linq;
		using System.Threading.Tasks;
		using DevIO.Business.Intefaces;
		using DevIO.Business.Models;
		using DevIO.Business.Models.Validations;

		namespace DevIO.Business.Services
		{
			public class FornecedorService : BaseService, IFornecedorService
			{
				private readonly IFornecedorRepository _fornecedorRepository;
				private readonly IEnderecoRepository _enderecoRepository;

				public FornecedorService(IFornecedorRepository fornecedorRepository, 
										 IEnderecoRepository enderecoRepository,
										 INotificador notificador) : base(notificador)
				{
					_fornecedorRepository = fornecedorRepository;
					_enderecoRepository = enderecoRepository;
				}

				public async Task<bool> Adicionar(Fornecedor fornecedor)
				{
					if (!ExecutarValidacao(new FornecedorValidation(), fornecedor) 
						|| !ExecutarValidacao(new EnderecoValidation(), fornecedor.Endereco)) return false;

					if (_fornecedorRepository.Buscar(f => f.Documento == fornecedor.Documento).Result.Any())
					{
						Notificar("Já existe um fornecedor com este documento informado.");
						return false;
					}

					await _fornecedorRepository.Adicionar(fornecedor);
					return true;
				}

				public async Task<bool> Atualizar(Fornecedor fornecedor)
				{
					if (!ExecutarValidacao(new FornecedorValidation(), fornecedor)) return false;

					if (_fornecedorRepository.Buscar(f => f.Documento == fornecedor.Documento && f.Id != fornecedor.Id).Result.Any())
					{
						Notificar("Já existe um fornecedor com este documento infomado.");
						return false;
					}

					await _fornecedorRepository.Atualizar(fornecedor);
					return true;
				}

				public async Task AtualizarEndereco(Endereco endereco)
				{
					if (!ExecutarValidacao(new EnderecoValidation(), endereco)) return;

					await _enderecoRepository.Atualizar(endereco);
				}

				public async Task<bool> Remover(Guid id)
				{
					if (_fornecedorRepository.ObterFornecedorProdutosEndereco(id).Result.Produtos.Any())
					{
						Notificar("O fornecedor possui produtos cadastrados!");
						return false;
					}

					var endereco = await _enderecoRepository.ObterEnderecoPorFornecedor(id);

					if (endereco != null)
					{
						await _enderecoRepository.Remover(endereco.Id);
					}

					await _fornecedorRepository.Remover(id);
					return true;
				}

				public void Dispose()
				{
					_fornecedorRepository?.Dispose();
					_enderecoRepository?.Dispose();
				}
			}
		}

criar a class ProdutoService.cs

	using System;
	using System.Threading.Tasks;
	using DevIO.Business.Intefaces;
	using DevIO.Business.Models;
	using DevIO.Business.Models.Validations;

	namespace DevIO.Business.Services
	{
		public class ProdutoService : BaseService, IProdutoService
		{
			private readonly IProdutoRepository _produtoRepository;
			private readonly IUser _user;

			public ProdutoService(IProdutoRepository produtoRepository,
								  INotificador notificador, 
								  IUser user) : base(notificador)
			{
				_produtoRepository = produtoRepository;
				_user = user;
			}

			public async Task Adicionar(Produto produto)
			{
				if (!ExecutarValidacao(new ProdutoValidation(), produto)) return;

				//var user = _user.GetUserId();

				await _produtoRepository.Adicionar(produto);
			}

			public async Task Atualizar(Produto produto)
			{
				if (!ExecutarValidacao(new ProdutoValidation(), produto)) return;

				await _produtoRepository.Atualizar(produto);
			}

			public async Task Remover(Guid id)
			{
				await _produtoRepository.Remover(id);
			}

			public void Dispose()
			{
				_produtoRepository?.Dispose();
			}
		}
	}

criar a class BaseService.cs

	using DevIO.Business.Intefaces;
	using DevIO.Business.Models;
	using DevIO.Business.Notificacoes;
	using FluentValidation;
	using FluentValidation.Results;

	namespace DevIO.Business.Services
	{
		public abstract class BaseService
		{
			private readonly INotificador _notificador;

			protected BaseService(INotificador notificador)
			{
				_notificador = notificador;
			}

			protected void Notificar(ValidationResult validationResult)
			{
				foreach (var error in validationResult.Errors)
				{
					Notificar(error.ErrorMessage);
				}
			}

			protected void Notificar(string mensagem)
			{
				_notificador.Handle(new Notificacao(mensagem));
			}

			protected bool ExecutarValidacao<TV, TE>(TV validacao, TE entidade) where TV : AbstractValidator<TE> where TE : Entity
			{
				var validator = validacao.Validate(entidade);

				if(validator.IsValid) return true;

				Notificar(validator);

				return false;
			}
		}
	}

DevIo.Bunsiness
	
	cria a pa pasta Intefaces

criar a class IEnderecoRepository.cs

	using System;
	using System.Threading.Tasks;
	using DevIO.Business.Models;

	namespace DevIO.Business.Intefaces
	{
		public interface IEnderecoRepository : IRepository<Endereco>
		{
			Task<Endereco> ObterEnderecoPorFornecedor(Guid fornecedorId);
		}
	}

criar a class IFornecedorRepository.cs

	using System;
	using System.Threading.Tasks;
	using DevIO.Business.Models;

	namespace DevIO.Business.Intefaces
	{
		public interface IFornecedorRepository : IRepository<Fornecedor>
		{
			Task<Fornecedor> ObterFornecedorEndereco(Guid id);
			Task<Fornecedor> ObterFornecedorProdutosEndereco(Guid id);
		}
	}

criar a class IFornecedorRepository.cs

	using System;
	using System.Threading.Tasks;
	using DevIO.Business.Models;

	namespace DevIO.Business.Intefaces
	{
		public interface IFornecedorService : IDisposable
		{
			Task<bool> Adicionar(Fornecedor fornecedor);
			Task<bool> Atualizar(Fornecedor fornecedor);
			Task<bool> Remover(Guid id);

			Task AtualizarEndereco(Endereco endereco);
		}
	}

criar a class IProdutoRepository.cs

	using System;
	using System.Collections.Generic;
	using System.Threading.Tasks;
	using DevIO.Business.Models;

	namespace DevIO.Business.Intefaces
	{
		public interface IProdutoRepository : IRepository<Produto>
		{
			Task<IEnumerable<Produto>> ObterProdutosPorFornecedor(Guid fornecedorId);
			Task<IEnumerable<Produto>> ObterProdutosFornecedores();
			Task<Produto> ObterProdutoFornecedor(Guid id);
		}
	}

criar a class IProdutoService.cs

	using System;
	using System.Threading.Tasks;
	using DevIO.Business.Models;

	namespace DevIO.Business.Intefaces
	{
		public interface IProdutoService : IDisposable
		{
			Task Adicionar(Produto produto);
			Task Atualizar(Produto produto);
			Task Remover(Guid id);
		}
	}

criar a class IRepository.cs

	using System;
	using System.Collections.Generic;
	using System.Linq.Expressions;
	using System.Threading.Tasks;
	using DevIO.Business.Models;

	namespace DevIO.Business.Intefaces
	{
		public interface IRepository<TEntity> : IDisposable where TEntity : Entity
		{
			Task Adicionar(TEntity entity);
			Task<TEntity> ObterPorId(Guid id);
			Task<List<TEntity>> ObterTodos();
			Task Atualizar(TEntity entity);
			Task Remover(Guid id);
			Task<IEnumerable<TEntity>> Buscar(Expression<Func<TEntity, bool>> predicate);
			Task<int> SaveChanges();
		}
	}

criar a class IUser.cs

	using System;
	using System.Collections.Generic;
	using System.Security.Claims;

	namespace DevIO.Business.Intefaces
	{
		public interface IUser
		{
			string Name { get; }
			Guid GetUserId();
			string GetUserEmail();
			bool IsAuthenticated();
			bool IsInRole(string role);
			IEnumerable<Claim> GetClaimsIdentity();
		}
	}

criar a class INotificador.cs

	using System.Collections.Generic;
	using DevIO.Business.Notificacoes;

	namespace DevIO.Business.Intefaces
	{
		public interface INotificador
		{
			bool TemNotificacao();
			List<Notificacao> ObterNotificacoes();
			void Handle(Notificacao notificacao);
		}
	}


DevIo.Data
	
	cria a pasta Context

criar a class MeuDbContext.cs

	using System;
	using System.Linq;
	using System.Threading;
	using System.Threading.Tasks;
	using DevIO.Business.Models;
	using Microsoft.EntityFrameworkCore;

	namespace DevIO.Data.Context
	{
		public class MeuDbContext : DbContext
		{
			public MeuDbContext(DbContextOptions<MeuDbContext> options) : base(options) { }

			public DbSet<Produto> Produtos { get; set; }
			public DbSet<Endereco> Enderecos { get; set; }
			public DbSet<Fornecedor> Fornecedores { get; set; }

			protected override void OnModelCreating(ModelBuilder modelBuilder)
			{
				foreach (var property in modelBuilder.Model.GetEntityTypes()
					.SelectMany(e => e.GetProperties()
						.Where(p => p.ClrType == typeof(string))))
					property.SetColumnType("varchar(100)");

				modelBuilder.ApplyConfigurationsFromAssembly(typeof(MeuDbContext).Assembly);

				foreach (var relationship in modelBuilder.Model.GetEntityTypes().SelectMany(e => e.GetForeignKeys())) relationship.DeleteBehavior = DeleteBehavior.ClientSetNull;

				base.OnModelCreating(modelBuilder);
			}

			public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = new CancellationToken())
			{
				foreach (var entry in ChangeTracker.Entries().Where(entry => entry.Entity.GetType().GetProperty("DataCadastro") != null))
				{
					if (entry.State == EntityState.Added)
					{
						entry.Property("DataCadastro").CurrentValue = DateTime.Now;
					}

					if (entry.State == EntityState.Modified)
					{
						entry.Property("DataCadastro").IsModified = false;
					}
				}

				return base.SaveChangesAsync(cancellationToken);
			}
		}
	}


DevIo.Data
	
	cria a pa pasta Repository

criar a class FornecedorRepository.cs

using System;
using System.Threading.Tasks;
using DevIO.Business.Intefaces;
using DevIO.Business.Models;
using DevIO.Data.Context;
using Microsoft.EntityFrameworkCore;

	namespace DevIO.Data.Repository
	{
		public class FornecedorRepository : Repository<Fornecedor>, IFornecedorRepository
		{
			public FornecedorRepository(MeuDbContext context) : base(context)
			{
			}

			public async Task<Fornecedor> ObterFornecedorEndereco(Guid id)
			{
				return await Db.Fornecedores.AsNoTracking()
					.Include(c => c.Endereco)
					.FirstOrDefaultAsync(c => c.Id == id);
			}

			public async Task<Fornecedor> ObterFornecedorProdutosEndereco(Guid id)
			{
				return await Db.Fornecedores.AsNoTracking()
					.Include(c => c.Produtos)
					.Include(c => c.Endereco)
					.FirstOrDefaultAsync(c => c.Id == id);
			}
		}
	}

criar a class EnderecoRepository.cs

	using System;
	using System.Threading.Tasks;
	using DevIO.Business.Intefaces;
	using DevIO.Business.Models;
	using DevIO.Data.Context;
	using Microsoft.EntityFrameworkCore;

	namespace DevIO.Data.Repository
	{
		public class EnderecoRepository : Repository<Endereco>, IEnderecoRepository
		{
			public EnderecoRepository(MeuDbContext context) : base(context) { }

			public async Task<Endereco> ObterEnderecoPorFornecedor(Guid fornecedorId)
			{
				return await Db.Enderecos.AsNoTracking()
					.FirstOrDefaultAsync(f => f.FornecedorId == fornecedorId);
			}
		}
	}

criar a class ProdutoRepository.cs

	using System;
	using System.Collections.Generic;
	using System.Linq;
	using System.Threading.Tasks;
	using DevIO.Business.Intefaces;
	using DevIO.Business.Models;
	using DevIO.Data.Context;
	using Microsoft.EntityFrameworkCore;

	namespace DevIO.Data.Repository
	{
		public class ProdutoRepository : Repository<Produto>, IProdutoRepository
		{
			public ProdutoRepository(MeuDbContext context) : base(context) { }

			public async Task<Produto> ObterProdutoFornecedor(Guid id)
			{
				return await Db.Produtos.AsNoTracking().Include(f => f.Fornecedor)
					.FirstOrDefaultAsync(p => p.Id == id);
			}

			public async Task<IEnumerable<Produto>> ObterProdutosFornecedores()
			{
				return await Db.Produtos.AsNoTracking().Include(f => f.Fornecedor)
					.OrderBy(p => p.Nome).ToListAsync();
			}

			public async Task<IEnumerable<Produto>> ObterProdutosPorFornecedor(Guid fornecedorId)
			{
				return await Buscar(p => p.FornecedorId == fornecedorId);
			}
		}
	}

criar a class Repository.cs

	using System;
	using System.Collections.Generic;
	using System.Linq;
	using System.Linq.Expressions;
	using System.Threading.Tasks;
	using DevIO.Business.Intefaces;
	using DevIO.Business.Models;
	using DevIO.Data.Context;
	using Microsoft.EntityFrameworkCore;

	namespace DevIO.Data.Repository
	{
		public abstract class Repository<TEntity> : IRepository<TEntity> where TEntity : Entity, new()
		{
			protected readonly MeuDbContext Db;
			protected readonly DbSet<TEntity> DbSet;

			protected Repository(MeuDbContext db)
			{
				Db = db;
				DbSet = db.Set<TEntity>();
			}

			public async Task<IEnumerable<TEntity>> Buscar(Expression<Func<TEntity, bool>> predicate)
			{
				return await DbSet.AsNoTracking().Where(predicate).ToListAsync();
			}

			public virtual async Task<TEntity> ObterPorId(Guid id)
			{
				return await DbSet.FindAsync(id);
			}

			public virtual async Task<List<TEntity>> ObterTodos()
			{
				return await DbSet.ToListAsync();
			}

			public virtual async Task Adicionar(TEntity entity)
			{
				DbSet.Add(entity);
				await SaveChanges();
			}

			public virtual async Task Atualizar(TEntity entity)
			{
				DbSet.Update(entity);
				await SaveChanges();
			}

			public virtual async Task Remover(Guid id)
			{
				DbSet.Remove(new TEntity { Id = id });
				await SaveChanges();
			}

			public async Task<int> SaveChanges()
			{
				return await Db.SaveChangesAsync();
			}

			public void Dispose()
			{
				Db?.Dispose();
			}
		}
	}


DevIo.Data
	
	cria a pasta Mapping

criar a class FornecedorMapping.cs

	using DevIO.Business.Models;
	using Microsoft.EntityFrameworkCore;
	using Microsoft.EntityFrameworkCore.Metadata.Builders;

	namespace DevIO.Data.Mappings
	{
		public class FornecedorMapping : IEntityTypeConfiguration<Fornecedor>
		{
			public void Configure(EntityTypeBuilder<Fornecedor> builder)
			{
				builder.HasKey(p => p.Id);

				builder.Property(p => p.Nome)
					.IsRequired()
					.HasColumnType("varchar(200)");

				builder.Property(p => p.Documento)
					.IsRequired()
					.HasColumnType("varchar(14)");

				// 1 : 1 => Fornecedor : Endereco
				builder.HasOne(f => f.Endereco)
					.WithOne(e => e.Fornecedor);

				// 1 : N => Fornecedor : Produtos
				builder.HasMany(f => f.Produtos)
					.WithOne(p => p.Fornecedor)
					.HasForeignKey(p => p.FornecedorId);

				builder.ToTable("Fornecedores");
			}
		}
	}

criar a class ProdutoMapping.cs

	using DevIO.Business.Models;
	using Microsoft.EntityFrameworkCore;
	using Microsoft.EntityFrameworkCore.Metadata.Builders;

	namespace DevIO.Data.Mappings
	{
		public class ProdutoMapping : IEntityTypeConfiguration<Produto>
		{
			public void Configure(EntityTypeBuilder<Produto> builder)
			{
				builder.HasKey(p => p.Id);

				builder.Property(p => p.Nome)
					.IsRequired()
					.HasColumnType("varchar(200)");

				builder.Property(p => p.Descricao)
					.IsRequired()
					.HasColumnType("varchar(1000)");

				builder.Property(p => p.Imagem)
					.IsRequired()
					.HasColumnType("varchar(100)");

				builder.ToTable("Produtos");
			}
		}
	}

criar a class EnderecoMapping.cs

	using DevIO.Business.Models;
	using Microsoft.EntityFrameworkCore;
	using Microsoft.EntityFrameworkCore.Metadata.Builders;

	namespace DevIO.Data.Mappings
	{
		public class EnderecoMapping : IEntityTypeConfiguration<Endereco>
		{
			public void Configure(EntityTypeBuilder<Endereco> builder)
			{
				builder.HasKey(p => p.Id);

				builder.Property(c => c.Logradouro)
					.IsRequired()
					.HasColumnType("varchar(200)");

				builder.Property(c => c.Numero)
					.IsRequired()
					.HasColumnType("varchar(50)");

				builder.Property(c => c.Cep)
					.IsRequired()
					.HasColumnType("varchar(8)");

				builder.Property(c => c.Complemento)
					.HasColumnType("varchar(250)");

				builder.Property(c => c.Bairro)
					.IsRequired()
					.HasColumnType("varchar(100)");

				builder.Property(c => c.Cidade)
					.IsRequired()
					.HasColumnType("varchar(100)");

				builder.Property(c => c.Estado)
					.IsRequired()
					.HasColumnType("varchar(50)");

				builder.ToTable("Enderecos");
			}
		}
	}

DevIo.Data
	
	criar as migration

Habilitando Migrações
	
	intalar o pacote Microsoft.EntityFrameworkCore.Design e Microsoft.EntityFrameworkCore.Tools

	dotnet tool install --global dotnet-ef --version 3.1.5

Comando no Package Nuget Console

	get-help entityframework
	dotnet ef migrations add MeuDbContextModelSnapshot
	dotnet ef migrations add MeuDbContextModelSnapshot -p .\EFCore\EFCore.csproj

	Add-Migration MeuDbContextModelSnapshot
	Update-Database

Aplicando a migração no Banco de Dados

	dotnet ef database update -p .\EFCore\EFCore.csproj -v
	dotnet ef migrations add AdicionarEmail -p .\EFCore\EFCore.csproj -v

se de erro faça manualmente o foco e a API :)

	nesse mesmo github a um repository sobre Entity Framework Core 

exibicao da class MeuDbContextModelSnapshot.cs

	// <auto-generated />
	using System;
	using DevIO.Data.Context;
	using Microsoft.EntityFrameworkCore;
	using Microsoft.EntityFrameworkCore.Infrastructure;
	using Microsoft.EntityFrameworkCore.Metadata;
	using Microsoft.EntityFrameworkCore.Storage.ValueConversion;

	namespace DevIO.Data.Migrations
	{
		[DbContext(typeof(MeuDbContext))]
		partial class MeuDbContextModelSnapshot : ModelSnapshot
		{
			protected override void BuildModel(ModelBuilder modelBuilder)
			{
	#pragma warning disable 612, 618
				modelBuilder
					.HasAnnotation("ProductVersion", "2.2.4-servicing-10062")
					.HasAnnotation("Relational:MaxIdentifierLength", 128)
					.HasAnnotation("SqlServer:ValueGenerationStrategy", SqlServerValueGenerationStrategy.IdentityColumn);

				modelBuilder.Entity("DevIO.Business.Models.Endereco", b =>
					{
						b.Property<Guid>("Id")
							.ValueGeneratedOnAdd();

						b.Property<string>("Bairro")
							.IsRequired()
							.HasColumnType("varchar(100)");

						b.Property<string>("Cep")
							.IsRequired()
							.HasColumnType("varchar(8)");

						b.Property<string>("Cidade")
							.IsRequired()
							.HasColumnType("varchar(100)");

						b.Property<string>("Complemento")
							.HasColumnType("varchar(250)");

						b.Property<string>("Estado")
							.IsRequired()
							.HasColumnType("varchar(50)");

						b.Property<Guid>("FornecedorId");

						b.Property<string>("Logradouro")
							.IsRequired()
							.HasColumnType("varchar(200)");

						b.Property<string>("Numero")
							.IsRequired()
							.HasColumnType("varchar(50)");

						b.HasKey("Id");

						b.HasIndex("FornecedorId")
							.IsUnique();

						b.ToTable("Enderecos");
					});

				modelBuilder.Entity("DevIO.Business.Models.Fornecedor", b =>
					{
						b.Property<Guid>("Id")
							.ValueGeneratedOnAdd();

						b.Property<bool>("Ativo");

						b.Property<string>("Documento")
							.IsRequired()
							.HasColumnType("varchar(14)");

						b.Property<string>("Nome")
							.IsRequired()
							.HasColumnType("varchar(200)");

						b.Property<int>("TipoFornecedor");

						b.HasKey("Id");

						b.ToTable("Fornecedores");
					});

				modelBuilder.Entity("DevIO.Business.Models.Produto", b =>
					{
						b.Property<Guid>("Id")
							.ValueGeneratedOnAdd();

						b.Property<bool>("Ativo");

						b.Property<DateTime>("DataCadastro");

						b.Property<string>("Descricao")
							.IsRequired()
							.HasColumnType("varchar(1000)");

						b.Property<Guid>("FornecedorId");

						b.Property<string>("Imagem")
							.IsRequired()
							.HasColumnType("varchar(100)");

						b.Property<string>("Nome")
							.IsRequired()
							.HasColumnType("varchar(200)");

						b.Property<decimal>("Valor");

						b.HasKey("Id");

						b.HasIndex("FornecedorId");

						b.ToTable("Produtos");
					});

				modelBuilder.Entity("DevIO.Business.Models.Endereco", b =>
					{
						b.HasOne("DevIO.Business.Models.Fornecedor", "Fornecedor")
							.WithOne("Endereco")
							.HasForeignKey("DevIO.Business.Models.Endereco", "FornecedorId");
					});

				modelBuilder.Entity("DevIO.Business.Models.Produto", b =>
					{
						b.HasOne("DevIO.Business.Models.Fornecedor", "Fornecedor")
							.WithMany("Produtos")
							.HasForeignKey("FornecedorId");
					});
	#pragma warning restore 612, 618
			}
		}
	}

criar a class 20190503042709_Initial.cs

	using System;
	using Microsoft.EntityFrameworkCore.Migrations;

	namespace DevIO.Data.Migrations
	{
		public partial class Initial : Migration
		{
			protected override void Up(MigrationBuilder migrationBuilder)
			{
				migrationBuilder.CreateTable(
					name: "Fornecedores",
					columns: table => new
					{
						Id = table.Column<Guid>(nullable: false),
						Nome = table.Column<string>(type: "varchar(200)", nullable: false),
						Documento = table.Column<string>(type: "varchar(14)", nullable: false),
						TipoFornecedor = table.Column<int>(nullable: false),
						Ativo = table.Column<bool>(nullable: false)
					},
					constraints: table =>
					{
						table.PrimaryKey("PK_Fornecedores", x => x.Id);
					});

				migrationBuilder.CreateTable(
					name: "Enderecos",
					columns: table => new
					{
						Id = table.Column<Guid>(nullable: false),
						FornecedorId = table.Column<Guid>(nullable: false),
						Logradouro = table.Column<string>(type: "varchar(200)", nullable: false),
						Numero = table.Column<string>(type: "varchar(50)", nullable: false),
						Complemento = table.Column<string>(type: "varchar(250)", nullable: true),
						Cep = table.Column<string>(type: "varchar(8)", nullable: false),
						Bairro = table.Column<string>(type: "varchar(100)", nullable: false),
						Cidade = table.Column<string>(type: "varchar(100)", nullable: false),
						Estado = table.Column<string>(type: "varchar(50)", nullable: false)
					},
					constraints: table =>
					{
						table.PrimaryKey("PK_Enderecos", x => x.Id);
						table.ForeignKey(
							name: "FK_Enderecos_Fornecedores_FornecedorId",
							column: x => x.FornecedorId,
							principalTable: "Fornecedores",
							principalColumn: "Id",
							onDelete: ReferentialAction.Restrict);
					});

				migrationBuilder.CreateTable(
					name: "Produtos",
					columns: table => new
					{
						Id = table.Column<Guid>(nullable: false),
						FornecedorId = table.Column<Guid>(nullable: false),
						Nome = table.Column<string>(type: "varchar(200)", nullable: false),
						Descricao = table.Column<string>(type: "varchar(1000)", nullable: false),
						Imagem = table.Column<string>(type: "varchar(100)", nullable: false),
						Valor = table.Column<decimal>(nullable: false),
						DataCadastro = table.Column<DateTime>(nullable: false),
						Ativo = table.Column<bool>(nullable: false)
					},
					constraints: table =>
					{
						table.PrimaryKey("PK_Produtos", x => x.Id);
						table.ForeignKey(
							name: "FK_Produtos_Fornecedores_FornecedorId",
							column: x => x.FornecedorId,
							principalTable: "Fornecedores",
							principalColumn: "Id",
							onDelete: ReferentialAction.Restrict);
					});

				migrationBuilder.CreateIndex(
					name: "IX_Enderecos_FornecedorId",
					table: "Enderecos",
					column: "FornecedorId",
					unique: true);

				migrationBuilder.CreateIndex(
					name: "IX_Produtos_FornecedorId",
					table: "Produtos",
					column: "FornecedorId");
			}

			protected override void Down(MigrationBuilder migrationBuilder)
			{
				migrationBuilder.DropTable(
					name: "Enderecos");

				migrationBuilder.DropTable(
					name: "Produtos");

				migrationBuilder.DropTable(
					name: "Fornecedores");
			}
		}
	}

20200507181203_inicial.cs

	using Microsoft.EntityFrameworkCore.Migrations;

	namespace DevIO.Data.Migrations
	{
		public partial class inicial : Migration
		{
			protected override void Up(MigrationBuilder migrationBuilder)
			{

			}

			protected override void Down(MigrationBuilder migrationBuilder)
			{

			}
		}
	}

 <br><br>

Agora vamos focar na Criação da camada de API

	A camada de acesso a dados e negocio ate entao foram feita surpeficialmente visando o foco na camada da aplicação
