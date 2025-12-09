# **Software Requirements Specification (SRS)**

# **Mini E-Commerce Microservices System**

---

## **1. Introduction**

### **1.1 Purpose**

This Software Requirements Specification (SRS) defines the functional and non-functional requirements for the Mini E-Commerce Microservices System. The objective is to build an end-to-end, production-grade microservices ecosystem consisting of:

* **Auth Service**
* **Product Service**
* **Order Service**
* **Payment Service**
* **API Gateway**
* **Service Registry**
* **Config Server**
* **Databases + Redis**

This SRS simulates real-world enterprise documentation.

### **1.2 Scope**

The system allows customers to:

* Register and log in
* Browse products
* Place orders
* Make payments
* Track order status

The system supports internal workflows such as product management, order validation, payment processing, and inventory deduction.

### **1.3 Definitions & Acronyms**

* **JWT** – JSON Web Token
* **API** – Application Programming Interface
* **RBAC** – Role-Based Access Control
* **DB** – Database
* **NFR** – Non-Functional Requirements

---

## **2. Overall Description**

### **2.1 Product Perspective**

The system follows a **distributed microservices architecture**. All services are independent, containerized, and communicate using REST APIs.

### **2.2 Product Features Overview**

1. **Authentication & Authorization**
2. **Product Management**
3. **Order Placement & Tracking**
4. **Payment Processing**
5. **Inventory Reservation & Deduction**
6. **API Gateway Routing & Security**
7. **Service Discovery & Configuration Management**

### **2.3 User Classes & Characteristics**

| User Type       | Permissions                          |
| --------------- | ------------------------------------ |
| Customer        | Browse products, order, pay          |
| Admin           | Manage products                      |
| System Services | Authenticated internal communication |

### **2.4 Operating Environment**

* JDK 17
* Spring Boot 3+
* Docker
* Kubernetes (optional)
* MySQL
* Redis
* PostgreSQL (optional)
* Linux-based deployment

---

## **3. Functional Requirements**

# **3.1 Auth Service Requirements**

### **3.1.1 Register User**

* System shall allow new users to register using email & password.
* Password shall be stored using BCrypt hashing.

### **3.1.2 Login User**

* System shall validate credentials.
* System shall return an access token (JWT) and refresh token.

### **3.1.3 Token Refresh**

* System shall issue new access tokens using valid refresh tokens.

### **3.1.4 Role-Based Access Control**

* System shall differentiate between Admin and Customer.

---

# **3.2 Product Service Requirements**

### **3.2.1 Add Product (Admin)**

* Admin can create products with name, price, stock, description.

### **3.2.2 Update Product (Admin)**

* Admin can update product details and stock.

### **3.2.3 Get Product List**

* System shall return paginated product lists.

### **3.2.4 Reserve Inventory** (internal API)

* Should reserve products during order creation.

### **3.2.5 Release Inventory**

* Should release reserved items on order cancellation.

---

# **3.3 Order Service Requirements**

### **3.3.1 Create Order**

* System shall validate product availability.
* System shall create order in PENDING status.

### **3.3.2 Update Order Status**

* Shall update after payment outcome.

### **3.3.3 Get Order Details**

* Users should view their order history.

### **3.3.4 Integrate With Payment Service**

* System shall send payment request to Payment Service.

---

# **3.4 Payment Service Requirements**

### **3.4.1 Process Payment**

* Simulate payment success/failure.
* Ensure idempotency via payment reference.

### **3.4.2 Notify Order Service**

* Update order status after payment.

### **3.4.3 Record Payment Transactions**

* Store all transactions for audit.

---

# **3.5 API Gateway Requirements**

### **3.5.1 Route Requests**

* Should forward client traffic to underlying services.

### **3.5.2 JWT Validation**

* Reject requests without valid JWT.

---

# **3.6 Service Registry Requirements**

### **3.6.1 Service Discovery**

* All services shall register and discover each other.

---

# **3.7 Config Server Requirements**

### **3.7.1 Central Config**

* Store environment configs externally.

---

## **4. Non-Functional Requirements (NFRs)**

### **4.1 Performance Requirements**

* Auth endpoints should respond within **< 150ms**.
* Product listing should respond within **< 300ms**.
* Orders & payments should complete in **< 3 seconds**.

### **4.2 Security Requirements**

* All internal services must validate JWT from Auth Service.
* Admin endpoints require role "ADMIN".
* All passwords must be encrypted.

### **4.3 Availability Requirements**

* System should maintain 99% uptime.
* Services must handle failure gracefully.

### **4.4 Scalability Requirements**

* Services must be stateless.
* Horizontal scaling via Kubernetes.

### **4.5 Data Integrity Requirements**

* Orders must be ACID-compliant.
* Inventory reservation must be atomic.

### **4.6 Logging & Monitoring**

* Provide logs for every event.
* Use centralized logging (ELK, Grafana, etc.).

---

## **5. System Models**

(Detailed LLD diagrams are already included in separate internal service documents.)

* Architecture Diagram
* ERD Diagram
* Sequence Diagrams
* Class Diagrams

---

## **6. External Interface Requirements**

### **6.1 User Interfaces**

(Not part of backend scope; optionally web UI can be added.)

### **6.2 API Interfaces**

* All APIs follow REST conventions.
* JSON request/response format.

### **6.3 Hardware Interfaces**

* Runs on cloud servers or on-premise Linux.

---

## **7. System Constraints**

* Must be containerized using Docker.
* DB consistency must be maintained.
* Services must not share databases.
* Payment processing must be idempotent.

---

## **8. Assumptions & Dependencies**

* Internet connectivity exists for requests.
* Payment is simulated (no real gateway).
* Microservices run on separate ports.

---

## **9. Acceptance Criteria**

1. User can register, log in, and get JWT.
2. Product listing works.
3. Orders can be created and tracked.
4. Payment workflow updates order status.
5. All services run independently.
6. API gateway routes correctly.
7. System works end-to-end in Docker Compose.

---

## **10. Future Enhancements**

* Real Payment Gateway (Stripe, Razorpay)
* Kafka event-driven architecture
* Cart Service
* Search Service with Elasticsearch
* Notification Service
* Admin Dashboard

---

## **11. Conclusion**

This SRS defines the complete functional and non-functional expectations for the Mini E-Commerce System. It acts as the master blueprint for development, testing, deployment, and future expansion.
