# System Architecture Decisions

## 📋 Tổng quan

Tài liệu này ghi lại các quyết định kiến trúc quan trọng trong quá trình phát triển Video View Count System.

## 🏗️ Architecture Overview

### **Current Architecture:**

```
User → API Gateway → View Count Service →
├── Update in-memory cache (immediate)
├── Write to MongoDB (raw view event)
└── Send message to Analytics Service →
    ├── Process & aggregate data from MongoDB
    ├── Update SQL Server (analytics + view count)
    └── Generate statistics
```

## 🔄 Data Flow Decisions

### **1. Database Strategy**

#### **MongoDB cho Raw Data**

- **Lý do:** High write throughput, flexible schema
- **Trade-off:** Eventual consistency vs ACID
- **Decision:** Chọn MongoDB cho view events

#### **SQL Server cho Aggregated Data**

- **Lý do:** ACID compliance, complex queries
- **Trade-off:** Performance vs Consistency
- **Decision:** Chọn SQL Server cho analytics

#### **IMemoryCache cho Fast Reads**

- **Lý do:** Sub-millisecond response time
- **Trade-off:** Memory usage vs Performance
- **Decision:** Chọn in-memory cache

### **2. Processing Strategy**

#### **Background Service vs Real-time Processing**

- **Current:** Background service mỗi 15 phút
- **Issue:** Data duplication
- **Decision:** Tạm thời giữ nguyên, plan incremental processing

#### **Batch Processing vs Event-driven**

- **Current:** Batch processing
- **Alternative:** Event-driven với message queue
- **Decision:** Bắt đầu với batch, migrate sau

## 🚨 Challenges & Solutions

### **Challenge 1: Data Duplication in Analytics**

#### **Problem:**

```csharp
// Mỗi 15 phút xử lý rolling 24h window
var last24Hours = DateTime.UtcNow.AddHours(-24);
// → Data bị xử lý nhiều lần
```

#### **Solutions Considered:**

1. **Incremental Processing** - Track last processing time
2. **Daily Processing** - Chỉ xử lý 1 lần/ngày
3. **Hourly Processing** - Xử lý mỗi giờ
4. **Smart Batching** - Chỉ xử lý video có thay đổi
5. **Event-driven** - Xử lý ngay khi có event

#### **Decision:** Hybrid approach (Incremental + Hourly)

### **Challenge 2: Database Connection Issues**

#### **Problem:**

- LocalDB không support trên macOS
- Connection string cần thay đổi

#### **Solution:**

```json
// Before
"DefaultConnection": "Server=(localdb)\\mssqllocaldb;..."

// After
"DefaultConnection": "Server=localhost;Database=VideoViewCount;User=sa;Password=MyStrongPass123;TrustServerCertificate=true;"
```

### **Challenge 3: Circular Dependencies**

#### **Problem:**

- Infrastructure project reference Services project
- Services project reference Infrastructure project

#### **Solution:**

- Move service registration to API project
- Create separate extension methods
- Break circular dependency

## 📊 Technology Stack Decisions

### **Backend Framework**

- **Choice:** .NET Core 8
- **Reason:** Cross-platform, high performance, rich ecosystem

### **Databases**

- **MongoDB:** Raw view events
- **SQL Server:** Aggregated analytics
- **IMemoryCache:** Fast reads

### **ORM**

- **Entity Framework Core:** SQL Server
- **MongoDB.Driver:** MongoDB

### **Background Processing**

- **BackgroundService:** Built-in .NET service
- **Alternative:** Hangfire, Quartz.NET

## 🔧 Implementation Decisions

### **1. Project Structure**

```
VideoViewCountSystem/
├── VideoViewCountSystem.API/          # Web API
├── VideoViewCountSystem.Core/         # Business logic & models
├── VideoViewCountSystem.Infrastructure/ # Database & external services
├── VideoViewCountSystem.Services/     # View Count & Analytics services
└── VideoViewCountSystem.Shared/       # Common utilities
```

### **2. Dependency Injection**

- **Pattern:** Constructor injection
- **Registration:** Extension methods
- **Lifetime:** Scoped for services, Singleton for contexts

### **3. Error Handling**

- **Strategy:** Try-catch with logging
- **Logging:** ILogger with structured logging
- **Monitoring:** Console logs (production: Application Insights)

### **4. Caching Strategy**

- **Cache Key:** `viewcount_{videoId}`
- **Expiration:** 5 minutes
- **Invalidation:** On new view

## 🚀 Future Considerations

### **Scaling Decisions**

1. **Horizontal Scaling:** Load balancer + multiple instances
2. **Database Sharding:** MongoDB sharding by videoId
3. **Caching:** Redis for distributed cache
4. **Message Queue:** RabbitMQ/Azure Service Bus

### **Performance Optimizations**

1. **Database Indexing:** Optimize MongoDB queries
2. **Connection Pooling:** Configure EF Core
3. **Async Processing:** Background services
4. **CDN:** Static content delivery

### **Monitoring & Observability**

1. **Application Insights:** Performance monitoring
2. **Health Checks:** Database connectivity
3. **Metrics:** View count, processing time
4. **Alerting:** Error rates, performance degradation

## 📝 Lessons Learned

### **What Worked Well:**

- Clean architecture separation
- Dependency injection pattern
- Async/await throughout
- Comprehensive logging

### **What Could Be Better:**

- Data duplication in analytics
- Error handling could be more robust
- Missing health checks
- No automated testing

### **Next Improvements:**

1. Fix analytics data duplication
2. Add comprehensive testing
3. Implement health checks
4. Add monitoring dashboard

---

**Last Updated:** 2024-01-18  
**Status:** Active  
**Review Cycle:** Monthly
