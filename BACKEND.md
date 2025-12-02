# 🛠️ Backend Specification

## 📋 Обзор Backend

### Технологический стек

| Компонент | Технология | Версия |
|-----------|-----------|--------|
| **Runtime** | .NET | 8.0+ |
| **Language** | C# | 12 |
| **Framework** | ASP.NET Core | 8.0+ |
| **Web API** | Ocelot (API Gateway) | 20.0+ |
| **Database** | PostgreSQL / MongoDB | 15 / 7 |
| **Search** | Elasticsearch | 8.11+ |
| **Cache** | Redis | 7+ |
| **Graph** | Neo4j | 5+ |
| **Message Queue** | RabbitMQ | 3.12+ |
| **Auth** | Keycloak | 22+ |
| **Testing** | xUnit, Moq | Latest |
| **Logging** | Serilog, ELK | Latest |

---

## 🏗️ Архитектура сервисов

### Структура проекта

```
backend/
├── src/
│   ├── ApiGateway/
│   │   ├── Startup.cs
│   │   ├── ocelot.json
│   │   └── Program.cs
│   │
│   └── Services/
│       ├── PrimaNext.User.API/
│       │   ├── Controllers/
│       │   ├── Services/
│       │   ├── Models/
│       │   ├── Data/
│       │   ├── Events/
│       │   └── Dockerfile
│       │
│       ├── PrimaNext.Company.API/
│       ├── PrimaNext.Search.API/
│       ├── PrimaNext.Report.API/
│       ├── PrimaNext.Monitoring.API/
│       ├── PrimaNext.Graph.API/
│       ├── PrimaNext.Integration.API/
│       └── ...
│
├── tests/
│   ├── PrimaNext.Tests.Unit/
│   ├── PrimaNext.Tests.Integration/
│   └── PrimaNext.Tests.E2E/
│
├── shared/
│   ├── PrimaNext.Domain/
│   ├── PrimaNext.Application/
│   └── PrimaNext.Infrastructure/
│
├── docker-compose.yml
└── Dockerfile
```

### Микросервисы

#### 1. User Service

**Назначение:** Управление пользователями, ролями, тарифами

**Endpoints:**
```
POST   /api/users/register              # Регистрация
POST   /api/users/login                 # Вход (JWT)
POST   /api/users/refresh-token         # Обновить токен
POST   /api/users/logout                # Выход
GET    /api/users/profile               # Профиль текущего пользователя
PUT    /api/users/profile               # Обновить профиль
GET    /api/users/{id}                  # Профиль пользователя (admin)
PUT    /api/users/{id}                  # Изменить пользователя (admin)
DELETE /api/users/{id}                  # Удалить пользователя (admin)
GET    /api/users/{id}/subscriptions    # Подписки пользователя
POST   /api/users/{id}/subscribe        # Оформить подписку
GET    /api/users/{id}/limits           # Лимиты тарифа
GET    /api/roles                       # Список ролей (admin)
POST   /api/roles                       # Создать роль (admin)
PUT    /api/roles/{id}                  # Изменить роль (admin)
DELETE /api/roles/{id}                  # Удалить роль (admin)
```

**Models:**
```csharp
public class User
{
    public Guid Id { get; set; }
    public string Email { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string PasswordHash { get; set; }
    public UserStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? LastLoginAt { get; set; }
    public ICollection<UserRole> Roles { get; set; }
    public ICollection<Subscription> Subscriptions { get; set; }
}

public class Subscription
{
    public Guid Id { get; set; }
    public Guid UserId { get; set; }
    public Guid PlanId { get; set; }
    public DateTime StartDate { get; set; }
    public DateTime? EndDate { get; set; }
    public SubscriptionStatus Status { get; set; }
}

public class SubscriptionPlan
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Code { get; set; }
    public decimal? PriceMonthly { get; set; }
    public decimal? PriceYearly { get; set; }
    public PlanLimits Limits { get; set; }
    public bool IsActive { get; set; }
}

public class PlanLimits
{
    public int? MaxSearchRequests { get; set; }
    public int? MaxExports { get; set; }
    public int? MaxMonitoringItems { get; set; }
    public int? MaxGraphDepth { get; set; }
    public int? MaxConcurrentReports { get; set; }
    public List<string> AllowedDataSources { get; set; }
}
```

**Database:**
```sql
-- PostgreSQL
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    password_hash VARCHAR(512) NOT NULL,
    status ENUM('Active', 'Inactive', 'Suspended') DEFAULT 'Active',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    last_login_at TIMESTAMP
);

CREATE TABLE roles (
    id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE user_roles (
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE subscription_plans (
    id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(50) NOT NULL UNIQUE,
    price_monthly DECIMAL(10, 2),
    price_yearly DECIMAL(10, 2),
    limits JSONB,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE subscriptions (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    plan_id UUID REFERENCES subscription_plans(id),
    start_date DATE NOT NULL,
    end_date DATE,
    status ENUM('Active', 'Expired', 'Cancelled') DEFAULT 'Active',
    created_at TIMESTAMP DEFAULT NOW()
);
```

#### 2. Company Service

**Назначение:** Управление данными компаний

**Endpoints:**
```
GET    /api/companies                    # Список с фильтрацией
GET    /api/companies/{id}               # Деталь компании
POST   /api/companies                    # Создать компанию
PUT    /api/companies/{id}               # Обновить компанию
DELETE /api/companies/{id}               # Удалить компанию
GET    /api/companies/{id}/financials    # Финансовые данные
GET    /api/companies/{id}/founders      # Учредители
GET    /api/companies/{id}/connections  # Связи
GET    /api/companies/{id}/risks         # Риски
GET    /api/companies/{id}/history       # История изменений
GET    /api/companies/{id}/documents     # Документы
POST   /api/companies/batch-check        # Проверка списка
```

**Models:**
```csharp
public class Company
{
    public Guid Id { get; set; }
    public string INN { get; set; }           // 10 или 12 цифр
    public string OGRN { get; set; }          // 13 цифр
    public string Name { get; set; }
    public string ShortName { get; set; }
    public CompanyType Type { get; set; }     // ООО, АО, ИП, и т.д.
    public string Status { get; set; }        // Active, Liquidating, Liquidated
    public string OKVED { get; set; }         // Основной вид деятельности
    public string Address { get; set; }
    public GeoCoordinates Coordinates { get; set; }
    public string RegistrationNumber { get; set; }
    public DateTime RegistrationDate { get; set; }
    public DateTime? LiquidationDate { get; set; }
    public string DirectorName { get; set; }
    public string DirectorINN { get; set; }
    public List<Founder> Founders { get; set; }
    public List<FinancialData> FinancialHistory { get; set; }
    public CompanyRisks Risks { get; set; }
    public DateTime DataUpdatedAt { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}

public class FinancialData
{
    public Guid Id { get; set; }
    public Guid CompanyId { get; set; }
    public int Year { get; set; }
    public decimal Revenue { get; set; }
    public decimal NetIncome { get; set; }
    public decimal Assets { get; set; }
    public decimal Liabilities { get; set; }
    public decimal Equity { get; set; }
    public decimal CurrentRatio { get; set; }
    public decimal DebtToEquity { get; set; }
    public DateTime ReportDate { get; set; }
}

public class Founder
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Type { get; set; }      // Individual, Legal Entity
    public string INN { get; set; }
    public decimal SharePercent { get; set; }
    public DateTime FromDate { get; set; }
    public DateTime? ToDate { get; set; }
}

public class CompanyRisks
{
    public FinancialRisk Financial { get; set; }
    public LegalRisk Legal { get; set; }
    public OperationalRisk Operational { get; set; }
    public ReputationRisk Reputation { get; set; }
    public double OverallScore { get; set; }  // 0-100
}
```

**Database (MongoDB):**
```javascript
db.createCollection("companies", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["inn", "ogrn", "name"],
            properties: {
                _id: { bsonType: "objectId" },
                inn: { bsonType: "string", pattern: "^[0-9]{10,12}$" },
                ogrn: { bsonType: "string", pattern: "^[0-9]{13}$" },
                name: { bsonType: "string" },
                type: { bsonType: "string", enum: ["OOO", "AO", "IP", "..."] },
                status: { bsonType: "string" },
                founders: { bsonType: "array" },
                financials: { bsonType: "array" },
                risks: { bsonType: "object" },
                dataUpdatedAt: { bsonType: "date" }
            }
        }
    }
});

// Индексы
db.companies.createIndex({ inn: 1 }, { unique: true });
db.companies.createIndex({ ogrn: 1 }, { unique: true });
db.companies.createIndex({ name: "text", shortName: "text" });
db.companies.createIndex({ status: 1 });
db.companies.createIndex({ okved: 1 });
```

#### 3. Search Service

**Назначение:** Полнотекстовый и фасетированный поиск

**Endpoints:**
```
GET    /api/search                       # Поиск по всем сущностям
GET    /api/search/companies             # Поиск компаний
GET    /api/search/founders              # Поиск учредителей
GET    /api/search/documents             # Поиск документов
POST   /api/search/advanced              # Расширенный поиск
GET    /api/search/suggestions           # Автодополнение
GET    /api/search/facets                # Фасеты для фильтрации
```

**Search Indices (Elasticsearch):**
```json
{
  "companies_v1": {
    "mappings": {
      "properties": {
        "inn": { "type": "keyword" },
        "ogrn": { "type": "keyword" },
        "name": { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
        "shortName": { "type": "text" },
        "type": { "type": "keyword" },
        "status": { "type": "keyword" },
        "okved": { "type": "keyword" },
        "address": { "type": "text" },
        "region": { "type": "keyword" },
        "dataUpdatedAt": { "type": "date" }
      }
    }
  }
}
```

---

## 📚 Common Patterns & Libraries

### Dependency Injection

```csharp
services
    .AddScoped<IUserService, UserService>()
    .AddScoped<ICompanyService, CompanyService>()
    .AddScoped<ISearchService, SearchService>()
    .AddSingleton<ICacheService, RedisCacheService>()
    .AddHttpClient<IExternalApiClient, ExternalApiClient>();
```

### Error Handling

```csharp
public class ApiException : Exception
{
    public int StatusCode { get; set; }
    public string ErrorCode { get; set; }
    public List<string> Details { get; set; }
}

[ApiExceptionFilter]
public class CompanyController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<IActionResult> GetCompany(Guid id)
    {
        try
        {
            var company = await _service.GetCompanyAsync(id);
            return Ok(company);
        }
        catch (NotFoundException ex)
        {
            throw new ApiException(404, "COMPANY_NOT_FOUND", ex.Message);
        }
    }
}
```

### Logging

```csharp
// Serilog configuration
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .WriteTo.Console()
    .WriteTo.File("logs/app-.txt", rollingInterval: RollingInterval.Day)
    .Enrich.FromLogContext()
    .CreateLogger();
```

### Database Migrations

```bash
# Create new migration
dotnet ef migrations add AddFinancialData --project src/Services/Company.API

# Apply migrations
dotnet ef database update --project src/Services/Company.API

# Rollback
dotnet ef migrations remove --project src/Services/Company.API
```

---

## 🔒 Authentication & Authorization

### JWT Claims

```csharp
var claims = new List<Claim>
{
    new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
    new Claim(ClaimTypes.Email, user.Email),
    new Claim("subscription_id", subscription.Id.ToString()),
    new Claim("plan_code", subscription.Plan.Code),
    new Claim(ClaimTypes.Role, "User")
};

var token = new JwtSecurityToken(
    issuer: _jwtSettings.Issuer,
    audience: _jwtSettings.Audience,
    claims: claims,
    expires: DateTime.UtcNow.AddHours(24),
    signingCredentials: new SigningCredentials(key, SecurityAlgorithm.HmacSha256)
);
```

### Authorization Policies

```csharp
services.AddAuthorization(options =>
{
    options.AddPolicy("UserOnly", policy =>
        policy.RequireRole("User", "Admin"));
    
    options.AddPolicy("CanExport", policy =>
        policy.RequireClaim("can_export", "true"));
    
    options.AddPolicy("PremiumOnly", policy =>
        policy.RequireClaim("plan_code", "premium", "enterprise"));
});
```

---

## 📊 Testing

### Unit Tests

```csharp
[Fact]
public async Task GetCompany_WithValidId_ReturnsCompany()
{
    // Arrange
    var companyId = Guid.NewGuid();
    var mockService = new Mock<ICompanyService>();
    mockService
        .Setup(s => s.GetCompanyAsync(companyId))
        .ReturnsAsync(new Company { Id = companyId, Name = "Test LLC" });
    
    var controller = new CompanyController(mockService.Object);
    
    // Act
    var result = await controller.GetCompany(companyId);
    
    // Assert
    var okResult = Assert.IsType<OkObjectResult>(result);
    var company = Assert.IsType<Company>(okResult.Value);
    Assert.Equal(companyId, company.Id);
}
```

### Integration Tests

```csharp
public class CompanyServiceIntegrationTests : IAsyncLifetime
{
    private readonly PostgresqlContainer _postgresContainer;
    private readonly ICompanyService _service;
    
    public async Task InitializeAsync()
    {
        _postgresContainer = new PostgresqlBuilder().Build();
        await _postgresContainer.StartAsync();
        // Setup DbContext with container connection string
    }
    
    [Fact]
    public async Task CreateCompany_SavesAndRetrieves()
    {
        var company = new Company { INN = "7701102700", ... };
        var created = await _service.CreateCompanyAsync(company);
        var retrieved = await _service.GetCompanyAsync(created.Id);
        
        Assert.NotNull(retrieved);
        Assert.Equal(company.INN, retrieved.INN);
    }
}
```

---

## 🚀 Deployment

### Docker

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 as runtime
FROM mcr.microsoft.com/dotnet/sdk:8.0 as builder

WORKDIR /app
COPY . .
RUN dotnet publish -c Release -o out

FROM runtime
WORKDIR /app
COPY --from=builder /app/out .
ENTRYPOINT ["dotnet", "PrimaNext.Company.API.dll"]
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: company-service
  namespace: primanext
spec:
  replicas: 3
  selector:
    matchLabels:
      app: company-service
  template:
    metadata:
      labels:
        app: company-service
    spec:
      containers:
      - name: company-service
        image: primanext/company-service:latest
        ports:
        - containerPort: 80
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__MongoDB
          valueFrom:
            secretKeyRef:
              name: primanext-secrets
              key: mongodb-connection-string
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

---

**Последнее обновление:** Декабрь 2024  
**Версия:** 2.0 Backend Spec
