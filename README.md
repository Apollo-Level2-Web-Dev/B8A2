# 🎉 EventHive – Assignment 2 Requirements Specification

> **Event & Ticket Booking Platform**
>
> *A scalable, modular backend platform for managing community events, ticket reservations, seat allocations, dynamic pricing, role-based event organization, and organizer analytics.*

---

## 🛠️ Technology Stack & Architectural Rules

| Technology / Concept | Requirement & Note |
| :--- | :--- |
| **Node.js** | LTS runtime (20.x / 24.x or higher) |
| **TypeScript** | Strict mode enabled (`noImplicitAny`), latest stable version |
| **Express.js** | Feature-based (modular) & Layered architecture (`route` → `controller` → `service` → `repository`) |
| **Express Version** | **Express 5.x** (Native support for async error catching without repetitive try/catch wrappers or explicit `catchAsync` wrappers) |
| **PostgreSQL** | Relational database using native `pg` (`Pool`) driver only. **No ORMs (Prisma/TypeORM) or Query Builders (Knex) allowed.** |
| **Raw SQL & Transactions** | Parameterized queries (`pool.query($1, $2)`) mandatory for every database operation to completely prevent SQL injection. Transactions (`BEGIN`, `COMMIT`, `ROLLBACK`) and row-level concurrency control (`SELECT ... FOR UPDATE`) must be handled inside the **Service layer** using a dedicated client acquired via `pool.connect()`. |
| **bcrypt** | Password hashing (salt rounds between 8 and 12) |
| **jsonwebtoken** | JWT generation & verification |
| **Zod** | Request body and query parameter validation via custom validation middleware |
| **Error Handling** | Centralized error-handling middleware utilizing a custom `AppError` class |

> 📌 **Note on Record IDs:** Identifiers (`id`) across all tables can be implemented as either auto-incrementing serial integers (`SERIAL PRIMARY KEY`) or standard UUIDs/CUIDs.

---

## 👥 User Roles & Permissions

| Role | Allowed Actions |
| :--- | :--- |
| **attendee** | • Register and log in<br>• Browse, search, and filter published events<br>• Book tickets for events (with automatic available seat validation & price calculation)<br>• View own booking history & cancel bookings |
| **organizer** | • All attendee permissions<br>• Create, update, and delete their own organized events<br>• View all ticket bookings across their specific organized events |

---

## 🗄️ Database Schema Design

### Table 1: `users`
| Field | Data Type / Constraint | Description / Rule |
| :--- | :--- | :--- |
| `id` | `SERIAL PRIMARY KEY` / UUID | Unique identifier for each account |
| `name` | `VARCHAR(150) NOT NULL` | Full display name of the user |
| `email` | `VARCHAR(150) UNIQUE NOT NULL` | Unique login email address |
| `password` | `VARCHAR(255) NOT NULL` | Hashed password string (**never returned in responses**) |
| `role` | `VARCHAR(30) DEFAULT 'attendee' NOT NULL` | Access control role (`attendee` or `organizer`) |
| `created_at` | `TIMESTAMP DEFAULT NOW()` | Account creation timestamp |
| `updated_at` | `TIMESTAMP DEFAULT NOW()` | Last update timestamp |

### Table 2: `events`
| Field | Data Type / Constraint | Description / Rule |
| :--- | :--- | :--- |
| `id` | `SERIAL PRIMARY KEY` / UUID | Unique identifier for each event |
| `title` | `VARCHAR(200) NOT NULL` | Headline title of the event |
| `description` | `TEXT NOT NULL` | Detailed description of the event |
| `category` | `VARCHAR(100) NOT NULL` | Category tag (e.g., `tech`, `music`, `workshop`, `sports`) |
| `date` | `TIMESTAMP NOT NULL` | Scheduled future date and time |
| `location` | `VARCHAR(255) NOT NULL` | Venue location or online link |
| `total_seats` | `INTEGER NOT NULL` | Maximum ticket capacity |
| `available_seats` | `INTEGER NOT NULL` | Current available seats (initialized equal to `total_seats` on create) |
| `price` | `DECIMAL(10, 2) NOT NULL DEFAULT 0.00` | Price per single ticket |
| `organizer_id` | `INTEGER REFERENCES users(id) ON DELETE CASCADE` | Foreign key referencing the organizer |
| `created_at` | `TIMESTAMP DEFAULT NOW()` | Event creation timestamp |
| `updated_at` | `TIMESTAMP DEFAULT NOW()` | Event update timestamp |

### Table 3: `bookings`
| Field | Data Type / Constraint | Description / Rule |
| :--- | :--- | :--- |
| `id` | `SERIAL PRIMARY KEY` / UUID | Unique identifier for each booking |
| `event_id` | `INTEGER REFERENCES events(id) ON DELETE CASCADE` | Foreign key referencing the booked event |
| `attendee_id` | `INTEGER REFERENCES users(id) ON DELETE CASCADE` | Foreign key referencing the booking user |
| `tickets_booked` | `INTEGER NOT NULL` | Number of seats reserved (1 to 10 max per request) |
| `total_price` | `DECIMAL(10, 2) NOT NULL` | Automatically calculated as `tickets_booked * event.price` |
| `status` | `VARCHAR(20) DEFAULT 'confirmed' NOT NULL` | Booking state (`confirmed` or `cancelled`) |
| `created_at` | `TIMESTAMP DEFAULT NOW()` | Booking creation timestamp |
| `updated_at` | `TIMESTAMP DEFAULT NOW()` | Booking update timestamp |

---

## 🌐 API Endpoints Specification (11 Endpoints)

### 🔹 Authentication Module

#### 1. User Registration
* **Endpoint:** `POST /api/v1/auth/signup`
* **Access:** Public
* **Request Body (Zod Validated):**
  ```json
  {
    "name": "Sarah Connor",
    "email": "sarah@eventhive.com",
    "password": "securePassword123",
    "role": "organizer"
  }
  ```
* **Success Response (201 Created):**
  ```json
  {
    "success": true,
    "message": "User registered successfully",
    "data": {
      "id": 1,
      "name": "Sarah Connor",
      "email": "sarah@eventhive.com",
      "role": "organizer",
      "created_at": "2026-11-15T09:00:00.000Z",
      "updated_at": "2026-11-15T09:00:00.000Z"
    }
  }
  ```

#### 2. User Login
* **Endpoint:** `POST /api/v1/auth/login`
* **Access:** Public
* **Request Body:**
  ```json
  {
    "email": "sarah@eventhive.com",
    "password": "securePassword123"
  }
  ```
* **Success Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Login successful",
    "data": {
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "user": {
        "id": 1,
        "name": "Sarah Connor",
        "email": "sarah@eventhive.com",
        "role": "organizer"
      }
    }
  }
  ```

---

### 🔹 Events Module

#### 3. Create Event
* **Endpoint:** `POST /api/v1/events`
* **Access:** Protected (`organizer` role only)
* **Headers:** `Authorization: Bearer <JWT_TOKEN>` (or `Authorization: <JWT_TOKEN>`)
* **Request Body:**
  ```json
  {
    "title": "TypeScript Advanced Architecture",
    "description": "Deep dive into enterprise TS patterns and modular design.",
    "category": "tech",
    "date": "2026-12-10T10:00:00.000Z",
    "location": "Dhaka Tech Hub",
    "total_seats": 50,
    "price": 1500.00
  }
  ```
* **Success Response (201 Created):**
  ```json
  {
    "success": true,
    "message": "Event created successfully",
    "data": {
      "id": 1,
      "title": "TypeScript Advanced Architecture",
      "description": "Deep dive into enterprise TS patterns and modular design.",
      "category": "tech",
      "date": "2026-12-10T10:00:00.000Z",
      "location": "Dhaka Tech Hub",
      "total_seats": 50,
      "available_seats": 50,
      "price": "1500.00",
      "organizer_id": 1,
      "created_at": "2026-11-15T10:30:00.000Z",
      "updated_at": "2026-11-15T10:30:00.000Z"
    }
  }
  ```
  > 💡 **Developer Hint:** Extract `organizer_id` securely from the decoded JWT token (`req.user.id`). Initialize `available_seats = total_seats`.

#### 4. Get All Events (Search, Filter, Sort & Pagination)
* **Endpoint:** `GET /api/v1/events?search=TypeScript&category=tech&sort=newest&page=1&limit=10`
* **Access:** Public
* **Query Parameters & Rules:**
  * `search`: Case-insensitive partial match (`ILIKE`) on `title` or `description`.
  * `category`: Exact match filter.
  * `sort`: Allowed values: `newest` (default), `oldest`, `price_asc`, `price_desc`.
  * `page`: Integer page number (default `1`).
  * `limit`: Integer records per page (default `10`, max `50`).
* **Success Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Events retrieved successfully",
    "meta": {
      "page": 1,
      "limit": 10,
      "total": 1
    },
    "data": [
      {
        "id": 1,
        "title": "TypeScript Advanced Architecture",
        "description": "Deep dive into enterprise TS patterns and modular design.",
        "category": "tech",
        "date": "2026-12-10T10:00:00.000Z",
        "location": "Dhaka Tech Hub",
        "total_seats": 50,
        "available_seats": 50,
        "price": "1500.00",
        "organizer_id": 1,
        "created_at": "2026-11-15T10:30:00.000Z",
        "updated_at": "2026-11-15T10:30:00.000Z"
      }
    ]
  }
  ```

#### 5. Get Single Event by ID
* **Endpoint:** `GET /api/v1/events/:id`
* **Access:** Public
* **Success Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Event retrieved successfully",
    "data": {
      "id": 1,
      "title": "TypeScript Advanced Architecture",
      "description": "Deep dive into enterprise TS patterns and modular design.",
      "category": "tech",
      "date": "2026-12-10T10:00:00.000Z",
      "location": "Dhaka Tech Hub",
      "total_seats": 50,
      "available_seats": 50,
      "price": "1500.00",
      "organizer": {
        "id": 1,
        "name": "Sarah Connor",
        "email": "sarah@eventhive.com"
      },
      "created_at": "2026-11-15T10:30:00.000Z",
      "updated_at": "2026-11-15T10:30:00.000Z"
    }
  }
  ```
  > 💡 **Developer Hint:** Use a SQL `JOIN` or execute a secondary query to embed the `organizer` object inside the response data.

#### 6. Update Event
* **Endpoint:** `PATCH /api/v1/events/:id`
* **Access:** Protected (`organizer` role who created the event)
* **Request Body:**
  ```json
  {
    "title": "TypeScript Masterclass",
    "total_seats": 60
  }
  ```
  > ⚠️ **Business Rule (Seat Adjustment):** If `total_seats` is updated, adjust `available_seats` proportionally based on current active bookings to prevent negative seat counts or inventory corruption.
* **Success Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Event updated successfully",
    "data": {
      "id": 1,
      "title": "TypeScript Masterclass",
      "description": "Deep dive into enterprise TS patterns and modular design.",
      "category": "tech",
      "date": "2026-12-10T10:00:00.000Z",
      "location": "Dhaka Tech Hub",
      "total_seats": 60,
      "available_seats": 60,
      "price": "1500.00",
      "organizer_id": 1,
      "created_at": "2026-11-15T10:30:00.000Z",
      "updated_at": "2026-11-15T11:00:00.000Z"
    }
  }
  ```

#### 7. Delete Event
* **Endpoint:** `DELETE /api/v1/events/:id`
* **Access:** Protected (`organizer` role who created the event)
* **Business Rule:** Before deleting, check if there are any active confirmed bookings for the event. If active bookings exist, prohibit deletion and return `400 Bad Request` to preserve booking history.
* **Success Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Event deleted successfully",
    "data": null
  }
  ```

---

### 🔹 Bookings & Organizer Module

#### 8. Book Event Tickets
* **Endpoint:** `POST /api/v1/bookings`
* **Access:** Protected (`attendee` or `organizer`)
* **Request Body:**
  ```json
  {
    "event_id": 1,
    "tickets_booked": 2
  }
  ```
* **Concurrency & Transaction Business Rules:**
  1. Acquire a transaction client from the pool (`const client = await pool.connect()`).
  2. Run `BEGIN`.
  3. Query the event with row-level locking (`SELECT * FROM events WHERE id = $1 FOR UPDATE`). If not found, return `404 Not Found`.
  4. Check if `available_seats >= tickets_booked`. If insufficient, return `400 Bad Request` (Sold Out).
  5. Calculate `total_price = event.price * tickets_booked`.
  6. Decrement `available_seats` on the event and insert the booking record.
  7. Run `COMMIT` and release client.
* **Success Response (201 Created):**
  ```json
  {
    "success": true,
    "message": "Tickets booked successfully",
    "data": {
      "id": 1,
      "event_id": 1,
      "attendee_id": 2,
      "tickets_booked": 2,
      "total_price": "3000.00",
      "status": "confirmed",
      "created_at": "2026-11-15T11:15:00.000Z",
      "updated_at": "2026-11-15T11:15:00.000Z"
    }
  }
  ```

#### 9. Get User Bookings
* **Endpoint:** `GET /api/v1/bookings`
* **Access:** Protected (Returns all ticket bookings for the authenticated user from `req.user.id`)
* **Success Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Bookings retrieved successfully",
    "data": [
      {
        "id": 1,
        "event": {
          "id": 1,
          "title": "TypeScript Masterclass",
          "date": "2026-12-10T10:00:00.000Z",
          "location": "Dhaka Tech Hub"
        },
        "tickets_booked": 2,
        "total_price": "3000.00",
        "status": "confirmed",
        "created_at": "2026-11-15T11:15:00.000Z"
      }
    ]
  }
  ```

#### 10. Cancel Booking
* **Endpoint:** `PATCH /api/v1/bookings/:id/cancel`
* **Access:** Protected (Authenticated attendee who owns the booking)
* **Business Rules:**
  1. Verify booking exists and belongs to user (`req.user.id`). If missing, return `404 Not Found`. If unauthorized, return `403 Forbidden`.
  2. Check if status is already `cancelled`. If so, return `409 Conflict`.
  3. Inside a database transaction, update booking status to `cancelled` and atomically restore `tickets_booked` back to the event's `available_seats`.
* **Success Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Booking cancelled successfully",
    "data": {
      "id": 1,
      "event_id": 1,
      "attendee_id": 2,
      "tickets_booked": 2,
      "total_price": "3000.00",
      "status": "cancelled",
      "created_at": "2026-11-15T11:15:00.000Z",
      "updated_at": "2026-11-15T12:00:00.000Z"
    }
  }
  ```

#### 11. Get Organizer Event Bookings
* **Endpoint:** `GET /api/v1/organizer/bookings`
* **Access:** Protected (`organizer` role only)
* **Description:** Retrieve all ticket bookings across all events created by the logged-in organizer.
* **Success Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Organizer bookings retrieved successfully",
    "data": [
      {
        "id": 1,
        "event_id": 1,
        "attendee_id": 2,
        "tickets_booked": 2,
        "total_price": "3000.00",
        "status": "confirmed",
        "created_at": "2026-11-15T11:15:00.000Z"
      }
    ]
  }
  ```

---

## 🚨 Response Patterns & HTTP Status Codes

* **Standard Success Response Structure:**
  ```json
  {
    "success": true,
    "message": "Operation description",
    "meta": { "page": 1, "limit": 10, "total": 50 },
    "data": {}
  }
  ```
* **Standard Error Response Structure:**
  ```json
  {
    "success": false,
    "message": "Validation Error",
    "errors": [{ "field": "tickets_booked", "message": "Insufficient available seats for this event" }]
  }
  ```
* **HTTP Status Codes Reference:**
  * `200 OK`: Successful `GET`, `PATCH`, `DELETE` requests.
  * `201 Created`: Successful `POST` request resulting in resource creation.
  * `400 Bad Request`: Zod validation failure, invalid input, or sold-out event / deletion prohibited due to active bookings.
  * `401 Unauthorized`: Missing, expired, or invalid JWT token.
  * `403 Forbidden`: Valid token but insufficient role permissions or attempting to access resources owned by another user.
  * `404 Not Found`: Requested resource (`event`, `booking`, `user`) does not exist.
  * `409 Conflict`: Duplicate unique email/ISBN or conflicting state (e.g., cancelling an already cancelled booking).
  * `500 Internal Server Error`: Unexpected server or database failure.

---

## 📂 Production-Ready Folder Structure

Your project must strictly follow the **feature-based layered architecture** (`route` → `controller` → `service`):

```text
src/
├── app.ts
├── server.ts
├── config/
├── middleware/
├── utils/
└── modules/
    ├── auth/
    ├── event/
    └── booking/

```

---

## 🎤 Technical Interview Video (Answer Any 2)

**Questions:**
1. **JWT Authentication Flow:** How do JSON Web Tokens work for stateless authentication, and how do you securely verify them in Express middleware?
2. **Role-Based Authorization:** Explain how you implement role-based access control (RBAC) middleware to restrict event creation to `organizer` accounts versus `attendee` accounts.
3. **SQL Injection & Parameterized Queries:** How does SQL injection happen in raw SQL queries, and how do parameterized queries (`$1, $2`) in `pg` prevent it?
4. **Database Connection Pooling:** What is database connection pooling (`pg.Pool`) in PostgreSQL, and why is it preferred over opening and closing a single client connection for every request?
5. **Centralized Error Handling:** How does a 4-parameter centralized error-handling middleware catch both synchronous and asynchronous errors across your layered architecture?

**Recording Instructions:**
* Use your smartphone selfie camera or laptop webcam in **landscape (horizontal) mode**.
* Record in a **well-lit, quiet room** with your **face fully visible** throughout the video.
* Select and answer **any 2 questions** from the list above.
* Keep each answer between **2–4 minutes**. Speak naturally from your understanding — avoid reading verbatim from notes or scripts.
* Upload your video to **Google Drive** (set access to *"Anyone with the link"*) or **YouTube (Unlisted)**, and include the public link in your submission and `README.md`.

---

## 📬 Submission Guidelines

### 1️⃣ Codebase & Documentation Requirements
* **Strict Specification Adherence:** Follow exact endpoint paths, HTTP methods, `snake_case` keys, and response structures.
* **Database Schema File (`schema.sql`):** Include a `schema.sql` file in your project root containing the `CREATE TABLE` statements so evaluators can set up your tables immediately.
* **API Documentation (`README.md`):** Include a well-structured `README.md` with project overview, live deployment URL, setup instructions, environment variable example (`.env.example`), interview video link, and complete API endpoint documentation.

### 2️⃣ Deployment Requirements
* Deploy backend to **Render**, **Railway**, or **Vercel**.
* Host PostgreSQL database on **NeonDB**, **Supabase**, or **Aiven**.

### 3️⃣ Final Submission Checklist
```text
✅ GitHub Repo (Public):      https://github.com/yourusername/eventhive-api
✅ Live Deployment (Public):  https://eventhive-api.onrender.com
✅ Interview Video (Public):  https://drive.google.com/... or https://youtu.be/...
```
> 💡 **Pro Tips:** Ensure your repository has **at least 10 meaningful commits** showing progressive development. Double-check all links in an incognito window before submitting.

---

## ⚠️ Academic Integrity Policy
* **Plagiarism will not be tolerated.** All submissions must be your original work.
* Any instance of copied repositories or AI-generated verbatim scripts during the video interview will result in **0 Marks**.
