# Технический проект: Система ПримаNEXT
## Полное содержание чата с детализацией всех компонентов

---

## Содержание

1. [Введение и цели проекта](#введение-и-цели-проекта)
2. [Административная панель](#1-административная-панель)
3. [Управление состоянием пользователя](#2-управление-состоянием-пользователя)
4. [Интеграция локальной LLM](#3-интеграция-локальной-llm)
5. [Картографический сервис](#4-картографический-сервис)
6. [Архитектура интерфейса](#5-архитектура-интерфейса)
7. [Технологический стек](#технологический-стек)

---

## Введение и цели проекта

### Описание системы

**ПримаNEXT** — система анализа и мониторинга российских компаний на основе данных ЕГРЮЛ, арбитражных судов, ФССП, финансовой отчетности и других открытых источников.

### Ключевые возможности

- Поиск и анализ компаний по ИНН, ОГРН, названию
- Визуализация данных: таблицы, дашборды, графы связей, карты
- AI-генерация аналитических отчетов
- Мониторинг изменений компаний
- Управление тарифами и правами доступа
- Геопространственная визуализация

---

## 1. Административная панель

### 1.1 Архитектура административной подсистемы

```
┌─────────────────────────────────────────────────────────────┐
│              ADMIN PANEL (Vue 3 + TypeScript)                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ User Management     │ Plan Management  │ Access Control│ │
│  │ - CRUD Users        │ - CRUD Plans     │ - Field Rules │ │
│  │ - Roles Assignment  │ - Pricing        │ - Visibility  │ │
│  │ - Subscriptions     │ - Features       │ - Permissions │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            ↓ HTTPS + JWT (Admin Role)
┌─────────────────────────────────────────────────────────────┐
│                    API GATEWAY (Ocelot)                      │
│  Authorization: Required Role = "admin"                      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    ADMIN SERVICE (C#)                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ UserAdministrationController                           │ │
│  │ PlanAdministrationController                           │ │
│  │ AccessControlController                                │ │
│  │ AuditLogController                                     │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Domain Models

#### SubscriptionPlan (Тарифный план)

```csharp
public class SubscriptionPlan
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Code { get; set; }
    public string Description { get; set; }
    public int SortOrder { get; set; }

    // Ценообразование
    public decimal? PriceMonthly { get; set; }
    public decimal? PriceYearly { get; set; }
    public string Currency { get; set; } = "RUB";
    public int? TrialDays { get; set; }

    // Лимиты
    public PlanLimits Limits { get; set; }

    // Доступные функции
    public List<string> Features { get; set; }

    // Правила доступа к данным
    public DataAccessRules AccessRules { get; set; }

    public bool IsActive { get; set; } = true;
    public bool IsPublic { get; set; } = true;
    public bool AllowSelfSubscription { get; set; } = true;

    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    public string CreatedBy { get; set; }
    public string UpdatedBy { get; set; }
}
```

#### PlanLimits (Лимиты тарифа)

```csharp
public class PlanLimits
{
    // Отчеты
    public int? ReportsPerMonth { get; set; }
    public int? ReportsPerDay { get; set; }
    public List<string> AllowedReportTypes { get; set; }
    public bool AllowReportWithSignature { get; set; }

    // Мониторинг
    public int? MonitoringSlots { get; set; }
    public List<string> MonitoringEvents { get; set; }

    // Поиск
    public int? SearchesPerDay { get; set; }
    public int? MassCheckLimit { get; set; }

    // API
    public int? ApiCallsPerDay { get; set; }
    public int? ApiRateLimitPerMinute { get; set; }

    // Экспорт
    public List<string> AllowedExportFormats { get; set; }
    public int? ExportsPerMonth { get; set; }

    // Списки
    public int? MaxListSize { get; set; }
    public int? MaxListsCount { get; set; }

    // Графы связей
    public int? GraphDepthLimit { get; set; }
    public int? GraphNodesLimit { get; set; }
}
```

#### DataAccessRules (Правила доступа к данным)

```csharp
public class DataAccessRules
{
    public List<string> AllowedCategories { get; set; }
    public Dictionary<string, FieldAccessRule> FieldRules { get; set; }
    public Dictionary<string, bool> DataSourceAccess { get; set; }
    public Dictionary<string, MaskingRule> MaskingRules { get; set; }
}

public class FieldAccessRule
{
    public string FieldPath { get; set; }
    public bool IsAccessible { get; set; }
    public DataCategory Category { get; set; }
    public FieldRestrictionType RestrictionType { get; set; }
    public string RestrictionMessage { get; set; }
}

public enum FieldRestrictionType
{
    FullAccess = 0,
    Restricted = 1,
    Masked = 2,
    Limited = 3,
    ReadOnly = 4
}
```

### 1.3 PostgreSQL Schema

```sql
CREATE TABLE subscription_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    code VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    sort_order INTEGER DEFAULT 0,

    price_monthly DECIMAL(10,2),
    price_yearly DECIMAL(10,2),
    currency VARCHAR(3) DEFAULT 'RUB',
    trial_days INTEGER,

    limits JSONB NOT NULL,
    features JSONB DEFAULT '[]',
    access_rules JSONB NOT NULL,

    is_active BOOLEAN DEFAULT true,
    is_public BOOLEAN DEFAULT true,
    allow_self_subscription BOOLEAN DEFAULT true,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(255),
    updated_by VARCHAR(255),

    INDEX idx_code (code),
    INDEX idx_is_active (is_active)
);

CREATE TABLE user_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id VARCHAR(255) NOT NULL,
    plan_id UUID REFERENCES subscription_plans(id),

    status VARCHAR(50) NOT NULL,
    start_date TIMESTAMP,
    end_date TIMESTAMP,
    auto_renew BOOLEAN DEFAULT false,

    is_custom_plan BOOLEAN DEFAULT false,
    custom_limits JSONB,
    custom_price DECIMAL(10,2),

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user_id (user_id),
    INDEX idx_plan_id (plan_id),
    INDEX idx_status (status)
);
```

### 1.4 API Controllers

```csharp
[ApiController]
[Route("api/admin/plans")]
[Authorize(Roles = "admin")]
public class PlanAdministrationController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<List<SubscriptionPlan>>> GetPlans()

    [HttpPost]
    public async Task<ActionResult<SubscriptionPlan>> CreatePlan([FromBody] CreatePlanRequest request)

    [HttpPut("{planId}")]
    public async Task<ActionResult<SubscriptionPlan>> UpdatePlan(Guid planId, [FromBody] UpdatePlanRequest request)

    [HttpDelete("{planId}")]
    public async Task<IActionResult> DeletePlan(Guid planId)
}
```

---

## 2. Управление состоянием пользователя

### 2.1 Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                    FRONTEND (Vue 3 + Pinia)                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ User State Management                                  │ │
│  │ - Search History Store                                 │ │
│  │ - Workspace Store (selections, filters, views)         │ │
│  │ - Sync with Backend                                    │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            ↓ Auto-save + Manual sync
┌─────────────────────────────────────────────────────────────┐
│                    BACKEND (User State Service)              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│              MONGODB (users.workspace)                       │
│  - Recent searches                                           │
│  - Saved search results                                      │
│  - Selected entities                                         │
│  - Active filters                                            │
│  - Current view mode                                         │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Domain Models

#### UserWorkspace

```csharp
public class UserWorkspace
{
    public string Id { get; set; }
    public string UserId { get; set; }

    public List<SearchHistoryEntry> SearchHistory { get; set; }
    public List<SavedSearch> SavedSearches { get; set; }
    public WorkspaceState CurrentWorkspace { get; set; }
    public List<SavedWorkspaceSnapshot> SavedWorkspaces { get; set; }

    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

#### WorkspaceState

```csharp
public class WorkspaceState
{
    public List<SelectedEntity> SelectedEntities { get; set; }
    public ViewMode ActiveView { get; set; }
    public string LastSearchQuery { get; set; }
    public SearchFilters ActiveFilters { get; set; }
    public SortingState Sorting { get; set; }
    public TableSettings TableSettings { get; set; }
    public DashboardSettings DashboardSettings { get; set; }
    public GraphSettings GraphSettings { get; set; }
    public ListsState ListsState { get; set; }
    public DateTime LastUpdated { get; set; }
}

public enum ViewMode
{
    Table = 0,
    Dashboard = 1,
    Lists = 2,
    Graph = 3
}
```

### 2.3 Pinia Store (Frontend)

```typescript
export const useWorkspaceStore = defineStore('workspace', () => {
  const workspace = ref<WorkspaceState>({
    selectedEntities: [],
    activeView: 'Table',
    lastSearchQuery: '',
    activeFilters: {},
    // ...
  });

  // Selections
  function addSelection(entity: SelectedEntity) { }
  function removeSelection(entityId: string) { }
  function toggleSelection(entity: SelectedEntity) { }
  function clearSelections() { }

  // View Management
  function changeView(viewMode: ViewMode) { }

  // Filters
  function updateFilters(filters: SearchFilters) { }
  function clearFilters() { }

  // Search State Reset
  async function resetSearchState(newQuery: string, preserveSelections: boolean) { }

  // History (Undo/Redo)
  function undo() { }
  function redo() { }

  // Sync
  async function sync() { }

  return {
    workspace,
    addSelection,
    removeSelection,
    // ...
  };
});
```

---

## 3. Интеграция локальной LLM

### 3.1 Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                    FRONTEND (Vue 3)                          │
│  Entity Card - "Сгенерировать отчет с AI"                   │
└─────────────────────────────────────────────────────────────┘
                            ↓ WebSocket / SSE
┌─────────────────────────────────────────────────────────────┐
│                    LLM SERVICE (C# / Python)                 │
│  - Report Generation Orchestrator                            │
│  - Streaming результата                                      │
└─────────────────────────────────────────────────────────────┘
                            ↓ HTTP/gRPC
┌─────────────────────────────────────────────────────────────┐
│              LOCAL LLM RUNTIME (Ollama / vLLM)               │
│  Model: llama3.1:8b / mistral:7b / qwen2.5:14b              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Domain Models

#### LLMConfiguration

```csharp
public class LLMConfiguration
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public string ModelName { get; set; }
    public string SystemPrompt { get; set; }
    public string UserPromptTemplate { get; set; }
    public GenerationParameters GenerationParams { get; set; }
    public bool IsActive { get; set; }
    public int Version { get; set; }
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; }
}

public class GenerationParameters
{
    public double Temperature { get; set; } = 0.7;
    public double TopP { get; set; } = 0.9;
    public int TopK { get; set; } = 40;
    public int MaxTokens { get; set; } = 2048;
    public double RepeatPenalty { get; set; } = 1.1;
    public List<string> StopSequences { get; set; }
    public int? Seed { get; set; }
    public bool Stream { get; set; } = true;
}
```

#### GeneratedReport

```csharp
public class GeneratedReport
{
    public string Id { get; set; }
    public string UserId { get; set; }
    public string EntityId { get; set; }
    public string EntityType { get; set; }
    public string ReportType { get; set; }
    public string ConfigurationId { get; set; }
    public string Content { get; set; }
    public GenerationMetadata GenerationMetadata { get; set; }
    public bool IsEdited { get; set; }
    public int? UserRating { get; set; }
    public string UserFeedback { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

### 3.3 API Controllers

```csharp
[ApiController]
[Route("api/llm/reports")]
public class LLMReportController : ControllerBase
{
    [HttpPost("generate/stream")]
    public async Task GenerateReportStream([FromBody] GenerateReportRequest request)

    [HttpPost("generate")]
    public async Task<ActionResult<GenerateReportResponse>> GenerateReport([FromBody] GenerateReportRequest request)

    [HttpGet("{reportId}")]
    public async Task<ActionResult<GeneratedReport>> GetReport(string reportId)

    [HttpPut("{reportId}")]
    public async Task<ActionResult<GeneratedReport>> UpdateReport(string reportId, [FromBody] UpdateReportRequest request)

    [HttpPost("{reportId}/rate")]
    public async Task<IActionResult> RateReport(string reportId, [FromBody] RateReportRequest request)
}
```

### 3.4 Системный промпт (пример)

```json
{
  "name": "Профессиональный аналитик компаний",
  "systemPrompt": "Ты — профессиональный аналитик российских компаний с 15-летним опытом...\n\nПри составлении отчетов:\n1. Используй только предоставленные данные\n2. Структурируй ответ логично\n3. Выделяй ключевые риски\n4. Используй деловой язык\n...",
  "userPromptTemplate": "Подготовь {{reportType}} отчет о компании.\n\n## Данные:\n```json\n{{entityDataJson}}\n```",
  "generationParams": {
    "temperature": 0.3,
    "maxTokens": 4096
  }
}
```

### 3.5 Frontend Component

```vue
<template>
  <div class="ai-report-section">
    <button @click="generateReport">Сгенерировать отчет</button>

    <div v-if="isGenerating" class="streaming-content">
      <div v-html="renderMarkdown(streamedContent)"></div>
      <span class="cursor">▊</span>
    </div>

    <div v-else-if="currentReport" class="report-content">
      <div v-html="renderMarkdown(currentReport.content)"></div>
    </div>
  </div>
</template>
```

---

## 4. Картографический сервис

### 4.1 Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                    FRONTEND (Vue 3)                          │
│  Map Components (Leaflet + Yandex Maps API)                 │
└─────────────────────────────────────────────────────────────┘
                            ↓ REST API
┌─────────────────────────────────────────────────────────────┐
│                    GEOCODING SERVICE (C#)                    │
│  - Geocode addresses → coordinates                           │
│  - Batch geocoding, Cache management                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│              EXTERNAL GEOCODING APIS                         │
│  - Yandex Geocoder (приоритет)                              │
│  - Nominatim/OSM (fallback)                                  │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Domain Models

#### GeocodedAddress

```csharp
public class GeocodedAddress
{
    public string Id { get; set; }
    public string OriginalAddress { get; set; }
    public string NormalizedAddress { get; set; }
    public GeoCoordinates Coordinates { get; set; }
    public GeocodingPrecision Precision { get; set; }
    public string Provider { get; set; }
    public AddressComponents Components { get; set; }
    public GeoBounds Bounds { get; set; }
    public double Quality { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class GeoCoordinates
{
    public double Latitude { get; set; }
    public double Longitude { get; set; }

    public double DistanceTo(GeoCoordinates other) {
        // Haversine formula
    }
}
```

#### MapMarker

```csharp
public class MapMarker
{
    public string EntityId { get; set; }
    public string EntityType { get; set; }
    public string Inn { get; set; }
    public string Name { get; set; }
    public string Address { get; set; }
    public GeoCoordinates Coordinates { get; set; }
    public MarkerType Type { get; set; }
    public MarkerData Data { get; set; }
}
```

### 4.3 API Controllers

```csharp
[ApiController]
[Route("api/geocoding")]
public class GeocodingController : ControllerBase
{
    [HttpPost("geocode")]
    public async Task<ActionResult<GeocodedAddress>> GeocodeAddress([FromBody] GeocodeRequest request)

    [HttpPost("geocode/batch")]
    public async Task<ActionResult<BatchGeocodeResponse>> GeocodeBatch([FromBody] BatchGeocodeRequest request)

    [HttpPost("markers")]
    public async Task<ActionResult<List<MapMarker>>> GetMarkers([FromBody] GetMarkersRequest request)

    [HttpPost("clusters")]
    public async Task<ActionResult<List<MarkerCluster>>> GetClusters([FromBody] GetClustersRequest request)

    [HttpPost("heatmap")]
    public async Task<ActionResult<HeatmapData>> GetHeatmapData([FromBody] HeatmapRequest request)
}
```

### 4.4 Frontend Map Component

```vue
<template>
  <div class="map-viewer">
    <!-- Provider Selector -->
    <div class="provider-selector">
      <button @click="switchProvider('yandex')">Яндекс.Карты</button>
      <button @click="switchProvider('osm')">OpenStreetMap</button>
    </div>

    <!-- Yandex Maps -->
    <div v-show="activeProvider === 'yandex'" ref="yandexMapRef"></div>

    <!-- OpenStreetMap (Leaflet) -->
    <div v-show="activeProvider === 'osm'" ref="osmMapRef"></div>
  </div>
</template>

<script setup lang="ts">
import L from 'leaflet';
import 'leaflet.markercluster';

const yandexMap = ref<any>(null);
const osmMap = ref<L.Map | null>(null);

function initializeYandexMap() {
  yandexMap.value = new window.ymaps.Map(yandexMapRef.value, {
    center: [55.751244, 37.618423],
    zoom: 10
  });
}

function initializeOSMMap() {
  osmMap.value = L.map(osmMapRef.value).setView([55.751244, 37.618423], 10);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(osmMap.value);
}
</script>
```

---

## 5. Архитектура интерфейса

### 5.1 Структура Layout

```
┌─────────────────────────────────────────────────────────────┐
│                           HEADER                             │
│  [Logo] [Search] [Advanced] [Upload] [Theme] [Plan] [User]  │
└─────────────────────────────────────────────────────────────┘
┌────┬──────────────────────────────────────────────────┬──────┐
│ L  │                 MAIN CONTENT                     │  R   │
│ E  │  ┌────────────────────────────────────────────┐ │  I   │
│ F  │  │ Entity Card Header                         │ │  G   │
│ T  │  │ [Name] [INN] [Status] [Actions]            │ │  H   │
│    │  ├────────────────────────────────────────────┤ │  T   │
│ S  │  │ Carousel Navigation (scroll-sync)          │ │      │
│ I  │  │ [Сводка][Риски][Информация]...             │ │  S   │
│ D  │  └────────────────────────────────────────────┘ │  I   │
│ E  │  ┌────────────────────────────────────────────┐ │  D   │
│ B  │  │ Scrollable Content Sections                │ │  E   │
│ A  │  │ • AI Analysis                              │ │  B   │
│ R  │  │ • General Info                             │ │  A   │
│    │  │ • Risks                                    │ │  R   │
│    │  │ • Map                                      │ │      │
│    │  └────────────────────────────────────────────┘ │      │
└────┴──────────────────────────────────────────────────┴──────┘
```

### 5.2 Header Component

```vue
<template>
  <header class="app-header">
    <div class="header-left">
      <RouterLink to="/" class="logo-link">
        <img src="@/assets/logo.svg" alt="ПримаNEXT" />
        <span>ПримаNEXT</span>
      </RouterLink>

      <div class="search-wrapper">
        <input v-model="searchQuery" placeholder="Поиск..." @keydown.enter="handleSearch" />
      </div>

      <button @click="$emit('advanced-search')">Расширенный поиск</button>
      <button @click="$emit('upload-list')">Загрузить список</button>
    </div>

    <div class="header-right">
      <button @click="$emit('toggle-theme')">
        <SunIcon v-if="!isDarkTheme" />
        <MoonIcon v-else />
      </button>

      <div class="plan-badge">{{ subscriptionPlan?.name }}</div>

      <div class="user-menu">
        <UserAvatar :user="user" />
        <span>{{ user?.firstName }}</span>
      </div>
    </div>
  </header>
</template>
```

### 5.3 Left Sidebar

```vue
<template>
  <aside class="left-sidebar" :class="{ collapsed }">
    <nav class="sidebar-nav">
      <div v-for="item in navigationItems" :key="item.id" 
           :class="['nav-item', { active: activeView === item.id }]"
           @click="$emit('navigate', item.id)">
        <component :is="item.icon" />
        <span v-if="!collapsed">{{ item.label }}</span>
      </div>
    </nav>
  </aside>
</template>

<script setup lang="ts">
const navigationItems = [
  { id: 'search', label: 'Результаты поиска', icon: TableIcon },
  { id: 'dashboard', label: 'Дашборд', icon: ChartBarIcon },
  { id: 'graph', label: 'Граф связей', icon: NetworkIcon },
  { id: 'lists', label: 'Списки компаний', icon: ListIcon }
];
</script>
```

### 5.4 Right Sidebar

```vue
<template>
  <aside class="right-sidebar">
    <h3>Действия</h3>

    <button class="action-btn" @click="downloadEgrul">
      <DownloadIcon />
      Выписка из ЕГРЮЛ
    </button>

    <button class="action-btn" @click="generateReport">
      <FileTextIcon />
      Сводный отчет (PDF)
    </button>

    <button class="action-btn" @click="analyzeFinancials">
      <ChartIcon />
      Анализ РСБУ
    </button>
  </aside>
</template>
```

### 5.5 Entity Card с Carousel

```vue
<template>
  <div class="entity-card-view">
    <!-- Header -->
    <div class="entity-header">
      <h1>{{ entity?.companyName?.fullName }}</h1>
      <div class="entity-meta">
        <span>ИНН: {{ entity?.inn }}</span>
        <div :class="['status-badge', entity?.isActive ? 'active' : 'inactive']">
          {{ entity?.isActive ? 'Действующая' : 'Ликвидирована' }}
        </div>
      </div>

      <div class="entity-actions">
        <button @click="editEntity">Редактировать</button>
        <button @click="addComment">Комментировать</button>
        <button @click="toggleMonitoring">Мониторить</button>
      </div>
    </div>

    <!-- Carousel Navigation -->
    <div class="carousel-nav" :class="{ sticky: isCarouselSticky }">
      <button v-for="section in sections" :key="section.id"
              :class="['carousel-item', { active: activeSection === section.id }]"
              @click="scrollToSection(section.id)">
        <component :is="section.icon" />
        <span>{{ section.label }}</span>
      </button>
    </div>

    <!-- Content Sections -->
    <div class="entity-content" @scroll="handleScroll">
      <section id="section-ai"><AIReportSection /></section>
      <section id="section-info"><GeneralInfoSection /></section>
      <section id="section-risks"><RisksSection /></section>
      <section id="section-map"><EntityAddressMap /></section>
    </div>
  </div>
</template>

<script setup lang="ts">
// Intersection Observer для автоподсветки активной секции
function setupIntersectionObserver() {
  const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        activeSection.value = entry.target.id.replace('section-', '');
      }
    });
  }, { rootMargin: '-100px 0px -60% 0px' });

  // Observe all sections
}

function scrollToSection(sectionId: string) {
  const element = document.getElementById(`section-${sectionId}`);
  element?.scrollIntoView({ behavior: 'smooth' });
}
</script>
```

### 5.6 Theme System

```typescript
// useTheme.ts
export function useTheme() {
  const isDarkTheme = ref(getInitialTheme());

  function toggleTheme() {
    isDarkTheme.value = !isDarkTheme.value;
  }

  watch(isDarkTheme, (dark) => {
    if (dark) {
      document.documentElement.classList.add('dark-theme');
    } else {
      document.documentElement.classList.remove('dark-theme');
    }
    localStorage.setItem('theme', dark ? 'dark' : 'light');
  }, { immediate: true });

  return { isDarkTheme, toggleTheme };
}
```

```scss
// themes.scss
:root {
  --bg-primary: #ffffff;
  --bg-secondary: #f9fafb;
  --text-primary: #111827;
  --accent-color: #3b82f6;
}

.dark-theme {
  --bg-primary: #0f172a;
  --bg-secondary: #1e293b;
  --text-primary: #f1f5f9;
  --accent-color: #3b82f6;
}
```

---

## Технологический стек

### Backend
- **Framework**: .NET 8 (C#)
- **API Gateway**: Ocelot
- **Databases**: 
  - MongoDB (основные данные, workspace)
  - PostgreSQL (структурированные данные, тарифы, статистика)
  - Redis (кеш, sessions)
- **Auth**: Keycloak (OAuth2/OIDC)
- **Message Queue**: RabbitMQ
- **LLM Runtime**: Ollama / vLLM

### Frontend
- **Framework**: Vue 3 + TypeScript
- **State Management**: Pinia
- **Router**: Vue Router
- **UI Components**: Custom + Headless UI
- **Charts**: Chart.js / ECharts
- **Maps**: Leaflet + Yandex Maps API
- **HTTP Client**: Axios
- **Build Tool**: Vite

### DevOps
- **Containerization**: Docker + Docker Compose
- **Orchestration**: Kubernetes
- **CI/CD**: GitLab CI / GitHub Actions
- **Monitoring**: Prometheus + Grafana
- **Logging**: ELK Stack

---

## Заключение

Данный технический проект описывает полную архитектуру системы ПримаNEXT, включая:

1. **Административную панель** с управлением пользователями, тарифами и правами доступа
2. **Систему управления состоянием** с сохранением workspace, undo/redo, синхронизацией
3. **Интеграцию локальной LLM** для AI-генерации отчетов с настраиваемыми промптами
4. **Картографический сервис** с поддержкой двух провайдеров и геопространственным анализом
5. **Современный интерфейс** с carousel-навигацией, темами, responsive-дизайном

Все компоненты спроектированы с учетом масштабируемости, безопасности и удобства использования.

**Версия документа**: 1.0  
**Дата**: 2 декабря 2025  
**Статус**: Финальная версия
