# 🎉 EventHive – Assignment 2 Requirements Specification

> **Event & Ticket Booking Platform**
>
> *A scalable, modular backend platform for managing community events, ticket reservations with seat-count inventory, and role-based event organization.*

> ⚠️ **CRITICAL EVALUATION RULE (ZERO-MARK POLICY):**
> You **must** follow this assignment specification strictly. Every API endpoint path, HTTP method, access criteria, request header, request body key (`snake_case`), and response body structure must match this document **100%**. Because your submission will be evaluated using automated testing scripts, **any deviation from the specified endpoints, methods, access criteria, headers, request bodies, or response structures will result in 0 Marks.**

---

## 🛠️ Technology Stack & Architectural Rules

| Technology / Concept | Requirement & Note |
| :--- | :--- |
| **Node.js** | LTS runtime (22.x or 24.x) |
| **TypeScript** | Strict mode enabled (`strict: true`, which includes `noImplicitAny`), latest stable version |
| **Express.js** | Feature-based (modular) & Layered architecture (`route` → `controller` → `service` → `repository`). The repository layer is **mandatory** and is the only layer that runs SQL. |
| **Express Version** | **Express 5.x** |
| **PostgreSQL** | Relational database using native `pg` (`Pool`) driver. **No ORMs allowed.** |
| **Raw SQL** | Parameterized queries (`pool.query($1, $2)`) mandatory for every database operation to completely prevent SQL injection. Never concatenate user input into SQL. |
| **bcrypt** | Password hashing (salt rounds between 8 and 12) |
| **jsonwebtoken** | JWT generation & verification |
| **Zod (Optional)** | Request body, query parameter and path parameter validation via custom validation middleware |
| **Error Handling** | Centralized error-handling middleware utilizing a custom `AppError` class |

---

## 👥 User Roles & Permissions

| Role | Allowed Actions |
| :--- | :--- |
| **attendee** | • Register and log in<br>• Browse and search events (all events are publicly visible; there is no draft/published state)<br>• Book tickets for events (with automatic available seat validation & price calculation)<br>• View own booking history & cancel own bookings |
| **organizer** | • All attendee permissions, **except** booking tickets for their own events (rejected with `403`)<br>• Create, update, and delete their own organized events |

---

## 🗄️ Database Schema Design

### Table 1: `users`
| Field | Constraint / Rules | Description |
| :--- | :--- | :--- |
| `id` | Primary Key | Unique identifier for each account |
| `name` | Required | Full display name of the user |
| `email` | Unique, Required | Unique login email address |
| `password` | Required | Hashed password string (**never returned in responses**) |
| `role` | Default: `attendee`, Required, must be `attendee` or `organizer` | Access control role |
| `created_at` | Auto Timestamp | Account creation timestamp |
| `updated_at` | Auto Timestamp | Last update timestamp |

### Table 2: `events`
| Field | Constraint / Rules | Description |
| :--- | :--- | :--- |
| `id` | Primary Key | Unique identifier for each event |
| `title` | Required | Headline title of the event |
| `description` | Required | Detailed description of the event |
| `category` | Required | Category tag (e.g., `tech`, `music`, `workshop`) |
| `date` | Required | Scheduled date and time |
| `location` | Required | Venue location or online link |
| `total_seats` | Required, must be greater than 0 | Maximum ticket capacity |
| `available_seats` | Required, never below 0 and never above `total_seats` | Current available seats (initialized equal to `total_seats` on create; recalculated on updates, see Update Event) |
| `price` | Required, must be 0 or greater, decimal with 2 places | Price per single ticket |
| `organizer_id` | Foreign Key (References `users.id`), Required | Links the event to the organizing user |
| `created_at` | Auto Timestamp | Event creation timestamp |
| `updated_at` | Auto Timestamp | Event update timestamp |

### Table 3: `bookings`
| Field | Constraint / Rules | Description |
| :--- | :--- | :--- |
| `id` | Primary Key | Unique identifier for each booking |
| `event_id` | Foreign Key (References `events.id`), Required | Links the booking to the event. Cancelled bookings are removed together with their event (cascade delete). Events with confirmed bookings cannot be deleted. |
| `attendee_id` | Foreign Key (References `users.id`), Required | Links the booking to the attending user |
| `tickets_booked` | Required, must be greater than 0 | Number of seats reserved |
| `total_price` | Required, decimal with 2 places | Automatically calculated as `tickets_booked * event.price` |
| `status` | Default: `confirmed`, Required, must be `confirmed` or `cancelled` | Booking state |
| `created_at` | Auto Timestamp | Booking creation timestamp |
| `updated_at` | Auto Timestamp | Booking update timestamp |

> 🕒 **Timestamps:** `created_at` and `updated_at` are set automatically on insert. `updated_at` must also be refreshed whenever a record is updated (event update, booking cancel, seat changes).

> 💡 **Database integrity:** Enforce the rules above at the database level too, using `CHECK`, `UNIQUE`, `NOT NULL` and foreign key constraints, in addition to your application checks.

> 📌 **Note on Relational Queries (`JOIN` vs. Multiple Queries):** Where an endpoint requires related data from another table (such as `organizer` details on a single event or `event` details on user bookings), you may either use a SQL `JOIN` query **or** execute separate queries on multiple tables and combine/structure the data using JavaScript/TypeScript object manipulation. Both approaches are fully allowed.

---

## 🔐 Authentication Rules

- **JWT payload:** `{ "id": <user id>, "role": "<role>" }`.
- **Environment variables:** `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN` (e.g. `7d`), `PORT`.
- **Header:** every protected endpoint expects the token in the `Authorization` header. The auth middleware must accept both `Authorization: Bearer <JWT_TOKEN>` and `Authorization: <JWT_TOKEN>`.
- Missing, malformed, expired or invalid token → `401`.
- Valid token but wrong role → `403`.
- **`req.user`:** after verifying the token, the auth middleware attaches the decoded payload (`id`, `role`) to `req.user`. Since Express's `Request` type has no `user` property, extend the type once (for example with a TypeScript declaration file) so `req.user` compiles under strict mode.
- For endpoints that use a resource id (`:id`), check that the resource exists first (`404`), then check ownership/permission (`403`).

---

## 🌐 API Endpoints Specification (10 Endpoints)

### 🔹 Authentication Module

#### 1. User Registration
- **Endpoint:** `POST /api/v1/auth/signup`
- **Access:** Public
- **Request Body (Zod Validated):** `role` is optional and defaults to `attendee` when omitted. Only `attendee` and `organizer` are accepted.
  ```json
  {
    "name": "Sarah Connor",
    "email": "sarah@eventhive.com",
    "password": "securePassword123",
    "role": "organizer"
  }
  ```
- **Success Response (201 Created):**
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
- **Error Cases:** `400` validation failure · `409` email already registered

#### 2. User Login
- **Endpoint:** `POST /api/v1/auth/login`
- **Access:** Public
- **Request Body:**
  ```json
  {
    "email": "sarah@eventhive.com",
    "password": "securePassword123"
  }
  ```
- **Success Response (200 OK):**
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
- **Error Cases:** `400` validation failure · `401` invalid email or password (use the same message for both cases)

---

### 🔹 Events Module

> 📌 **Canonical event shape:** every endpoint that returns an event uses the same fields: `id, title, description, category, date, location, total_seats, available_seats, price, organizer_id, created_at, updated_at`. `price` and other decimal values are returned as **strings** with 2 decimals (the default `pg` behavior for `NUMERIC`), while request bodies send `price` as a **number**.

#### 3. Create Event
- **Endpoint:** `POST /api/v1/events`
- **Access:** Protected (`organizer` role only)
- **Headers:** `Authorization: Bearer <JWT_TOKEN>` (or `Authorization: <JWT_TOKEN>`)
- **Request Body:**
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
- **Success Response (201 Created):**
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
  > 💡 **Developer Hint:** Extract `organizer_id` securely from the decoded JWT token (`req.user.id`), never from the request body. Initialize `available_seats = total_seats`.
- **Error Cases:** `400` validation failure · `401` missing/invalid token · `403` caller is not an organizer

#### 4. Get All Events (With Optional Search Query)
- **Endpoint:** `GET /api/v1/events` (or `GET /api/v1/events?search=TypeScript`)
- **Access:** Public
- **Query Parameters & Rules:**
  - `search` *(optional)*: Case-insensitive partial match (`ILIKE`) on `title` or `description`. When omitted, return all events ordered by `created_at` descending (newest first).
- **Success Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Events retrieved successfully",
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
  > 💡 An empty result returns `200` with `"data": []`.

#### 5. Get Single Event by ID
- **Endpoint:** `GET /api/v1/events/:id`
- **Access:** Public
- **Success Response (200 OK):**
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
      "organizer_id": 1,
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
  > 💡 **Developer Hint:** If you are comfortable with SQL `JOIN`, you can fetch the event and organizer in a single query. Otherwise, query the `events` table first, then query the `users` table by `organizer_id`, and attach the `organizer` object using JavaScript/TypeScript object manipulation.
- **Error Cases:** `400` invalid id · `404` event not found

#### 6. Update Event
- **Endpoint:** `PATCH /api/v1/events/:id`
- **Access:** Protected (`organizer` role who created the event; any other user gets `403`)
- **Headers:** `Authorization: Bearer <JWT_TOKEN>` (or `Authorization: <JWT_TOKEN>`)
- **Updatable fields:** `title`, `description`, `category`, `date`, `location`, `total_seats`, `price` (at least one required).
- **Request Body:**
  ```json
  {
    "title": "TypeScript Masterclass",
    "total_seats": 60
  }
  ```
  > 💡 **Developer Hint:** Only the fields present in the request body should be changed. Build the `SET` part of your `UPDATE` query dynamically from the provided fields, and keep every value parameterized (`$1, $2, ...`). Never insert user input directly into the SQL text.
  >
  > ⚠️ **Business Rule (Seat Adjustment):** Count the confirmed tickets already sold for the event (the sum of `tickets_booked` for its bookings with status `confirmed`). When `total_seats` is updated, set `available_seats = new_total_seats - sold`. If `total_seats` is not part of the request, `available_seats` stays unchanged. If that value would be negative (the new total is lower than the tickets already sold), reject with `400 Bad Request`.
  >
  > *Example:* an event has 50 seats and 10 are sold. Updating `total_seats` to 60 gives `available_seats = 50`.
  >
  > Changing `price` affects only future bookings. Existing bookings keep their original `total_price`.
- **Success Response (200 OK):** *(the example assumes no tickets have been sold yet)*
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
- **Error Cases:** `400` validation failure / empty body / `total_seats` below tickets sold · `401` · `403` not the owner · `404` event not found

#### 7. Delete Event
- **Endpoint:** `DELETE /api/v1/events/:id`
- **Access:** Protected (`organizer` role who created the event)
- **Headers:** `Authorization: Bearer <JWT_TOKEN>` (or `Authorization: <JWT_TOKEN>`)
- **Business Rule:** Before deleting, check whether the event has any bookings with status `confirmed`. If any exist, prohibit deletion and return `400 Bad Request` to preserve booking history. Otherwise delete the event. Any remaining `cancelled` bookings of that event are removed together with it (cascade delete).
- **Success Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Event deleted successfully",
    "data": null
  }
  ```
- **Error Cases:** `400` event has confirmed bookings · `401` · `403` not the owner · `404` event not found

---

### 🔹 Bookings Module

#### 8. Book Event Tickets
- **Endpoint:** `POST /api/v1/bookings`
- **Access:** Protected (`attendee` or `organizer`)
- **Headers:** `Authorization: Bearer <JWT_TOKEN>` (or `Authorization: <JWT_TOKEN>`)
- **Request Body:**
  ```json
  {
    "event_id": 1,
    "tickets_booked": 2
  }
  ```
- **Business Rules:**
  1. Find the event by `event_id`. If not found, return `404 Not Found`.
  2. If the event's `organizer_id` equals `req.user.id`, return `403 Forbidden` (organizers cannot book their own events).
  3. If the event `date` has already passed, return `400 Bad Request`.
  4. Check if `available_seats >= tickets_booked`. If insufficient, return `400 Bad Request` (Sold Out / Insufficient seats).
  5. Calculate `total_price = event.price * tickets_booked`.
  6. Decrement `available_seats` on the event and insert the booking record with status `confirmed`.
- **Success Response (201 Created):**
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
- **Error Cases:** `400` validation failure / insufficient seats / event already passed · `401` · `403` organizer booking own event · `404` event not found

#### 9. Get User Bookings
- **Endpoint:** `GET /api/v1/bookings`
- **Access:** Protected (any authenticated user; returns only the bookings of `req.user.id`, newest first)
- **Headers:** `Authorization: Bearer <JWT_TOKEN>` (or `Authorization: <JWT_TOKEN>`)
- **Success Response (200 OK):**
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
  > 💡 **Developer Hint:** You can populate the nested `event` object using a SQL `JOIN`, or by querying the user's bookings first, fetching the matching events from the `events` table, and structuring the nested `event` object using JavaScript/TypeScript object manipulation. An empty list returns `200` with `"data": []`.
- **Error Cases:** `401` missing/invalid token

#### 10. Cancel Booking
- **Endpoint:** `PATCH /api/v1/bookings/:id/cancel`
- **Access:** Protected (any authenticated user, attendee or organizer, who owns the booking)
- **Headers:** `Authorization: Bearer <JWT_TOKEN>` (or `Authorization: <JWT_TOKEN>`)
- **Business Rules:**
  1. Find the booking. If it does not exist, return `404 Not Found`. If it exists but `attendee_id` is not `req.user.id`, return `403 Forbidden`.
  2. If the booking status is already `cancelled`, return `409 Conflict`.
  3. If the event `date` has already passed, return `400 Bad Request` (a booking cannot be cancelled after the event has started).
  4. Update the booking status to `cancelled` and add `tickets_booked` back to the event's `available_seats`.
- **Success Response (200 OK):**
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
- **Error Cases:** `400` invalid id / event already passed · `401` · `403` booking belongs to another user · `404` booking not found · `409` already cancelled

---

## 🚨 Response Patterns & HTTP Status Codes

- **Standard Success Response Structure:**
  ```json
  {
    "success": true,
    "message": "Operation description",
    "data": {}
  }
  ```
- **Standard Error Response Structure:**
  ```json
  {
    "success": false,
    "message": "Validation Error",
    "errors": [{ "field": "tickets_booked", "message": "Must be at least 1" }]
  }
  ```
  > Business-rule failures (for example sold out, or deletion blocked) return a clear `message` such as `"Insufficient available seats for this event"`. The `errors` array is optional for them. Never expose stack traces or raw SQL errors.
- **HTTP Status Codes Reference:**
  - `200 OK`: Successful `GET`, `PATCH`, `DELETE` requests.
  - `201 Created`: Successful `POST` request resulting in resource creation.
  - `400 Bad Request`: Zod validation failure, invalid input, sold-out/insufficient seats, booking or cancelling for an event that already passed, `total_seats` lower than tickets sold, or deletion prohibited due to active bookings.
  - `401 Unauthorized`: Missing, expired, or invalid JWT token.
  - `403 Forbidden`: Valid token but insufficient role permissions, an organizer booking their own event, or attempting to access/modify resources owned by another user.
  - `404 Not Found`: Requested resource (`event`, `booking`, `user`) does not exist.
  - `409 Conflict`: Duplicate unique email, or conflicting state (e.g., cancelling an already cancelled booking).
  - `500 Internal Server Error`: Unexpected server or database failure.

---

## 📂 Production-Ready Folder Structure

Your project must strictly follow the **feature-based layered architecture** (`route` → `controller` → `service` → `repository`):

```text
├── README.md
├── schema.sql
└── src/
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

Each module contains its own `route`, `controller`, `service`, `repository` and Zod validation schema files.

---

## 🎤 Technical Interview Video (Answer Any 2)

**Questions:**
1. **JWT Authentication Flow:** How do JSON Web Tokens work for stateless authentication, and how do you securely verify them in Express middleware?
2. **Role-Based Authorization:** Explain how you implement role-based access control (RBAC) middleware to restrict event creation to `organizer` accounts versus `attendee` accounts.
3. **SQL Injection & Parameterized Queries:** How does SQL injection happen in raw SQL queries, and how do parameterized queries (`$1, $2`) in `pg` prevent it?
4. **Database Connection Pooling:** What is database connection pooling (`pg.Pool`) in PostgreSQL, and why is it preferred over opening and closing a single client connection for every request?
5. **Centralized Error Handling:** How does a 4-parameter centralized error-handling middleware catch both synchronous and asynchronous errors across your layered architecture?

**Recording Instructions:**
- Use your smartphone selfie camera or laptop webcam in **landscape (horizontal) mode**.
- Record in a **well-lit, quiet room** with your **face fully visible** throughout the video.
- Select and answer **any 2 questions** from the list above.
- Keep each answer between **2–4 minutes**. Speak naturally from your understanding. Avoid reading verbatim from notes or scripts.
- Upload your video to **Google Drive** (set access to *"Anyone with the link"*) or **YouTube (Unlisted)**, and include the public link in your submission and `README.md`.

---

## 📬 Submission Guidelines

### 1️⃣ Codebase & Documentation Requirements
- **Strict Specification Adherence (Zero-Mark Warning):** You must follow the exact API endpoint paths, HTTP methods, access criteria, headers, request body structure, and response body structure specified in this document. Any deviation will cause automated evaluation tests to fail and result in **0 Marks**.
- **Database Schema File (`schema.sql`):** Write your own `schema.sql` in the project root containing the `CREATE TABLE` statements (with all constraints described in the schema section) so evaluators can set up your tables immediately.
- **API Documentation (`README.md`):** Include a well-structured `README.md` with project overview, live deployment URL, setup instructions, environment variable example (`.env.example`), interview video link, and complete API endpoint documentation.

### 2️⃣ Deployment Requirements
- Deploy backend to **Render**, **Railway**, or **Vercel**.
- Host PostgreSQL database on **NeonDB**, **Supabase**, or **Aiven**.

### 3️⃣ Final Submission Checklist
```text
✅ GitHub Repo (Public):      https://github.com/yourusername/eventhive-api
✅ Live Deployment (Public):  https://eventhive-api.onrender.com
✅ Interview Video (Public):  https://drive.google.com/... or https://youtu.be/...
```
> 💡 **Pro Tips:** Ensure your repository has **at least 10 meaningful commits** showing progressive development. Double-check all links in an incognito window before submitting.

### 4️⃣ Deadline & Marking
- **Due date:** *to be announced by the instructor*
- **Marks distribution:** *to be announced by the instructor*

---

## ⚠️ Academic Integrity Policy
- **Plagiarism will not be tolerated.** All submissions must be your original work.
- Copied repositories, or interview answers read out verbatim from AI-generated or other pre-written scripts, will result in **0 Marks**. You may be asked to explain any part of your code.
