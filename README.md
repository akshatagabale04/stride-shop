# stride-shop
An online running-shoe store built as a microservices e-commerce system. A React storefront talks to an API gateway, which routes to small Spring Boot services. Each service owns its own PostgreSQL database, and RabbitMQ sends the order receipt asynchronously. The project follows an "E-Commerce Order Flow Architecture" reference.

Tech stack: React (Vite) · Java 17 · Spring Boot 3.3 · Spring Cloud Gateway · PostgreSQL 16 · RabbitMQ · Docker Compose · Maven multi-module

<!-- Add screenshots after you run the app: ![Shop](docs/shop.png) ![Cart](docs/cart.png) -->
Features
Browse 8 seeded shoes and filter by category (running, trail, casual, training)
Cart with quantities and a checkout using an email address
Order flow: order created as PENDING, payment charged, stock reserved, order CONFIRMED
No overselling: stock is reserved with an atomic conditional update (WHERE on_hand >= qty)
Failure handling: a declined payment or out-of-stock item ends the order as FAILED with a reason
Receipt email sent in the background through RabbitMQ, so the shopper never waits for it

Architecture
React UI (:3000) -> API Gateway (:8080) -> catalog-service (:8081) -> catalog_db
                                        -> order-service   (:8082) -> order_db
                                                |-> payment-service   (:8083) -> payment_db
                                                |-> inventory-service (:8084) -> inventory_db
                                                '--publishes order.confirmed--> RabbitMQ --> notification-service (:8085)
Service	Port	Owns	Role
frontend (React)	3000	none	Product grid, cart, checkout, confirmation
gateway	8080	none	Single public entry for /api/products and /api/orders, plus CORS
catalog-service	8081	catalog_db	Products, with filtering by category
order-service	8082	order_db	Runs the checkout steps and publishes order.confirmed
payment-service	8083	payment_db	Mock charge that returns PAID or FAILED
inventory-service	8084	inventory_db	Atomic stock reserve and release
notification-service	8085	none	Consumes order.confirmed and logs the receipt email

Golden rule: no service reads another service's database. They talk through REST calls or RabbitMQ events.

What happens when you click "Place order"
POST /api/orders reaches the gateway, which forwards it to the order service.
The order service saves the order as PENDING in order_db.
The payment service charges the total and returns PAID or FAILED.
The inventory service reserves stock for each item. If any item is short, the order fails.
The order becomes CONFIRMED and the order is returned to the browser.
An order.confirmed event goes to RabbitMQ, and the notification service logs the receipt.
Getting started

Option 1: Docker (everything in one command)
Requires Docker Desktop.

bash
docker compose up --build
Store: http://localhost:3000
RabbitMQ dashboard: http://localhost:15672 (login guest / guest)
Watch the receipt email: docker compose logs -f notification-service
Option 2: Develop from your PC

Requires Java 17, Maven 3.9+ and Node 20+.

In docker-compose.yml, publish the ports the services expect on your machine: add ports: [ "5432:5432" ] to postgres and "5672:5672" to the rabbitmq ports. Then:

bash
docker compose up postgres rabbitmq -d
mvn -q -DskipTests install

# one terminal per service
cd catalog-service && mvn spring-boot:run
cd inventory-service && mvn spring-boot:run
cd payment-service && mvn spring-boot:run
cd order-service && mvn spring-boot:run
cd notification-service && mvn spring-boot:run
cd gateway && mvn spring-boot:run

cd frontend && npm install && npm run dev   # http://localhost:3000

Services connect to localhost:5432 by default (user and password shop). If port 5432 is taken by a local PostgreSQL, use another left-hand port such as "5433:5432" and set DB_URL for each service.

Try the failure paths
Out of stock: each product starts with 25 units. Order 26 of one shoe.
Payment declined: make the cart total go above ₹5,00,000 (for example 6 × Tempo Carbon).
API reference

Everything goes through the gateway on port 8080. Amounts are in paise (100 paise = ₹1).

Method	Endpoint	Purpose
GET	/api/products?category=	List products
GET	/api/products/{id}	One product
POST	/api/orders	Place an order
GET	/api/orders/{id}	One order
GET	/api/orders?email=	Orders for an email, newest first
Project structure
gateway/               Spring Cloud Gateway routes
catalog-service/       Products
order-service/         Checkout orchestration, event publishing
payment-service/       Mock payments
inventory-service/     Stock with atomic reserve and release
notification-service/  RabbitMQ consumer
frontend/              React storefront
db/init.sql            Creates the four databases
docker-compose.yml
Not included yet

JWT login at the gateway, a user service and server-side cart (the cart lives in the browser), a real payment provider, refunds, the outbox pattern for reliable events, and automated tests.
