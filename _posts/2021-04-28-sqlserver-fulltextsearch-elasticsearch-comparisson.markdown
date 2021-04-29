---
layout: post
title:  "SQL Server Full-Text Search and Elasticsearch comparison"
---

# Environment

1. Hardware
  * 4GB memory
2. Software
  * Windows 10 Enterprise 1903 build number
  * Microsoft SQL Server 2019 (RTM) - 15.0.2000.5 (X64)
Sep 24 2019 13:48:23
Copyright (C) 2019 Microsoft Corporation
Developer Edition (64-bit) on Windows 10 Enterprise 10.0 <X64> (Build 18362: ) (Hypervisor) 
  * elasticsearch 7.12.0 Basic
3. SDK
  * [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet)
  * Entity framework core
  * [NEST](https://github.com/elastic/elasticsearch-net)
  * [Fake data generator Bogus](https://github.com/bchavez/Bogus)

# SQL Server setup

### Limit sql server memory

``` sql
sp_configure 'show advanced options', 1;
GO

RECONFIGURE;
GO

exec sp_configure 'max server memory', 4096;
GO

RECONFIGURE;
GO
```

### Code first

Connection string
```
Server=(local);Database=Benchmark;Trusted_Connection=True
```

Create database
``` bash
dotnet ef migrations add Init --project ElasticsearchSqlServerBenchmark/ElasticsearchSqlServerBenchmark.csproj
dotnet ef database update --project ElasticsearchSqlServerBenchmark/ElasticsearchSqlServerBenchmark.csproj
```

### Raw SQL to setup Full-Text index
``` sql
CREATE FULLTEXT CATALOG BenchmarkFTCalatog AS DEFAULT;
GO

CREATE FULLTEXT INDEX ON dbo.Entities(
	NullString
	,NonNullString
	,ShortString
	,AverageString)
	KEY INDEX PK_Entities
		ON BenchmarkFTCalatog;
GO
```

# Benchmarks

Different character lengths. Columns:

1. Null string NVARCHAR(1000)
2. Non-null string NVARCHAR(1000)
3. Short string NVARCHAR(10)
6. Average string NVARCHAR(250)

``` csharp
public class Entity
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public Guid BookId { get; set; }

    [MaxLength(1000)]
    public string NullString { get; set; }

    [MaxLength(1000)]
    [Required]
    public string NonNullString { get; set; }

    [MaxLength(10)]
    [Required]
    public string ShortString { get; set; }

    [MaxLength(250)]
    [Required]
    public string AverageString { get; set; }

    [Required]
    public DateTimeOffset Date { get; set; }

    [Required]
    public decimal Amount { get; set; }
}
```

Benchmark runner:
``` csharp
[SimpleJob(RunStrategy.ColdStart)]
[MinColumn, MaxColumn]
public class EsVsSqlServer
{
    private Lorem lorem = new Lorem("ru");
    private Random rng = new Random();
    private ElasticClient elasticClient = new ElasticClient(new ConnectionSettings(new Uri("http://127.0.0.1:9200")));
    private const string indexName = "entities";
    private BenchmarkContext db;

    [Params(
        10,
        1000,
        100 * 1000
    // 1000 * 1000 - takes a whole day for single benchmark
    // 1000 * 1000 * 1000
    )]
    public int N { get; set; }

    [GlobalSetup]
    public void GlobalStartup()
    {
        db = new BenchmarkContext();
        if (db.Entities.Count() >= N)
            return;

        Console.WriteLine($"N={N}-----GlobalSetup");
        db.Database.ExecuteSqlRaw("TRUNCATE TABLE dbo.Entities");
        elasticClient.Indices.Delete(indexName);
        var autoDetectChangesEnabled = db.ChangeTracker.AutoDetectChangesEnabled;
        try
        {
            db.ChangeTracker.AutoDetectChangesEnabled = false;
            const int CREATE_BULK_CHUNK_SIZE = 10000;
            for (int skip = 0; skip < N; skip += CREATE_BULK_CHUNK_SIZE)
            {
                var count = Math.Min(CREATE_BULK_CHUNK_SIZE, N - skip);
                var data = GenerateEntities(count);
                db.Entities.AddRange(data);
                db.SaveChanges();
                elasticClient.Index(data, d => d.Index(indexName));
            }
        }
        finally
        {
            db.SaveChanges();
            db.ChangeTracker.AutoDetectChangesEnabled = autoDetectChangesEnabled;
        }
    }

    [GlobalCleanup]
    public void GlobalCleanup()
    {
        db.Dispose();
    }

    [Benchmark]
    public Entity SqlServerContains_250()
    {
        var word = lorem.Word();
        var entity = db.Entities
            .FromSqlRaw(
                $"SELECT * FROM dbo.Entities WHERE CONTAINS({nameof(Entity.AverageString)}, N'{word}')")
            .FirstOrDefault();

        return entity;
    }

    [Benchmark]
    public Entity SqlServerContains_1000_Nullable()
    {
        var word = lorem.Word();
        var entity = db.Entities
            .FromSqlRaw(
                $"SELECT * FROM dbo.Entities WHERE CONTAINS({nameof(Entity.NullString)}, N'{word}')")
            .FirstOrDefault();

        return entity;
    }

    [Benchmark]
    public Entity SqlServerContains_1000()
    {
        var word = lorem.Word();
        var entity = db.Entities
            .FromSqlRaw(
                $"SELECT * FROM dbo.Entities WHERE CONTAINS({nameof(Entity.NonNullString)}, N'{word}')")
            .FirstOrDefault();

        return entity;
    }

    [Benchmark]
    public Entity SqlServerContains_10()
    {
        var word = lorem.Word();
        var entity = db.Entities
            .FromSqlRaw(
                $"SELECT * FROM dbo.Entities WHERE CONTAINS({nameof(Entity.ShortString)}, N'{word}')")
            .FirstOrDefault();

        return entity;
    }

    [Benchmark]
    public Entity ElasticsearchMatch_250()
    {
        var word = lorem.Word();
        var entity = elasticClient.Search<Entity>(search => search
            .Index(indexName)
            .Query(q => q
                .Match(mq => mq.Field(e => e.AverageString).Query(word))))
            .Hits.Select(x => x.Source)
            .FirstOrDefault();

        return entity;
    }

    [Benchmark]
    public Entity ElasticsearchMatch_1000_Nullable()
    {
        var word = lorem.Word();
        var entity = elasticClient.Search<Entity>(search => search
            .Index(indexName)
            .Query(q => q
                .Match(mq => mq.Field(e => e.NullString).Query(word))))
            .Hits.Select(x => x.Source)
            .FirstOrDefault();

        return entity;
    }

    [Benchmark]
    public Entity ElasticsearchMatch_1000()
    {
        var word = lorem.Word();
        var entity = elasticClient.Search<Entity>(search => search
            .Index(indexName)
            .Query(q => q
                .Match(mq => mq.Field(e => e.NonNullString).Query(word))))
            .Hits.Select(x => x.Source)
            .FirstOrDefault();

        return entity;
    }

    [Benchmark]
    public Entity ElasticsearchMatch_10()
    {
        var word = lorem.Word();
        var entity = elasticClient.Search<Entity>(search => search
            .Index(indexName)
            .Query(q => q
                .Match(mq => mq.Field(e => e.ShortString).Query(word))))
            .Hits.Select(x => x.Source)
            .FirstOrDefault();

        return entity;
    }

    private Entity[] GenerateEntities(int n)
    {
        return Enumerable
            .Range(0, n)
            .Select(x =>
            {
                var text = lorem.Sentence(250);
                var _1000 = text.Substring(0, 1000);
                var _10 = text.Substring(0, 10);
                var _250 = text.Substring(0, 250);

                return new Entity
                {
                    BookId = Guid.NewGuid(),
                    ShortString = _10,
                    NullString = (rng.Next() % 2 == 0) ? (string)null : _1000,
                    NonNullString = _1000,
                    AverageString = _250,
                    Amount = rng.Next(),
                    Date = DateTimeOffset.UtcNow
                };
            })
            .ToArray();
    }
}
```

# Results

``` ini

BenchmarkDotNet=v0.12.1, OS=Windows 10.0.18362.1256 (1903/May2019Update/19H1)
Intel Core i7-6700 CPU 3.40GHz (Skylake), 1 CPU, 8 logical and 4 physical cores
.NET Core SDK=3.1.407
  [Host]     : .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT
  Job-JJCSWT : .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT

RunStrategy=ColdStart  

```
|                           Method |      N |      Mean |     Error |     StdDev |   Median |       Min |         Max |
|--------------------------------- |------- |----------:|----------:|-----------:|---------:|----------:|------------:|
|            **SqlServerContains_250** |     **10** |  **3.478 ms** |  **1.417 ms** |   **4.178 ms** | **2.921 ms** | **0.6198 ms** |    **42.21 ms** |
|  SqlServerContains_1000_Nullable |     10 |  4.586 ms |  4.874 ms |  14.371 ms | 2.997 ms | 0.7489 ms |   145.99 ms |
|           SqlServerContains_1000 |     10 |  4.457 ms |  4.529 ms |  13.355 ms | 3.050 ms | 0.7292 ms |   135.75 ms |
|             SqlServerContains_10 |     10 |  4.260 ms |  3.369 ms |   9.934 ms | 2.784 ms | 0.6755 ms |    79.82 ms |
|           ElasticsearchMatch_250 |     10 |  5.642 ms | 12.971 ms |  38.247 ms | 1.673 ms | 0.8903 ms |   384.20 ms |
| ElasticsearchMatch_1000_Nullable |     10 |  5.658 ms | 12.821 ms |  37.803 ms | 1.521 ms | 0.8079 ms |   379.77 ms |
|          ElasticsearchMatch_1000 |     10 |  5.654 ms | 12.788 ms |  37.706 ms | 1.727 ms | 0.8111 ms |   378.84 ms |
|            ElasticsearchMatch_10 |     10 |  5.558 ms | 12.979 ms |  38.269 ms | 1.472 ms | 0.8153 ms |   384.33 ms |
|            **SqlServerContains_250** |   **1000** |  **3.504 ms** |  **1.849 ms** |   **5.451 ms** | **2.954 ms** | **0.5868 ms** |    **55.09 ms** |
|  SqlServerContains_1000_Nullable |   1000 |  4.404 ms |  4.476 ms |  13.198 ms | 2.906 ms | 0.6928 ms |   134.09 ms |
|           SqlServerContains_1000 |   1000 |  4.512 ms |  4.470 ms |  13.181 ms | 3.049 ms | 0.7372 ms |   133.83 ms |
|             SqlServerContains_10 |   1000 |  4.392 ms |  4.409 ms |  13.000 ms | 2.979 ms | 0.7309 ms |   131.97 ms |
|           ElasticsearchMatch_250 |   1000 |  5.799 ms | 13.322 ms |  39.280 ms | 1.710 ms | 0.9348 ms |   394.56 ms |
| ElasticsearchMatch_1000_Nullable |   1000 |  5.490 ms | 13.140 ms |  38.744 ms | 1.284 ms | 0.8597 ms |   388.97 ms |
|          ElasticsearchMatch_1000 |   1000 |  5.626 ms | 12.905 ms |  38.049 ms | 1.430 ms | 0.8736 ms |   382.17 ms |
|            ElasticsearchMatch_10 |   1000 |  5.705 ms | 13.195 ms |  38.905 ms | 1.568 ms | 0.8949 ms |   390.76 ms |
|            **SqlServerContains_250** | **100000** | **27.463 ms** | **63.280 ms** | **186.581 ms** | **3.391 ms** | **0.8667 ms** | **1,762.45 ms** |
|  SqlServerContains_1000_Nullable | 100000 |  4.567 ms |  4.599 ms |  13.561 ms | 3.036 ms | 0.7599 ms |   137.84 ms |
|           SqlServerContains_1000 | 100000 |  4.286 ms |  4.715 ms |  13.903 ms | 2.989 ms | 0.7403 ms |   140.98 ms |
|             SqlServerContains_10 | 100000 |  4.509 ms |  3.300 ms |   9.731 ms | 3.006 ms | 0.7464 ms |    78.98 ms |
|           ElasticsearchMatch_250 | 100000 |  6.321 ms | 15.380 ms |  45.348 ms | 1.315 ms | 0.8325 ms |   455.14 ms |
| ElasticsearchMatch_1000_Nullable | 100000 |  5.589 ms | 13.104 ms |  38.637 ms | 1.438 ms | 0.8653 ms |   387.99 ms |
|          ElasticsearchMatch_1000 | 100000 |  5.590 ms | 13.012 ms |  38.365 ms | 1.312 ms | 0.8760 ms |   385.26 ms |
|            ElasticsearchMatch_10 | 100000 |  5.498 ms | 13.403 ms |  39.519 ms | 1.253 ms | 0.8845 ms |   396.67 ms |

### Histograms

``` ini
EsVsSqlServer.SqlServerContains_250: Job-JJCSWT(RunStrategy=ColdStart) [N=10]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 3.478 ms, StdErr = 0.418 ms (12.01%), N = 100, StdDev = 4.178 ms
Min = 0.620 ms, Q1 = 2.631 ms, Median = 2.921 ms, Q3 = 3.506 ms, Max = 42.215 ms
IQR = 0.875 ms, LowerFence = 1.319 ms, UpperFence = 4.818 ms
ConfidenceInterval = [2.061 ms; 4.895 ms] (CI 99.9%), Margin = 1.417 ms (40.74% of Mean)
Skewness = 7.99, Kurtosis = 73.98, MValue = 2.55
-------------------- Histogram --------------------
[-0.562 ms ;  1.922 ms) | @@@@@@@@@@@@@@@
[ 1.922 ms ;  4.285 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 4.285 ms ;  7.199 ms) | @@@@@@@@
[ 7.199 ms ;  9.596 ms) | @@
[ 9.596 ms ; 11.959 ms) |
[11.959 ms ; 14.322 ms) |
[14.322 ms ; 16.685 ms) |
[16.685 ms ; 19.047 ms) |
[19.047 ms ; 21.410 ms) |
[21.410 ms ; 23.773 ms) |
[23.773 ms ; 26.136 ms) |
[26.136 ms ; 28.498 ms) |
[28.498 ms ; 30.861 ms) |
[30.861 ms ; 33.224 ms) |
[33.224 ms ; 35.587 ms) |
[35.587 ms ; 37.950 ms) |
[37.950 ms ; 41.033 ms) |
[41.033 ms ; 43.396 ms) | @
---------------------------------------------------

EsVsSqlServer.SqlServerContains_1000_Nullable: Job-JJCSWT(RunStrategy=ColdStart) [N=10]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 4.586 ms, StdErr = 1.437 ms (31.34%), N = 100, StdDev = 14.371 ms
Min = 0.749 ms, Q1 = 2.632 ms, Median = 2.997 ms, Q3 = 3.900 ms, Max = 145.989 ms
IQR = 1.267 ms, LowerFence = 0.732 ms, UpperFence = 5.801 ms
ConfidenceInterval = [-0.288 ms; 9.460 ms] (CI 99.9%), Margin = 4.874 ms (106.28% of Mean)
Skewness = 9.52, Kurtosis = 93.73, MValue = 3.65
-------------------- Histogram --------------------
[  0.256 ms ;   8.384 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[  8.384 ms ;  16.511 ms) |
[ 16.511 ms ;  24.639 ms) |
[ 24.639 ms ;  32.766 ms) |
[ 32.766 ms ;  40.894 ms) |
[ 40.894 ms ;  49.021 ms) |
[ 49.021 ms ;  57.149 ms) |
[ 57.149 ms ;  65.276 ms) |
[ 65.276 ms ;  73.403 ms) |
[ 73.403 ms ;  81.531 ms) |
[ 81.531 ms ;  89.658 ms) |
[ 89.658 ms ;  97.786 ms) |
[ 97.786 ms ; 105.913 ms) |
[105.913 ms ; 114.041 ms) |
[114.041 ms ; 122.168 ms) |
[122.168 ms ; 130.295 ms) |
[130.295 ms ; 138.423 ms) |
[138.423 ms ; 141.926 ms) |
[141.926 ms ; 150.053 ms) | @
---------------------------------------------------

EsVsSqlServer.SqlServerContains_1000: Job-JJCSWT(RunStrategy=ColdStart) [N=10]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 4.457 ms, StdErr = 1.336 ms (29.97%), N = 100, StdDev = 13.355 ms
Min = 0.729 ms, Q1 = 2.735 ms, Median = 3.050 ms, Q3 = 3.683 ms, Max = 135.747 ms
IQR = 0.947 ms, LowerFence = 1.314 ms, UpperFence = 5.104 ms
ConfidenceInterval = [-0.073 ms; 8.986 ms] (CI 99.9%), Margin = 4.529 ms (101.63% of Mean)
Skewness = 9.5, Kurtosis = 93.4, MValue = 3
-------------------- Histogram --------------------
[  0.494 ms ;   8.047 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[  8.047 ms ;  13.143 ms) | @
[ 13.143 ms ;  20.696 ms) |
[ 20.696 ms ;  28.248 ms) |
[ 28.248 ms ;  35.801 ms) |
[ 35.801 ms ;  43.354 ms) |
[ 43.354 ms ;  50.907 ms) |
[ 50.907 ms ;  58.459 ms) |
[ 58.459 ms ;  66.012 ms) |
[ 66.012 ms ;  73.565 ms) |
[ 73.565 ms ;  81.118 ms) |
[ 81.118 ms ;  88.671 ms) |
[ 88.671 ms ;  96.223 ms) |
[ 96.223 ms ; 103.776 ms) |
[103.776 ms ; 111.329 ms) |
[111.329 ms ; 118.882 ms) |
[118.882 ms ; 126.434 ms) |
[126.434 ms ; 131.970 ms) |
[131.970 ms ; 139.523 ms) | @
---------------------------------------------------

EsVsSqlServer.SqlServerContains_10: Job-JJCSWT(RunStrategy=ColdStart) [N=10]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 4.260 ms, StdErr = 0.993 ms (23.32%), N = 100, StdDev = 9.934 ms
Min = 0.675 ms, Q1 = 2.216 ms, Median = 2.784 ms, Q3 = 3.215 ms, Max = 79.817 ms
IQR = 0.999 ms, LowerFence = 0.717 ms, UpperFence = 4.713 ms
ConfidenceInterval = [0.891 ms; 7.629 ms] (CI 99.9%), Margin = 3.369 ms (79.08% of Mean)
Skewness = 6.58, Kurtosis = 46.67, MValue = 3.02
-------------------- Histogram --------------------
[ 0.668 ms ;  6.286 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 6.286 ms ; 11.627 ms) | @@@
[11.627 ms ; 17.245 ms) |
[17.245 ms ; 22.863 ms) |
[22.863 ms ; 28.481 ms) |
[28.481 ms ; 34.099 ms) |
[34.099 ms ; 39.716 ms) |
[39.716 ms ; 45.334 ms) |
[45.334 ms ; 50.952 ms) |
[50.952 ms ; 56.570 ms) |
[56.570 ms ; 61.322 ms) |
[61.322 ms ; 66.940 ms) | @
[66.940 ms ; 72.558 ms) |
[72.558 ms ; 77.008 ms) |
[77.008 ms ; 82.626 ms) | @
---------------------------------------------------

EsVsSqlServer.ElasticsearchMatch_250: Job-JJCSWT(RunStrategy=ColdStart) [N=10]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 5.642 ms, StdErr = 3.825 ms (67.79%), N = 100, StdDev = 38.247 ms
Min = 0.890 ms, Q1 = 1.118 ms, Median = 1.673 ms, Q3 = 2.453 ms, Max = 384.198 ms
IQR = 1.335 ms, LowerFence = -0.884 ms, UpperFence = 4.455 ms
ConfidenceInterval = [-7.330 ms; 18.613 ms] (CI 99.9%), Margin = 12.971 ms (229.92% of Mean)
Skewness = 9.7, Kurtosis = 95.97, MValue = 3.67
-------------------- Histogram --------------------
[ -9.925 ms ;  13.545 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 13.545 ms ;  35.175 ms) |
[ 35.175 ms ;  56.805 ms) |
[ 56.805 ms ;  78.435 ms) |
[ 78.435 ms ; 100.065 ms) |
[100.065 ms ; 121.694 ms) |
[121.694 ms ; 143.324 ms) |
[143.324 ms ; 164.954 ms) |
[164.954 ms ; 186.584 ms) |
[186.584 ms ; 208.214 ms) |
[208.214 ms ; 229.844 ms) |
[229.844 ms ; 251.474 ms) |
[251.474 ms ; 273.104 ms) |
[273.104 ms ; 294.734 ms) |
[294.734 ms ; 316.364 ms) |
[316.364 ms ; 337.993 ms) |
[337.993 ms ; 359.623 ms) |
[359.623 ms ; 373.383 ms) |
[373.383 ms ; 395.013 ms) | @
---------------------------------------------------

EsVsSqlServer.ElasticsearchMatch_1000_Nullable: Job-JJCSWT(RunStrategy=ColdStart) [N=10]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 5.658 ms, StdErr = 3.780 ms (66.81%), N = 100, StdDev = 37.803 ms
Min = 0.808 ms, Q1 = 1.110 ms, Median = 1.521 ms, Q3 = 2.395 ms, Max = 379.767 ms
IQR = 1.285 ms, LowerFence = -0.818 ms, UpperFence = 4.323 ms
ConfidenceInterval = [-7.163 ms; 18.479 ms] (CI 99.9%), Margin = 12.821 ms (226.59% of Mean)
Skewness = 9.69, Kurtosis = 95.92, MValue = 2.55
-------------------- Histogram --------------------
[ -6.657 ms ;  14.722 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 14.722 ms ;  36.101 ms) |
[ 36.101 ms ;  57.480 ms) |
[ 57.480 ms ;  78.859 ms) |
[ 78.859 ms ; 100.237 ms) |
[100.237 ms ; 121.616 ms) |
[121.616 ms ; 142.995 ms) |
[142.995 ms ; 164.374 ms) |
[164.374 ms ; 185.753 ms) |
[185.753 ms ; 207.132 ms) |
[207.132 ms ; 228.511 ms) |
[228.511 ms ; 249.889 ms) |
[249.889 ms ; 271.268 ms) |
[271.268 ms ; 292.647 ms) |
[292.647 ms ; 314.026 ms) |
[314.026 ms ; 335.405 ms) |
[335.405 ms ; 356.784 ms) |
[356.784 ms ; 369.077 ms) |
[369.077 ms ; 390.456 ms) | @
---------------------------------------------------

EsVsSqlServer.ElasticsearchMatch_1000: Job-JJCSWT(RunStrategy=ColdStart) [N=10]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 5.654 ms, StdErr = 3.771 ms (66.68%), N = 100, StdDev = 37.706 ms
Min = 0.811 ms, Q1 = 1.200 ms, Median = 1.727 ms, Q3 = 2.353 ms, Max = 378.844 ms
IQR = 1.153 ms, LowerFence = -0.529 ms, UpperFence = 4.082 ms
ConfidenceInterval = [-7.134 ms; 18.443 ms] (CI 99.9%), Margin = 12.788 ms (226.16% of Mean)
Skewness = 9.69, Kurtosis = 95.95, MValue = 3.28
-------------------- Histogram --------------------
[ -7.013 ms ;  14.312 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 14.312 ms ;  35.636 ms) |
[ 35.636 ms ;  56.961 ms) |
[ 56.961 ms ;  78.285 ms) |
[ 78.285 ms ;  99.610 ms) |
[ 99.610 ms ; 120.934 ms) |
[120.934 ms ; 142.259 ms) |
[142.259 ms ; 163.583 ms) |
[163.583 ms ; 184.908 ms) |
[184.908 ms ; 206.232 ms) |
[206.232 ms ; 227.557 ms) |
[227.557 ms ; 248.881 ms) |
[248.881 ms ; 270.206 ms) |
[270.206 ms ; 291.530 ms) |
[291.530 ms ; 312.855 ms) |
[312.855 ms ; 334.179 ms) |
[334.179 ms ; 355.504 ms) |
[355.504 ms ; 368.182 ms) |
[368.182 ms ; 389.507 ms) | @
---------------------------------------------------

EsVsSqlServer.ElasticsearchMatch_10: Job-JJCSWT(RunStrategy=ColdStart) [N=10]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 5.558 ms, StdErr = 3.827 ms (68.86%), N = 100, StdDev = 38.269 ms
Min = 0.815 ms, Q1 = 1.070 ms, Median = 1.472 ms, Q3 = 2.203 ms, Max = 384.330 ms
IQR = 1.133 ms, LowerFence = -0.630 ms, UpperFence = 3.903 ms
ConfidenceInterval = [-7.421 ms; 18.537 ms] (CI 99.9%), Margin = 12.979 ms (233.53% of Mean)
Skewness = 9.69, Kurtosis = 95.96, MValue = 2.47
-------------------- Histogram --------------------
[ -7.471 ms ;  14.172 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 14.172 ms ;  35.815 ms) |
[ 35.815 ms ;  57.457 ms) |
[ 57.457 ms ;  79.100 ms) |
[ 79.100 ms ; 100.743 ms) |
[100.743 ms ; 122.386 ms) |
[122.386 ms ; 144.028 ms) |
[144.028 ms ; 165.671 ms) |
[165.671 ms ; 187.314 ms) |
[187.314 ms ; 208.957 ms) |
[208.957 ms ; 230.600 ms) |
[230.600 ms ; 252.242 ms) |
[252.242 ms ; 273.885 ms) |
[273.885 ms ; 295.528 ms) |
[295.528 ms ; 317.171 ms) |
[317.171 ms ; 338.813 ms) |
[338.813 ms ; 360.456 ms) |
[360.456 ms ; 373.508 ms) |
[373.508 ms ; 395.151 ms) | @
---------------------------------------------------

EsVsSqlServer.SqlServerContains_250: Job-JJCSWT(RunStrategy=ColdStart) [N=1000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 3.504 ms, StdErr = 0.545 ms (15.55%), N = 100, StdDev = 5.451 ms
Min = 0.587 ms, Q1 = 2.509 ms, Median = 2.954 ms, Q3 = 3.645 ms, Max = 55.094 ms
IQR = 1.136 ms, LowerFence = 0.805 ms, UpperFence = 5.349 ms
ConfidenceInterval = [1.656 ms; 5.353 ms] (CI 99.9%), Margin = 1.849 ms (52.75% of Mean)
Skewness = 8.5, Kurtosis = 80.32, MValue = 2.65
-------------------- Histogram --------------------
[ 0.576 ms ;  3.658 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 3.658 ms ;  6.806 ms) | @@@@@@@@@@@@@@@@@@@@@@
[ 6.806 ms ; 10.120 ms) |
[10.120 ms ; 13.203 ms) | @
[13.203 ms ; 16.285 ms) |
[16.285 ms ; 19.368 ms) |
[19.368 ms ; 22.451 ms) |
[22.451 ms ; 25.533 ms) |
[25.533 ms ; 28.616 ms) |
[28.616 ms ; 31.698 ms) |
[31.698 ms ; 34.781 ms) |
[34.781 ms ; 37.863 ms) |
[37.863 ms ; 40.946 ms) |
[40.946 ms ; 44.028 ms) |
[44.028 ms ; 47.111 ms) |
[47.111 ms ; 50.194 ms) |
[50.194 ms ; 53.553 ms) |
[53.553 ms ; 56.635 ms) | @
---------------------------------------------------

EsVsSqlServer.SqlServerContains_1000_Nullable: Job-JJCSWT(RunStrategy=ColdStart) [N=1000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 4.404 ms, StdErr = 1.320 ms (29.97%), N = 100, StdDev = 13.198 ms
Min = 0.693 ms, Q1 = 2.609 ms, Median = 2.906 ms, Q3 = 3.659 ms, Max = 134.087 ms
IQR = 1.050 ms, LowerFence = 1.033 ms, UpperFence = 5.235 ms
ConfidenceInterval = [-0.072 ms; 8.880 ms] (CI 99.9%), Margin = 4.476 ms (101.64% of Mean)
Skewness = 9.48, Kurtosis = 93.23, MValue = 2.16
-------------------- Histogram --------------------
[  0.643 ms ;   8.107 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[  8.107 ms ;  12.421 ms) | @@@
[ 12.421 ms ;  19.885 ms) |
[ 19.885 ms ;  27.349 ms) |
[ 27.349 ms ;  34.813 ms) |
[ 34.813 ms ;  42.277 ms) |
[ 42.277 ms ;  49.740 ms) |
[ 49.740 ms ;  57.204 ms) |
[ 57.204 ms ;  64.668 ms) |
[ 64.668 ms ;  72.132 ms) |
[ 72.132 ms ;  79.596 ms) |
[ 79.596 ms ;  87.060 ms) |
[ 87.060 ms ;  94.524 ms) |
[ 94.524 ms ; 101.988 ms) |
[101.988 ms ; 109.451 ms) |
[109.451 ms ; 116.915 ms) |
[116.915 ms ; 124.379 ms) |
[124.379 ms ; 130.355 ms) |
[130.355 ms ; 137.819 ms) | @
---------------------------------------------------

EsVsSqlServer.SqlServerContains_1000: Job-JJCSWT(RunStrategy=ColdStart) [N=1000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 4.512 ms, StdErr = 1.318 ms (29.22%), N = 100, StdDev = 13.181 ms
Min = 0.737 ms, Q1 = 2.710 ms, Median = 3.049 ms, Q3 = 3.599 ms, Max = 133.829 ms
IQR = 0.889 ms, LowerFence = 1.376 ms, UpperFence = 4.933 ms
ConfidenceInterval = [0.041 ms; 8.982 ms] (CI 99.9%), Margin = 4.470 ms (99.09% of Mean)
Skewness = 9.44, Kurtosis = 92.65, MValue = 2.21
-------------------- Histogram --------------------
[  0.682 ms ;   8.137 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[  8.137 ms ;  13.319 ms) | @@
[ 13.319 ms ;  20.774 ms) |
[ 20.774 ms ;  28.228 ms) |
[ 28.228 ms ;  35.683 ms) |
[ 35.683 ms ;  43.137 ms) |
[ 43.137 ms ;  50.591 ms) |
[ 50.591 ms ;  58.046 ms) |
[ 58.046 ms ;  65.500 ms) |
[ 65.500 ms ;  72.954 ms) |
[ 72.954 ms ;  80.409 ms) |
[ 80.409 ms ;  87.863 ms) |
[ 87.863 ms ;  95.317 ms) |
[ 95.317 ms ; 102.772 ms) |
[102.772 ms ; 110.226 ms) |
[110.226 ms ; 117.681 ms) |
[117.681 ms ; 125.135 ms) |
[125.135 ms ; 130.102 ms) |
[130.102 ms ; 137.556 ms) | @
---------------------------------------------------

EsVsSqlServer.SqlServerContains_10: Job-JJCSWT(RunStrategy=ColdStart) [N=1000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 4.392 ms, StdErr = 1.300 ms (29.60%), N = 100, StdDev = 13.000 ms
Min = 0.731 ms, Q1 = 2.623 ms, Median = 2.979 ms, Q3 = 3.522 ms, Max = 131.975 ms
IQR = 0.899 ms, LowerFence = 1.275 ms, UpperFence = 4.870 ms
ConfidenceInterval = [-0.017 ms; 8.801 ms] (CI 99.9%), Margin = 4.409 ms (100.39% of Mean)
Skewness = 9.45, Kurtosis = 92.76, MValue = 2.62
-------------------- Histogram --------------------
[  0.490 ms ;   7.843 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[  7.843 ms ;  13.287 ms) | @@
[ 13.287 ms ;  20.639 ms) |
[ 20.639 ms ;  27.991 ms) |
[ 27.991 ms ;  35.343 ms) |
[ 35.343 ms ;  42.695 ms) |
[ 42.695 ms ;  50.047 ms) |
[ 50.047 ms ;  57.400 ms) |
[ 57.400 ms ;  64.752 ms) |
[ 64.752 ms ;  72.104 ms) |
[ 72.104 ms ;  79.456 ms) |
[ 79.456 ms ;  86.808 ms) |
[ 86.808 ms ;  94.160 ms) |
[ 94.160 ms ; 101.512 ms) |
[101.512 ms ; 108.865 ms) |
[108.865 ms ; 116.217 ms) |
[116.217 ms ; 123.569 ms) |
[123.569 ms ; 128.299 ms) |
[128.299 ms ; 135.651 ms) | @
---------------------------------------------------

EsVsSqlServer.ElasticsearchMatch_250: Job-JJCSWT(RunStrategy=ColdStart) [N=1000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 5.799 ms, StdErr = 3.928 ms (67.74%), N = 100, StdDev = 39.280 ms
Min = 0.935 ms, Q1 = 1.158 ms, Median = 1.710 ms, Q3 = 2.142 ms, Max = 394.560 ms
IQR = 0.985 ms, LowerFence = -0.319 ms, UpperFence = 3.619 ms
ConfidenceInterval = [-7.523 ms; 19.121 ms] (CI 99.9%), Margin = 13.322 ms (229.73% of Mean)
Skewness = 9.69, Kurtosis = 95.95, MValue = 3.09
-------------------- Histogram --------------------
[ -7.581 ms ;  14.633 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 14.633 ms ;  36.847 ms) |
[ 36.847 ms ;  59.062 ms) |
[ 59.062 ms ;  81.276 ms) |
[ 81.276 ms ; 103.490 ms) |
[103.490 ms ; 125.705 ms) |
[125.705 ms ; 147.919 ms) |
[147.919 ms ; 170.133 ms) |
[170.133 ms ; 192.348 ms) |
[192.348 ms ; 214.562 ms) |
[214.562 ms ; 236.776 ms) |
[236.776 ms ; 258.991 ms) |
[258.991 ms ; 281.205 ms) |
[281.205 ms ; 303.419 ms) |
[303.419 ms ; 325.634 ms) |
[325.634 ms ; 347.848 ms) |
[347.848 ms ; 370.062 ms) |
[370.062 ms ; 383.453 ms) |
[383.453 ms ; 405.667 ms) | @
---------------------------------------------------

EsVsSqlServer.ElasticsearchMatch_1000_Nullable: Job-JJCSWT(RunStrategy=ColdStart) [N=1000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 5.490 ms, StdErr = 3.874 ms (70.57%), N = 100, StdDev = 38.744 ms
Min = 0.860 ms, Q1 = 1.087 ms, Median = 1.284 ms, Q3 = 1.960 ms, Max = 388.971 ms
IQR = 0.873 ms, LowerFence = -0.223 ms, UpperFence = 3.270 ms
ConfidenceInterval = [-7.650 ms; 18.630 ms] (CI 99.9%), Margin = 13.140 ms (239.33% of Mean)
Skewness = 9.7, Kurtosis = 95.98, MValue = 2.52
-------------------- Histogram --------------------
[-10.096 ms ;  13.882 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 13.882 ms ;  35.793 ms) |
[ 35.793 ms ;  57.704 ms) |
[ 57.704 ms ;  79.615 ms) |
[ 79.615 ms ; 101.526 ms) |
[101.526 ms ; 123.437 ms) |
[123.437 ms ; 145.348 ms) |
[145.348 ms ; 167.260 ms) |
[167.260 ms ; 189.171 ms) |
[189.171 ms ; 211.082 ms) |
[211.082 ms ; 232.993 ms) |
[232.993 ms ; 254.904 ms) |
[254.904 ms ; 276.815 ms) |
[276.815 ms ; 298.726 ms) |
[298.726 ms ; 320.637 ms) |
[320.637 ms ; 342.548 ms) |
[342.548 ms ; 364.459 ms) |
[364.459 ms ; 378.015 ms) |
[378.015 ms ; 399.926 ms) | @
---------------------------------------------------

EsVsSqlServer.ElasticsearchMatch_1000: Job-JJCSWT(RunStrategy=ColdStart) [N=1000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 5.626 ms, StdErr = 3.805 ms (67.64%), N = 100, StdDev = 38.049 ms
Min = 0.874 ms, Q1 = 1.031 ms, Median = 1.430 ms, Q3 = 2.234 ms, Max = 382.172 ms
IQR = 1.203 ms, LowerFence = -0.772 ms, UpperFence = 4.038 ms
ConfidenceInterval = [-7.279 ms; 18.530 ms] (CI 99.9%), Margin = 12.905 ms (229.39% of Mean)
Skewness = 9.69, Kurtosis = 95.91, MValue = 2.75
-------------------- Histogram --------------------
[ -7.368 ms ;  14.151 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 14.151 ms ;  35.669 ms) |
[ 35.669 ms ;  57.188 ms) |
[ 57.188 ms ;  78.706 ms) |
[ 78.706 ms ; 100.224 ms) |
[100.224 ms ; 121.743 ms) |
[121.743 ms ; 143.261 ms) |
[143.261 ms ; 164.780 ms) |
[164.780 ms ; 186.298 ms) |
[186.298 ms ; 207.817 ms) |
[207.817 ms ; 229.335 ms) |
[229.335 ms ; 250.853 ms) |
[250.853 ms ; 272.372 ms) |
[272.372 ms ; 293.890 ms) |
[293.890 ms ; 315.409 ms) |
[315.409 ms ; 336.927 ms) |
[336.927 ms ; 358.445 ms) |
[358.445 ms ; 371.413 ms) |
[371.413 ms ; 392.931 ms) | @
---------------------------------------------------

EsVsSqlServer.ElasticsearchMatch_10: Job-JJCSWT(RunStrategy=ColdStart) [N=1000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 5.705 ms, StdErr = 3.891 ms (68.20%), N = 100, StdDev = 38.905 ms
Min = 0.895 ms, Q1 = 1.115 ms, Median = 1.568 ms, Q3 = 2.217 ms, Max = 390.759 ms
IQR = 1.102 ms, LowerFence = -0.537 ms, UpperFence = 3.869 ms
ConfidenceInterval = [-7.490 ms; 18.900 ms] (CI 99.9%), Margin = 13.195 ms (231.29% of Mean)
Skewness = 9.69, Kurtosis = 95.95, MValue = 2.15
-------------------- Histogram --------------------
[ -7.496 ms ;  14.506 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 14.506 ms ;  36.508 ms) |
[ 36.508 ms ;  58.511 ms) |
[ 58.511 ms ;  80.513 ms) |
[ 80.513 ms ; 102.515 ms) |
[102.515 ms ; 124.518 ms) |
[124.518 ms ; 146.520 ms) |
[146.520 ms ; 168.523 ms) |
[168.523 ms ; 190.525 ms) |
[190.525 ms ; 212.527 ms) |
[212.527 ms ; 234.530 ms) |
[234.530 ms ; 256.532 ms) |
[256.532 ms ; 278.534 ms) |
[278.534 ms ; 300.537 ms) |
[300.537 ms ; 322.539 ms) |
[322.539 ms ; 344.541 ms) |
[344.541 ms ; 366.544 ms) |
[366.544 ms ; 379.758 ms) |
[379.758 ms ; 401.761 ms) | @
---------------------------------------------------

EsVsSqlServer.SqlServerContains_250: Job-JJCSWT(RunStrategy=ColdStart) [N=100000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 27.463 ms, StdErr = 18.658 ms (67.94%), N = 100, StdDev = 186.581 ms
Min = 0.867 ms, Q1 = 2.967 ms, Median = 3.391 ms, Q3 = 3.900 ms, Max = 1,762.446 ms
IQR = 0.933 ms, LowerFence = 1.568 ms, UpperFence = 5.299 ms
ConfidenceInterval = [-35.816 ms; 90.743 ms] (CI 99.9%), Margin = 63.280 ms (230.42% of Mean)
Skewness = 8.4, Kurtosis = 75.96, MValue = 2.52
-------------------- Histogram --------------------
[  -51.893 ms ;    58.579 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[   58.579 ms ;   164.098 ms) |
[  164.098 ms ;   269.617 ms) |
[  269.617 ms ;   375.136 ms) |
[  375.136 ms ;   480.655 ms) |
[  480.655 ms ;   590.803 ms) |
[  590.803 ms ;   696.322 ms) | @
[  696.322 ms ;   801.841 ms) |
[  801.841 ms ;   907.360 ms) |
[  907.360 ms ; 1,012.879 ms) |
[1,012.879 ms ; 1,118.398 ms) |
[1,118.398 ms ; 1,223.917 ms) |
[1,223.917 ms ; 1,329.436 ms) |
[1,329.436 ms ; 1,434.955 ms) |
[1,434.955 ms ; 1,540.473 ms) |
[1,540.473 ms ; 1,645.992 ms) |
[1,645.992 ms ; 1,709.687 ms) |
[1,709.687 ms ; 1,815.206 ms) | @
---------------------------------------------------

EsVsSqlServer.SqlServerContains_1000_Nullable: Job-JJCSWT(RunStrategy=ColdStart) [N=100000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 4.567 ms, StdErr = 1.356 ms (29.69%), N = 100, StdDev = 13.561 ms
Min = 0.760 ms, Q1 = 2.760 ms, Median = 3.036 ms, Q3 = 3.424 ms, Max = 137.842 ms
IQR = 0.665 ms, LowerFence = 1.763 ms, UpperFence = 4.421 ms
ConfidenceInterval = [-0.032 ms; 9.166 ms] (CI 99.9%), Margin = 4.599 ms (100.70% of Mean)
Skewness = 9.49, Kurtosis = 93.3, MValue = 2.32
-------------------- Histogram --------------------
[  0.549 ms ;   8.218 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[  8.218 ms ;  12.852 ms) | @@@
[ 12.852 ms ;  20.521 ms) |
[ 20.521 ms ;  28.190 ms) |
[ 28.190 ms ;  35.859 ms) |
[ 35.859 ms ;  43.529 ms) |
[ 43.529 ms ;  51.198 ms) |
[ 51.198 ms ;  58.867 ms) |
[ 58.867 ms ;  66.536 ms) |
[ 66.536 ms ;  74.205 ms) |
[ 74.205 ms ;  81.874 ms) |
[ 81.874 ms ;  89.543 ms) |
[ 89.543 ms ;  97.212 ms) |
[ 97.212 ms ; 104.881 ms) |
[104.881 ms ; 112.551 ms) |
[112.551 ms ; 120.220 ms) |
[120.220 ms ; 127.889 ms) |
[127.889 ms ; 134.007 ms) |
[134.007 ms ; 141.676 ms) | @
---------------------------------------------------

EsVsSqlServer.SqlServerContains_1000: Job-JJCSWT(RunStrategy=ColdStart) [N=100000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 4.286 ms, StdErr = 1.390 ms (32.44%), N = 100, StdDev = 13.903 ms
Min = 0.740 ms, Q1 = 2.030 ms, Median = 2.989 ms, Q3 = 3.317 ms, Max = 140.976 ms
IQR = 1.286 ms, LowerFence = 0.101 ms, UpperFence = 5.246 ms
ConfidenceInterval = [-0.429 ms; 9.001 ms] (CI 99.9%), Margin = 4.715 ms (110.01% of Mean)
Skewness = 9.5, Kurtosis = 93.44, MValue = 3.23
-------------------- Histogram --------------------
[  0.500 ms ;   8.362 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[  8.362 ms ;  14.265 ms) | @
[ 14.265 ms ;  22.128 ms) |
[ 22.128 ms ;  29.990 ms) |
[ 29.990 ms ;  37.853 ms) |
[ 37.853 ms ;  45.716 ms) |
[ 45.716 ms ;  53.579 ms) |
[ 53.579 ms ;  61.441 ms) |
[ 61.441 ms ;  69.304 ms) |
[ 69.304 ms ;  77.167 ms) |
[ 77.167 ms ;  85.029 ms) |
[ 85.029 ms ;  92.892 ms) |
[ 92.892 ms ; 100.755 ms) |
[100.755 ms ; 108.617 ms) |
[108.617 ms ; 116.480 ms) |
[116.480 ms ; 124.343 ms) |
[124.343 ms ; 132.206 ms) |
[132.206 ms ; 137.045 ms) |
[137.045 ms ; 144.907 ms) | @
---------------------------------------------------

EsVsSqlServer.SqlServerContains_10: Job-JJCSWT(RunStrategy=ColdStart) [N=100000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 4.509 ms, StdErr = 0.973 ms (21.58%), N = 100, StdDev = 9.731 ms
Min = 0.746 ms, Q1 = 2.698 ms, Median = 3.006 ms, Q3 = 3.567 ms, Max = 78.985 ms
IQR = 0.870 ms, LowerFence = 1.393 ms, UpperFence = 4.872 ms
ConfidenceInterval = [1.209 ms; 7.810 ms] (CI 99.9%), Margin = 3.300 ms (73.18% of Mean)
Skewness = 6.55, Kurtosis = 46.58, MValue = 3.36
-------------------- Histogram --------------------
[ 0.484 ms ;  5.987 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 5.987 ms ; 11.547 ms) | @@@@@@@
[11.547 ms ; 17.050 ms) |
[17.050 ms ; 22.553 ms) |
[22.553 ms ; 28.056 ms) |
[28.056 ms ; 33.559 ms) |
[33.559 ms ; 39.062 ms) |
[39.062 ms ; 44.565 ms) |
[44.565 ms ; 50.069 ms) |
[50.069 ms ; 55.572 ms) |
[55.572 ms ; 59.341 ms) |
[59.341 ms ; 64.845 ms) | @
[64.845 ms ; 70.348 ms) |
[70.348 ms ; 76.233 ms) |
[76.233 ms ; 81.736 ms) | @
---------------------------------------------------

EsVsSqlServer.ElasticsearchMatch_250: Job-JJCSWT(RunStrategy=ColdStart) [N=100000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 6.321 ms, StdErr = 4.535 ms (71.74%), N = 100, StdDev = 45.348 ms
Min = 0.833 ms, Q1 = 1.099 ms, Median = 1.315 ms, Q3 = 2.239 ms, Max = 455.144 ms
IQR = 1.140 ms, LowerFence = -0.610 ms, UpperFence = 3.948 ms
ConfidenceInterval = [-9.059 ms; 21.701 ms] (CI 99.9%), Margin = 15.380 ms (243.30% of Mean)
Skewness = 9.69, Kurtosis = 95.95, MValue = 2.65
-------------------- Histogram --------------------
[ -8.569 ms ;  17.077 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 17.077 ms ;  42.723 ms) |
[ 42.723 ms ;  68.369 ms) |
[ 68.369 ms ;  94.015 ms) |
[ 94.015 ms ; 119.661 ms) |
[119.661 ms ; 145.307 ms) |
[145.307 ms ; 170.953 ms) |
[170.953 ms ; 196.599 ms) |
[196.599 ms ; 222.245 ms) |
[222.245 ms ; 247.891 ms) |
[247.891 ms ; 273.537 ms) |
[273.537 ms ; 299.183 ms) |
[299.183 ms ; 324.830 ms) |
[324.830 ms ; 350.476 ms) |
[350.476 ms ; 376.122 ms) |
[376.122 ms ; 401.768 ms) |
[401.768 ms ; 427.414 ms) |
[427.414 ms ; 442.321 ms) |
[442.321 ms ; 467.967 ms) | @
---------------------------------------------------

EsVsSqlServer.ElasticsearchMatch_1000_Nullable: Job-JJCSWT(RunStrategy=ColdStart) [N=100000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 5.589 ms, StdErr = 3.864 ms (69.13%), N = 100, StdDev = 38.637 ms
Min = 0.865 ms, Q1 = 1.116 ms, Median = 1.438 ms, Q3 = 2.046 ms, Max = 387.995 ms
IQR = 0.930 ms, LowerFence = -0.279 ms, UpperFence = 3.441 ms
ConfidenceInterval = [-7.514 ms; 18.693 ms] (CI 99.9%), Margin = 13.104 ms (234.44% of Mean)
Skewness = 9.69, Kurtosis = 95.96, MValue = 2.77
-------------------- Histogram --------------------
[ -7.548 ms ;  14.303 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 14.303 ms ;  36.153 ms) |
[ 36.153 ms ;  58.004 ms) |
[ 58.004 ms ;  79.854 ms) |
[ 79.854 ms ; 101.705 ms) |
[101.705 ms ; 123.555 ms) |
[123.555 ms ; 145.406 ms) |
[145.406 ms ; 167.256 ms) |
[167.256 ms ; 189.107 ms) |
[189.107 ms ; 210.957 ms) |
[210.957 ms ; 232.808 ms) |
[232.808 ms ; 254.658 ms) |
[254.658 ms ; 276.509 ms) |
[276.509 ms ; 298.359 ms) |
[298.359 ms ; 320.210 ms) |
[320.210 ms ; 342.060 ms) |
[342.060 ms ; 363.911 ms) |
[363.911 ms ; 377.069 ms) |
[377.069 ms ; 398.920 ms) | @
---------------------------------------------------

EsVsSqlServer.ElasticsearchMatch_1000: Job-JJCSWT(RunStrategy=ColdStart) [N=100000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 5.590 ms, StdErr = 3.837 ms (68.63%), N = 100, StdDev = 38.365 ms
Min = 0.876 ms, Q1 = 1.072 ms, Median = 1.312 ms, Q3 = 2.205 ms, Max = 385.259 ms
IQR = 1.132 ms, LowerFence = -0.626 ms, UpperFence = 3.903 ms
ConfidenceInterval = [-7.421 ms; 18.602 ms] (CI 99.9%), Margin = 13.012 ms (232.75% of Mean)
Skewness = 9.69, Kurtosis = 95.91, MValue = 2.69
-------------------- Histogram --------------------
[ -6.369 ms ;  15.328 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 15.328 ms ;  37.025 ms) |
[ 37.025 ms ;  58.722 ms) |
[ 58.722 ms ;  80.420 ms) |
[ 80.420 ms ; 102.117 ms) |
[102.117 ms ; 123.814 ms) |
[123.814 ms ; 145.511 ms) |
[145.511 ms ; 167.208 ms) |
[167.208 ms ; 188.905 ms) |
[188.905 ms ; 210.602 ms) |
[210.602 ms ; 232.299 ms) |
[232.299 ms ; 253.996 ms) |
[253.996 ms ; 275.693 ms) |
[275.693 ms ; 297.390 ms) |
[297.390 ms ; 319.087 ms) |
[319.087 ms ; 340.784 ms) |
[340.784 ms ; 362.481 ms) |
[362.481 ms ; 374.410 ms) |
[374.410 ms ; 396.108 ms) | @
---------------------------------------------------

EsVsSqlServer.ElasticsearchMatch_10: Job-JJCSWT(RunStrategy=ColdStart) [N=100000]
Runtime = .NET Core 3.1.13 (CoreCLR 4.700.21.11102, CoreFX 4.700.21.11602), X64 RyuJIT; GC = Concurrent Workstation
Mean = 5.498 ms, StdErr = 3.952 ms (71.88%), N = 100, StdDev = 39.519 ms
Min = 0.884 ms, Q1 = 1.065 ms, Median = 1.253 ms, Q3 = 1.833 ms, Max = 396.668 ms
IQR = 0.768 ms, LowerFence = -0.087 ms, UpperFence = 2.985 ms
ConfidenceInterval = [-7.905 ms; 18.901 ms] (CI 99.9%), Margin = 13.403 ms (243.78% of Mean)
Skewness = 9.7, Kurtosis = 95.99, MValue = 2.28
-------------------- Histogram --------------------
[-10.290 ms ;  13.706 ms) | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
[ 13.706 ms ;  36.056 ms) |
[ 36.056 ms ;  58.405 ms) |
[ 58.405 ms ;  80.755 ms) |
[ 80.755 ms ; 103.104 ms) |
[103.104 ms ; 125.454 ms) |
[125.454 ms ; 147.803 ms) |
[147.803 ms ; 170.153 ms) |
[170.153 ms ; 192.502 ms) |
[192.502 ms ; 214.852 ms) |
[214.852 ms ; 237.201 ms) |
[237.201 ms ; 259.551 ms) |
[259.551 ms ; 281.900 ms) |
[281.900 ms ; 304.250 ms) |
[304.250 ms ; 326.599 ms) |
[326.599 ms ; 348.949 ms) |
[348.949 ms ; 371.298 ms) |
[371.298 ms ; 385.494 ms) |
[385.494 ms ; 407.843 ms) | @
---------------------------------------------------
```

# Links
1. [CREATE FULLTEXT INDEX](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-fulltext-index-transact-sql?view=sql-server-ver15)
2. [ALTER FULLTEXT INDEX](https://docs.microsoft.com/en-us/sql/t-sql/statements/alter-fulltext-index-transact-sql?view=sql-server-ver15)
3. <https://benchmarkdotnet.org/>
4. <https://github.com/elastic/elasticsearch-net>
