
---

# **📅 DAY 1 — FOUNDATIONS + CORE BUILDING BLOCKS**

## **1. What is NestJS? (30 mins)**
- Node.js framework built on TypeScript  
- Inspired by Angular architecture  
- Uses decorators, DI, modules  
- Opinionated but scalable  

**Key idea:** NestJS = Express/Fastify + TypeScript + Architecture

---

## **2. Project Structure (30 mins)**
Understand:
- `main.ts` (bootstrap)  
- `app.module.ts`  
- Controllers  
- Providers  
- Modules  

**Mental model:**  
Module → Controller → Service → Repository

---

## **3. Modules (45 mins)**
- `@Module()` decorator  
- Imports / Providers / Exports  
- Feature modules  
- Shared modules  

**Goal:** Understand how Nest organizes code.

---

## **4. Controllers (1 hour)**
- `@Controller()`  
- Route handlers (`@Get`, `@Post`, etc.)  
- Route params (`@Param`, `@Query`, `@Body`)  
- Status codes  
- DTOs  

**Practice:** Build CRUD endpoints.

---

## **5. Services & Dependency Injection (1 hour)**
- `@Injectable()`  
- DI container  
- Provider scopes  
- Constructor injection  

**Goal:** Understand Nest’s DI system deeply.

---

## **6. Pipes (45 mins)**
- ValidationPipe  
- TransformationPipe  
- Custom pipes  

**Key:** Use `class-validator` + DTOs.

---

## **7. Guards (45 mins)**
- `CanActivate`  
- Auth guards  
- Role-based access  

**Use case:** JWT Auth.

---

## **8. Interceptors (45 mins)**
- Logging  
- Transforming responses  
- Timeout interceptor  
- Caching interceptor  

**Key:** Interceptors wrap request/response.

---

## **9. Filters (30 mins)**
- Exception filters  
- Global filters  
- Custom error handling  

---

## **10. Providers Deep Dive (30 mins)**
- Factory providers  
- Class providers  
- Value providers  
- Injection tokens  

---

### **End of Day 1 Outcome**
You can build:
- A full CRUD API  
- With validation  
- With authentication  
- With clean architecture  

---

# **📅 DAY 2 — ADVANCED + PRODUCTION + MICROSERVICES**

## **11. Database Integration (1 hour)**
Choose one:
- TypeORM  
- Prisma  
- Mongoose  

Learn:
- Entities / Models  
- Repositories  
- Relations  
- Migrations  

---

## **12. Configuration Management (30 mins)**
- `@nestjs/config`  
- `.env`  
- ConfigService  
- Environment-based configs  

---

## **13. Middleware (30 mins)**
- Logging  
- Request shaping  
- Third-party middleware (morgan, helmet)  

---

## **14. Caching (30 mins)**
- In-memory cache  
- Redis cache  
- Cache interceptors  

---

## **15. Authentication & Authorization (1 hour)**
- JWT strategy  
- Passport.js  
- Refresh tokens  
- Role-based guards  

---

## **16. File Uploads (30 mins)**
- Multer  
- Streaming uploads  
- Validation  

---

## **17. WebSockets (45 mins)**
- `@WebSocketGateway()`  
- Rooms  
- Events  
- Real-time updates  

---

## **18. Microservices (1.5 hours)**
NestJS supports:
- TCP  
- Redis  
- NATS  
- Kafka  
- RabbitMQ  
- gRPC  

Learn:
- Message patterns  
- Event patterns  
- Client proxies  
- Transport layer  
- Microservice architecture  

---

## **19. Inter-service Communication Patterns (45 mins)**
- Request-response  
- Event-driven  
- Pub/sub  
- Sagas  
- Outbox pattern  

---

## **20. Testing (45 mins)**
- Unit tests  
- E2E tests  
- Testing controllers/services  
- Mocking providers  

---

## **21. Deployment (45 mins)**
- Docker  
- PM2  
- CI/CD  
- Environment configs  
- Scaling  

---

# **🔥 Final Outcome After 2 Days**
You will understand:

### **Core NestJS**
- Modules  
- Controllers  
- Services  
- DI  
- Pipes  
- Guards  
- Interceptors  
- Filters  

### **Advanced**
- Auth  
- Database  
- Caching  
- WebSockets  
- Testing  

### **Microservices**
- Message patterns  
- Event-driven architecture  
- Kafka/NATS/RMQ  
- Distributed workflows  

### **Production**
- Logging  
- Config  
- Deployment  
- Error handling  

---
