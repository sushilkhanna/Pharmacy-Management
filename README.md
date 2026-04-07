<div align="center">

# 🏥 PharmaFlow

### *Microservices-Based Healthcare Management System*

> A production-grade **healthcare platform** built on a microservices architecture — covering **Role-Based Authentication**, **Doctor Appointment Management**, and a **Pharmacy E-Commerce System** with real-time inventory tracking via WebSockets. All services are secured with **stateless JWT authentication**.

</div>

---

## 📊 At a Glance

| Metric | Value |
|---|---|
| 🧩 Microservices | **3** (Auth · Appointment · Pharmacy) |
| 🔐 Auth Strategy | **JWT — Stateless** |
| 👥 User Roles | **Patient · Doctor · Admin · Pharmacist** |
| ⚡ Real-Time | **WebSocket** (pharmacy inventory updates) |
| 🗄️ Pharmacy DB Tables | **10** (products, batches, orders, payments…) |
| 📡 Total API Endpoints | **15+** |
| 🔄 Service Sync | **User sync endpoint** across services |

---

## 🏗️ System Architecture

```
                        ┌──────────────────────────────────┐
                        │           CLIENT / UI            │
                        └────────┬──────────┬──────────────┘
                                 │          │ WebSocket (ws://)
                        HTTP/JWT │          │
          ┌──────────────────────▼──────────▼──────────────────────┐
          │                   API Gateway (planned)                 │
          └──────────┬─────────────────┬──────────────────┬────────┘
                     │                 │                  │
          ┌──────────▼──────┐ ┌────────▼────────┐ ┌──────▼──────────────┐
          │  AUTH SERVICE   │ │ APPOINTMENT SVC │ │  PHARMACY SERVICE   │
          │   port: 8080    │ │   port: 8081    │ │     port: 8082      │
          │                 │ │                 │ │                     │
          │  • Signup/Login │ │  • Book appt    │ │  • Product catalog  │
          │  • JWT issue    │ │  • Doctor list  │ │  • Medicine batches │
          │  • Role mgmt    │ │  • Slot mgmt    │ │  • Order processing │
          │                 │ │  • My patients  │ │  • Stock movements  │
          │                 │ │  • User sync ←──┼─┼── syncs on signup   │
          └──────────┬──────┘ └────────┬────────┘ └──────┬──────────────┘
                     │                 │                  │
          ┌──────────▼─────────────────▼──────────────────▼────────┐
          │                      Database Layer                     │
          │          (Each service owns its own database)           │
          └─────────────────────────────────────────────────────────┘

  ✅ Auth + Appointment → Already joined as microservices
  🔄 Pharmacy → Standalone microservice (integration in progress)
```

---

## 🔐 Service 1 — Authentication (`port 8080`)

Handles user registration, login, and JWT token issuance. All other services validate this token to authorize requests.

**Security Model:** Stateless JWT — no session stored server-side. Every request carries the token; the service validates it on each call.

### Endpoints

| Method | Endpoint | Access | Description |
|---|---|---|---|
| `POST` | `/api/auth/signup` | Public | Register a new user with role |
| `POST` | `/api/auth/login` | Public | Login and receive JWT token |

### Example — Signup
```http
POST /api/auth/signup
Content-Type: application/json

{
  "name": "Dr. Sarah Khan",
  "email": "sarah@clinic.com",
  "password": "securePass123",
  "role": "DOCTOR"
}
```
```json
{ "message": "Successfully Registered" }
```

### Example — Login
```http
POST /api/auth/login
Content-Type: application/json

{ "email": "sarah@clinic.com", "password": "securePass123" }
```
```json
{ "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." }
```

> All subsequent requests must include: `Authorization: Bearer <token>`

---

## 📅 Service 2 — Doctor Appointment (`port 8081`)

Manages the full appointment lifecycle — patients book slots, doctors manage their availability and view their daily patients list. Identity is always derived from the JWT, never passed as a parameter.

### Endpoints

| Method | Endpoint | Role | Description |
|---|---|---|---|
| `POST` | `/api/appointments/book` | Patient | Book appointment with a doctor |
| `GET` | `/api/appointments/doctorList` | Any | List all available doctors |
| `GET` | `/api/appointments/doctor/by-specialization/{spec}` | Any | Filter doctors by specialization |
| `GET` | `/api/appointments/my-appointments` | Patient | View own appointments (JWT-scoped) |
| `PUT` | `/api/doctors/availability` | Doctor | Update available time slots |
| `GET` | `/api/doctors/my-patients` | Doctor | View patients scheduled for a date |
| `POST` | `/api/users/sync` | Internal | Sync user from Auth service |

### Example — Book Appointment
```http
POST /api/appointments/book?doctorId=3&date=2025-08-10&time=10:00
Authorization: Bearer <token>
```
```json
{
  "appointmentId": 42,
  "status": "CONFIRMED",
  "date": "2025-08-10",
  "time": "10:00"
}
```

### Security Highlights
- `GET /my-appointments` — patient identity extracted from JWT, not from a passed parameter. This prevents a patient from accessing another patient's records.
- `GET /my-patients` — doctor identity extracted from JWT, scoped to their own patient list only.
- `PUT /availability` — only callable by users with the `DOCTOR` role.

---

## 💊 Service 3 — Pharmacy Management (`port 8082`)

A full-featured pharmacy e-commerce backend with real-time inventory tracking via **WebSockets**. Currently operates as a standalone microservice — integration with the Auth/Appointment cluster is in progress.

### Real-Time Updates (WebSocket)
Stock movements and inventory changes are broadcast live to connected clients. No need to poll the API — the moment a batch is deducted or restocked, all subscribed clients receive the update instantly.

```
ws://localhost:8082/ws/pharmacy
  └── /topic/inventory   → live stock movement events
  └── /topic/orders      → live order status updates
```

### Database Schema (10 Tables)

The pharmacy module is designed around batch-tracked medicine management — every product is linked to medicine batches, and every stock change is recorded as a movement.

```
USERS ──────────────── ORDERS ──────────── PAYMENTS
                          │
                    ORDER_ITEMS
                       │      │
                  PRODUCTS   MEDICINE_BATCHES ── STOCK_MOVEMENTS
                    │    │
              CATEGORIES  SUPPLIERS

PRESCRIPTIONS ──────────────────────────── ORDERS
```

| Table | Purpose |
|---|---|
| `USERS` | Customers with roles (patient, admin, pharmacist) |
| `CATEGORIES` | Hierarchical medicine categories (supports subcategories via `parent_id`) |
| `SUPPLIERS` | Supplier directory with contact info |
| `PRODUCTS` | Medicine catalog with pricing, dosage form, strength, prescription flag |
| `MEDICINE_BATCHES` | Batch-level tracking — manufacture date, expiry, cost price, quantity |
| `STOCK_MOVEMENTS` | Full audit trail of every inventory in/out movement |
| `PRESCRIPTIONS` | Patient prescriptions with doctor, hospital, validity, image |
| `ORDERS` | Customer orders linked to user and optionally a prescription |
| `ORDER_ITEMS` | Individual items per order with batch reference and unit price |
| `PAYMENTS` | Payment records with method, status, transaction ID |

### Key Features
- **Batch-level stock tracking** — expiry dates, manufacture dates, cost price per batch
- **Prescription-gated products** — `requires_prescription` flag on products; orders validated against uploaded prescriptions
- **Full audit trail** — every stock movement (type, quantity, reason, timestamp) is recorded in `STOCK_MOVEMENTS`
- **Hierarchical categories** — categories support parent-child nesting via `parent_id`

---

## 🛠️ Tech Stack

| | Technology |
|---|---|
| **Language** | Java 21 |
| **Framework** | Spring Boot 4.x |
| **Security** | Spring Security + JWT (Stateless) |
| **Real-Time** | WebSocket (STOMP) |
| **Architecture** | Microservices |
| **Build Tool** | Maven |
| **API Style** | REST (JSON) |

---

## ⚙️ Getting Started

**Prerequisites:** Java 21+, Maven 3.8+, Git, a running database (MySQL / PostgreSQL)

```bash
# Clone the repository
git clone https://github.com/your-username/clinicflow.git
cd clinicflow

# Start Auth Service (port 8080)
cd auth-service
mvn spring-boot:run

# Start Appointment Service (port 8081)
cd ../appointment-service
mvn spring-boot:run

# Start Pharmacy Service (port 8082)
cd ../pharmacy-service
mvn spring-boot:run
```

### Making an Authenticated Request

```bash
# 1. Login to get token
TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@clinic.com","password":"pass"}' | jq -r '.token')

# 2. Use token on any protected endpoint
curl http://localhost:8081/api/appointments/my-appointments \
  -H "Authorization: Bearer $TOKEN"
```

---

## 🗺️ Roadmap

- [x] JWT stateless authentication with role-based access
- [x] Doctor appointment booking and management
- [x] User sync between Auth and Appointment services
- [x] Pharmacy CRUD with batch tracking and prescription validation
- [x] Real-time inventory updates via WebSocket
- [ ] 🔄 Integrate Pharmacy service into the microservices cluster
- [ ] API Gateway (Spring Cloud Gateway)
- [ ] Service discovery (Eureka)
- [ ] Centralized config server


Built with ❤️ using Spring Boot · WebSocket · JWT &nbsp;·&nbsp; ⭐ Star this repo if it helped you!

</div>
