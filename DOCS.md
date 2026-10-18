# ParkEase – Design Documentation

(Open in VS Code Markdown Preview or GitHub to see the diagrams.)

## 1. Architecture

**Style:** 3-tier client–server architecture, REST API, stateless authentication (JWT). The backend follows MVC: Routes -> Middleware -> Controllers -> Models.

```mermaid
flowchart LR
  subgraph Client["Presentation tier - React + Tailwind (browser)"]
    P[Pages: Login, Areas, AreaSlots, MyBookings, Admin]
    AC[AuthContext + Axios instance<br/>attaches JWT to every request]
  end
  subgraph Server["Application tier - Node.js + Express"]
    R[Routes] --> M[Middleware<br/>protect / adminOnly]
    M --> C[Controllers<br/>business logic]
    C --> U[Utils: slot locking, QR generator,<br/>email + SMS notifier]
  end
  subgraph Data["Data tier - MongoDB via Mongoose"]
    DB[(Users, Areas, Slots, SlotLocks,<br/>Bookings, Payments)]
  end
  P --> AC -->|HTTP JSON + Bearer token| R
  C -->|Mongoose models| DB
  C -->|create order / verify signature| RZ[Razorpay]
  U -->|SMTP| Mail[Email service]
  U -->|REST| SMS[Twilio SMS]
```

| Layer | Responsibility |
|---|---|
| React pages | UI and form handling only, no business rules |
| Axios + AuthContext | Stores the JWT, sends it in `Authorization: Bearer`, keeps login state |
| Routes | Map URL + HTTP method to a controller and decide which middleware protects it |
| Middleware | `protect` verifies the JWT; `adminOnly` checks role |
| Controllers | All business rules (locking, price, payment verification, check-in) |
| Models | Schema, validation, DB access |

**Why this design:** separation of concerns (UI / logic / data), so each part can change independently. All rules are enforced on the server, because the client can never be trusted.

## 2. Database design (ER diagram)

```mermaid
erDiagram
  USER ||--o{ BOOKING : makes
  AREA ||--o{ SLOT : contains
  AREA ||--o{ BOOKING : "booked in"
  SLOT ||--o{ BOOKING : reserved
  BOOKING ||--o| PAYMENT : "paid by"
  BOOKING ||--o{ SLOTLOCK : holds
  SLOT ||--o{ SLOTLOCK : "locked hours"
  USER ||--o{ PAYMENT : pays
  USER { string name string email string password string phone string role }
  AREA { string name string address string city number pricePerHour }
  SLOT { ObjectId area string slotNumber string type bool isActive }
  BOOKING { ObjectId user ObjectId area ObjectId slot string vehicleNumber date startTime date endTime number amount string status string qrCode date checkedInAt string razorpayOrderId }
  SLOTLOCK { ObjectId slot date hour ObjectId booking }
  PAYMENT { ObjectId booking ObjectId user number amount string method string transactionId string status }
```

- Relations are ObjectId references (like foreign keys), resolved with `populate()`.
- A slot has no `isBooked` flag. Availability depends on the time range: one `SlotLock` document exists per slot per booked hour, so a slot can be booked many times at different hours.
- `SlotLock` has a **unique index (slot + hour)**. This makes double booking impossible at database level (section 5).
- `Slot` has a unique index (area + slotNumber); `User.email` is unique; passwords are stored only as bcrypt hashes.

## 3. Data flow diagrams

### Level 0 (context)
```mermaid
flowchart LR
  U[User] -->|register, login, search, book, pay, cancel| S((ParkEase System))
  S -->|slots, booking history, QR code| U
  A[Admin] -->|manage areas, slots, users, verify QR| S
  S -->|statistics, bookings, payments, CSV reports| A
  S -->|create order, verify payment| G[Razorpay]
  S -->|confirmation email + SMS| N[Email / SMS services]
```

### Level 1 (main processes and data stores)
```mermaid
flowchart TB
  U[User] --> P1[1. Authentication]
  P1 <--> D1[(Users)]
  U --> P2[2. Search areas and check slots]
  P2 <--> D2[(Areas)]
  P2 <--> D3[(Slots)]
  P2 --> D6[(SlotLocks)]
  U --> P3[3. Create booking + lock slot]
  P3 <--> D4[(Bookings)]
  P3 <--> D6
  U --> P4[4. Payment and QR]
  P4 <--> G[Razorpay]
  P4 --> D5[(Payments)]
  P4 --> D4
  P4 --> N[Email + SMS]
  U --> P5[5. Cancel booking]
  P5 --> D4
  P5 --> D5
  P5 --> D6
  A[Admin] --> P6[6. Manage areas, slots, users]
  P6 <--> D1
  P6 <--> D2
  P6 <--> D3
  A --> P7[7. Reports, dashboard, QR check-in]
  D4 --> P7
  D5 --> P7
```

## 4. Workflow

### 4.1 Main booking workflow
```mermaid
sequenceDiagram
  actor U as User
  participant FE as React
  participant API as Express API
  participant DB as MongoDB
  U->>FE: Login
  FE->>API: POST /auth/login
  API->>DB: find user, bcrypt compare
  API-->>FE: JWT + user
  U->>FE: Open area, choose start/end time
  FE->>API: GET /slots/area/:id?start&end
  API->>DB: slots + locked hours in range
  API-->>FE: slots with available true/false
  U->>FE: Select slot, vehicle no., click Book
  FE->>API: POST /bookings (JWT)
  API->>DB: save booking = pending_payment
  API->>DB: insert SlotLock per hour (unique index)
  API-->>FE: booking (10 min hold) or 409 if slot taken
  U->>FE: Click Pay
  FE->>API: POST /payments/:id/order, then /verify (see 4.4)
  API->>DB: save Payment, booking = confirmed
  API->>API: generate QR from booking details
  API-->>FE: booking + QR
  API-->>U: Confirmation email + SMS
```

### 4.2 Booking status lifecycle
```mermaid
stateDiagram-v2
  [*] --> pending_payment: user books slot (hours locked)
  pending_payment --> confirmed: payment verified
  pending_payment --> cancelled: 10 min expired / user cancels (locks released)
  confirmed --> cancelled: user cancels before start time (payment marked refunded, locks released)
  cancelled --> [*]
```

### 4.3 Authentication and authorization flow
1. Register: input validated, password hashed with bcrypt, role forced to `user` (admin only via seed/DB).
2. Login: server verifies the hash and returns a JWT (`{id}`, 7 days) signed with `JWT_SECRET`.
3. React saves the token in localStorage; the Axios interceptor adds it to every request.
4. `protect` middleware verifies the token and loads the user; `adminOnly` checks `role === 'admin'`.
5. React `Guard` hides pages for UX only. The real protection is the server middleware.

### 4.4 Payment workflow (Razorpay, test mode)
```mermaid
sequenceDiagram
  actor U as User
  participant FE as React
  participant API as Express API
  participant RZ as Razorpay
  participant DB as MongoDB
  U->>FE: Click Pay
  FE->>API: POST /payments/:id/order
  API->>DB: load booking, amount from DB (never from browser)
  API->>RZ: create order (amount in paise)
  RZ-->>API: order_id
  API-->>FE: order_id + public key id
  FE->>RZ: Checkout popup, user pays
  RZ-->>FE: payment_id + signature
  FE->>API: POST /payments/:id/verify
  API->>API: HMAC-SHA256(order_id|payment_id, secret) == signature ?
  API->>DB: Payment saved, booking confirmed, QR generated
  API-->>FE: confirmed booking
```
The browser only receives the public key id; the secret stays on the server. The booking is confirmed **only** after the signature check. If no Razorpay keys are set, the app runs in demo mode (`/simulate`), which is automatically disabled once keys exist.

### 4.5 QR check-in workflow
```mermaid
flowchart LR
  A[Admin pastes QR content] --> B[POST /admin/verify]
  B --> C{Booking found?}
  C -- no --> X[INVALID]
  C -- yes --> D{Status confirmed?}
  D -- no --> X
  D -- yes --> E{Within time window?<br/>30 min before start to end}
  E -- no --> X
  E -- yes --> F[VALID - set checkedInAt]
```

## 5. Key logic to explain

**Double-booking prevention (slot locks).**
1. The requested time is rounded outward to whole hours (10:30-11:15 becomes 10:00-12:00). Booking granularity is 1 hour.
2. The server creates the booking as `pending_payment`, then inserts one `SlotLock` document per hour: `{slot, hour, booking}`.
3. `SlotLock` has a **unique index on (slot, hour)**. If two users try the same hour at the same moment, MongoDB accepts one insert and rejects the other with duplicate-key error 11000. The loser gets HTTP 409 "Slot already booked" and their booking is deleted; any partial locks are rolled back.
4. Cancelling or expiring a booking deletes its locks, so the slot becomes free.
5. The slot grid uses the same locks: a slot is "Booked" if any lock exists for the requested hours (`utils/availability.js`).

Why not "check then insert"? Two requests can both pass the check and then both insert. A unique index turns the decision into one atomic database operation. MongoDB transactions would also work, but need a replica set, which a plain local MongoDB does not have.

**Price:** `hours x area.pricePerHour`, calculated on the server.

**Unpaid hold and auto-expiry:** an unpaid booking holds the slot for 10 minutes. A job runs every minute (and before each availability check and "my bookings" load) that cancels unpaid bookings older than that and deletes their locks.

**Notifications:** `utils/notify.js` sends email (Nodemailer) and SMS (Twilio REST API) together. Failures are swallowed, so a failed notification never breaks a booking. Without credentials both are printed in the console.

**QR code:** generated after payment with the `qrcode` library. It encodes booking id, area, slot, vehicle and times as JSON and is stored as a base64 image in the booking.

**Reports:** `GET /admin/report` builds a CSV (bookings or payments, optional date range) that the browser downloads.

**Dashboard:** `countDocuments` and `aggregate` ($group, $lookup) for totals, revenue and confirmed bookings per area.

## 6. REST API

| Method | Endpoint | Access | Purpose |
|---|---|---|---|
| POST | /api/auth/register | public | Register user |
| POST | /api/auth/login | public | Login, returns JWT |
| GET | /api/auth/me | user | Current user |
| GET | /api/areas?q= | public | Search areas |
| GET | /api/areas/:id | public | Area details |
| POST/PUT/DELETE | /api/areas(/:id) | admin | Add / edit / delete area |
| GET | /api/slots/area/:areaId?start&end | public | Slots with availability |
| POST/PUT/DELETE | /api/slots(/:id) | admin | Add / edit / delete slot |
| POST | /api/bookings | user | Create booking (pending) and lock hours |
| GET | /api/bookings/my | user | Booking history |
| PUT | /api/bookings/:id/cancel | user | Cancel booking |
| POST | /api/payments/:bookingId/order | user | Create Razorpay order (or report demo mode) |
| POST | /api/payments/:bookingId/verify | user | Verify signature, confirm booking, QR, email + SMS |
| POST | /api/payments/:bookingId/simulate | user | Demo payment (only when no Razorpay keys) |
| GET | /api/admin/stats | admin | Dashboard statistics |
| GET | /api/admin/bookings, /payments, /users | admin | Lists |
| DELETE | /api/admin/users/:id | admin | Delete user |
| POST | /api/admin/verify | admin | Verify QR / booking id and check the vehicle in |
| GET | /api/admin/report?type&from&to | admin | Download bookings or payments CSV |

## 7. Security measures
- Passwords hashed (bcrypt, 10 rounds), never returned by the API.
- JWT verification on protected routes; role-based access for admin routes; role cannot be set at registration.
- Payment amount is read from the database, and payment is confirmed only after the Razorpay signature is verified on the server. The order id must match the one stored on the booking.
- Users can only pay for or cancel their own bookings.
- Server-side validation (email, password length, phone, vehicle number, time range, slot active).
- Search text is regex-escaped to avoid regex injection.

## 8. Honest limitations (say these before the examiner asks)
1. **Razorpay runs in test mode only.** Going live needs a verified account, live keys and HTTPS. There is no webhook: if a user pays but closes the browser before `/verify` runs, the booking is not confirmed and expires. A production system adds a Razorpay webhook to handle this.
2. **Refunds are only recorded** (`status: refunded`), not sent back through the gateway. A real refund would call Razorpay's refund API.
3. **SMS needs Twilio credentials**; otherwise it is printed in the console. Twilio trial accounts can only text verified numbers.
4. **"Real-time" availability uses polling** (frontend refreshes every 10 s). True real-time would use Socket.io.
5. Booking granularity is 1 hour. QR check-in is by pasting the QR text; a camera scanner would be an extension.
6. No rate limiting, no automated tests, JWT stored in localStorage (an httpOnly cookie is safer).

## 9. Likely viva questions

**Why MERN?** One language end to end, JSON everywhere, flexible documents, large ecosystem.

**Why MongoDB and not SQL?** The data is simple and document-shaped. SQL would also fit (the data is relational); the trade-off is that referential integrity is handled through Mongoose refs instead of foreign keys.

**Why JWT instead of sessions?** Stateless: the server stores no session, so it scales easily and suits a separate React frontend. Downside: a token cannot be revoked before expiry without extra work.

**How do you prevent two users booking the same slot at the same time?** A unique index on (slot, hour) in the SlotLock collection. The database rejects the second insert, so it works even when both requests arrive together.

**How do you make sure a payment is genuine?** Razorpay signs orderId|paymentId with a secret only the server knows. The server recomputes the HMAC and confirms only if it matches. The amount comes from our database, not the browser.

**What if the user never pays?** After 10 minutes the booking is cancelled and its locks deleted by a job that runs every minute; the pay endpoint also rejects an expired booking.

**What is the QR code for?** It holds the booking details. At the gate the admin verifies it: the server checks status and time window and marks the vehicle as checked in; a second scan is flagged as already used.

**How is admin secured?** Admin is created by seed only. Every admin route passes `protect` then `adminOnly`.

**Where is business logic and why?** In server controllers, so it cannot be bypassed by editing the frontend.

**How would you scale it?** Index frequently queried fields, add caching, Socket.io for live updates, run several stateless API instances behind a load balancer.

## 10. Demo script (5 minutes)
1. Login as admin -> Dashboard -> add an area and a few slots.
2. Register a new user -> search the area -> pick a time range -> slots turn green/red.
3. Book a slot -> status "pending payment" -> Pay (Razorpay test popup, or demo mode) -> QR appears; email and SMS lines appear in the server console.
4. In a second browser as another user, same time range: that slot now shows "Booked".
5. Cancel the booking -> slot becomes free again; admin sees the cancelled booking and refunded payment.
6. Admin dashboard: revenue and bookings per area.
7. Admin -> Verify QR: paste the QR text of a confirmed booking -> VALID. Paste again -> already checked in.
8. Admin -> Reports: download the bookings CSV.
