# Blood Bank Management System (BBMS) — Complete Project Documentation

---

## Table of Contents

1. [What This Application Does](#1-what-this-application-does)
2. [Tech Stack](#2-tech-stack)
3. [Project Structure](#3-project-structure)
4. [User Roles & Permissions](#4-user-roles--permissions)
5. [Database Models](#5-database-models)
6. [Backend Architecture](#6-backend-architecture)
7. [Frontend Architecture](#7-frontend-architecture)
8. [Complete User Flow (Step by Step)](#8-complete-user-flow-step-by-step)
9. [API Reference](#9-api-reference)
10. [How to Run the Project](#10-how-to-run-the-project)
11. [How to Test Everything from the Frontend](#11-how-to-test-everything-from-the-frontend)
12. [Key Design Decisions](#12-key-design-decisions)

---

## 1. What This Application Does

BBMS is a full-stack web application that digitizes and coordinates the entire blood donation lifecycle. It connects four types of users:

- **Donors** — register, view blood camps, track their donation history
- **Blood Labs** — manage blood stock, organize donation camps, record donations, accept/reject hospital requests
- **Hospitals** — request blood from labs, track their blood inventory, browse the donor directory
- **Admin** — approve/reject facility registrations, view system-wide statistics

The core workflow is:
```
Donor donates blood at a camp → Blood Lab records it → Blood Lab stores it as stock
→ Hospital requests blood from Lab → Lab accepts → Blood transfers to Hospital inventory
```

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite, Tailwind CSS, React Router v6, Axios, React Hot Toast |
| Backend | Node.js, Express.js |
| Database | MongoDB with Mongoose ODM |
| Auth | JWT (JSON Web Tokens), bcryptjs for password hashing |
| API Docs | Swagger (OpenAPI) at `/api/doc` |
| Containerization | Docker + Docker Compose |

---

## 3. Project Structure

```
project-root/
├── backend/
│   ├── config/db.js              # MongoDB connection
│   ├── controllers/              # Business logic
│   │   ├── authContoller.js      # Register, Login, Profile
│   │   ├── donorController.js    # Donor profile, history, camps, search
│   │   ├── bloodLabController.js # Camps, stock, requests, donor management
│   │   ├── hospitalController.js # Blood requests, inventory, donor directory
│   │   └── adminController.js    # Facility approval, dashboard stats
│   ├── middlewares/
│   │   ├── authMiddleware.js     # protect() — for admin/general routes
│   │   ├── donorMiddleware.js    # protectDonor() — attaches req.donor
│   │   └── facilityMiddleware.js # protectFacility() — attaches req.user (facility)
│   ├── models/
│   │   ├── donorModel.js         # Donor schema
│   │   ├── facilityModel.js      # Hospital + Blood Lab schema (shared)
│   │   ├── adminModel.js         # Admin schema
│   │   ├── bloodModel.js         # Blood stock schema
│   │   ├── bloodCampModel.js     # Blood camp schema
│   │   └── bloodRequestModel.js  # Hospital→Lab blood request schema
│   ├── routes/                   # Express route definitions
│   ├── openapi/                  # Swagger docs
│   ├── seedAdmin.js              # Script to create the first admin account
│   └── server.js                 # App entry point
│
└── frontend/
    └── src/
        ├── App.jsx               # Route definitions
        ├── components/
        │   ├── Header.jsx        # Public landing header
        │   ├── Footer.jsx
        │   ├── ProtectedRoute.jsx # JWT validation guard
        │   └── layouts/
        │       └── DashboardLayout.jsx # Sidebar + header for all dashboards
        ├── pages/
        │   ├── auth/             # Login, DonorRegister, FacultyRegister
        │   ├── donor/            # Dashboard, Profile, Camps, History
        │   ├── bloodlab/         # Dashboard, BloodStock, BloodCamps, Requests, Donors
        │   ├── hospital/         # Dashboard, RequestBlood, RequestHistory, Stock, Donors
        │   └── admin/            # Dashboard, Facilities, Donors
        └── utils/
            ├── api.js            # Axios instance with auto-auth header
            └── auth.js           # Token validation helpers
```

---

## 4. User Roles & Permissions

### Admin
- Created via `node seedAdmin.js` (not through the UI)
- Default credentials: `admin@bbms.com` / `bbms@admin`
- Can: view all donors, view all facilities, approve/reject facility registrations, see system stats

### Blood Lab
- Registers via `/register/facility` with type "blood-lab"
- Must be approved by admin before logging in
- Can: create/manage blood camps, add/remove blood stock, record donor donations, accept/reject hospital blood requests

### Hospital
- Registers via `/register/facility` with type "hospital"
- Must be approved by admin before logging in
- Can: request blood from labs, view their own blood inventory, browse donor directory

### Donor
- Registers via `/register/donor`
- No admin approval needed — can log in immediately
- Can: view their profile, see donation history, browse blood camps

---

## 5. Database Models

### Donor (`donorModel.js`)
Stores individual blood donors.

Key fields:
- `fullName`, `email`, `password` (hashed), `phone`
- `bloodGroup` — enum: A+, A-, B+, B-, O+, O-, AB+, AB-
- `age` (18–65), `gender`, `weight` (min 45kg)
- `address` — street, city, state, pincode
- `lastDonationDate` — updated every time a donation is recorded
- `eligibleToDonate` — manual override flag
- `donationHistory[]` — embedded array of donation records, each with: `donationDate`, `facility` (ref to Facility), `bloodGroup`, `quantity`, `verified`
- **Virtual field** `isEligible` — computed: returns `true` if 90+ days have passed since last donation

### Facility (`facilityModel.js`)
Shared model for both hospitals and blood labs.

Key fields:
- `name`, `email`, `password` (hashed), `phone`, `emergencyContact`
- `facilityType` — "hospital" or "blood-lab"
- `role` — auto-set from `facilityType` via pre-save hook
- `registrationNumber` — unique, required
- `status` — "pending" | "approved" | "rejected" (default: "pending")
- `documents.registrationProof` — URL to uploaded document
- `operatingHours` — open/close times, working days
- `history[]` — activity log: Login, Stock Update, Blood Camp, Donation, etc.

### Admin (`adminModel.js`)
Simple model: `name`, `email`, `password`, `role` (admin/superadmin).

### Blood (`bloodModel.js`)
Tracks blood units in stock.

Key fields:
- `bloodGroup` — enum
- `quantity` — number of units
- `expiryDate` — auto-set to 42 days from creation
- `bloodLab` OR `hospital` — exactly one must be set (validated in pre-save hook)

### BloodCamp (`bloodCampModel.js`)
Represents a blood donation event.

Key fields:
- `hospital` — ref to Facility (the blood lab that organized it)
- `title`, `description`, `date`
- `time` — `{ start, end }` (string times like "09:00")
- `location` — `{ venue, city, state, pincode }`
- `expectedDonors`, `actualDonors`
- `status` — "Upcoming" | "Ongoing" | "Completed" | "Cancelled"

### BloodRequest (`bloodRequestModel.js`)
Tracks blood transfer requests from hospitals to labs.

Key fields:
- `hospitalId` — ref to Facility
- `labId` — ref to Facility
- `bloodType`, `units`
- `status` — "pending" | "accepted" | "rejected"
- `processedAt` — timestamp when lab acted on it

---

## 6. Backend Architecture

### Authentication Flow

Three separate middleware functions handle auth depending on the route:

**`protect`** (authMiddleware.js) — used for admin routes and `/api/auth/profile`
- Verifies JWT, looks up user across Donor, Admin, and Facility models
- Attaches `req.user = { id, role }`

**`protectDonor`** (donorMiddleware.js) — used for all `/api/donor/*` routes
- Verifies JWT, looks up only in Donor model
- Attaches `req.donor` (full Mongoose document)

**`protectFacility`** (facilityMiddleware.js) — used for `/api/blood-lab/*` and `/api/hospital/*`
- Verifies JWT, looks up only in Facility model
- Attaches `req.user` (full Facility Mongoose document)

### Login Logic (`authContoller.js`)

1. Search for user across Donor → Admin → Facility (in that order)
2. Compare password with bcrypt
3. If Facility: check status — block login if "pending" or "rejected"
4. Sign JWT with `{ id, role }`, 7-day expiry
5. Return token + redirect path based on role

### Blood Transfer Logic (when lab accepts a request)

Located in `bloodLabController.js → updateBloodRequestStatus`:

1. Verify request belongs to this lab and is still "pending"
2. Check lab has enough stock of the requested blood type
3. Deduct units from lab's Blood document
4. Add units to hospital's Blood document (create if doesn't exist)
5. Log history entries for both facilities
6. Update request status to "accepted"

### Donation Recording Logic (`donorController.js → markDonation`)

1. Find donor by ID
2. Check 3-month cooldown (last donation must be > 3 months ago)
3. Update `donor.lastDonationDate`
4. Push new entry to `donor.donationHistory[]`
5. Log event in facility history
6. Call `addToBloodStock()` helper to add units to lab's inventory

---

## 7. Frontend Architecture

### Routing (`App.jsx`)

Routes are organized into four protected groups:

```
/donor/*     → ProtectedRoute → DashboardLayout (userRole="donor")
/hospital/*  → ProtectedRoute → DashboardLayout (userRole="hospital")
/lab/*       → ProtectedRoute → DashboardLayout (userRole="blood-lab")
/admin/*     → ProtectedRoute → DashboardLayout (userRole="admin")
```

Public routes: `/`, `/login`, `/register/donor`, `/register/facility`, `/about`, `/contact`

### ProtectedRoute (`ProtectedRoute.jsx`)

Checks if the JWT in localStorage is valid (not expired) using `isTokenValid()`. If invalid, clears the token and redirects to `/login`.

### DashboardLayout (`DashboardLayout.jsx`)

Shared layout for all four dashboards. On mount:
1. Reads token from localStorage
2. Calls `GET /api/auth/profile` to verify the user
3. Checks that the returned role matches the expected `userRole` prop — if mismatch, logs out
4. Renders sidebar navigation based on role
5. Passes `userData` and `theme` to child pages via React Router's `<Outlet context>`

The sidebar menu items are defined in `menuConfig` keyed by role.

### API Utility (`utils/api.js`)

An Axios instance pre-configured with:
- `baseURL` from `VITE_API_URL` env variable (defaults to `http://localhost:5000/api`)
- Request interceptor that automatically attaches `Authorization: Bearer <token>` from localStorage

All pages that use `import api from "../../utils/api"` get auth headers for free.

---

## 8. Complete User Flow (Step by Step)

### Step 1 — Seed the Admin (one-time setup)
```bash
cd backend
node seedAdmin.js
```
This creates: `admin@bbms.com` / `bbms@admin`

---

### Step 2 — Register a Blood Lab
1. Go to `/register/facility`
2. Fill in all fields, select **Facility Type: Blood Lab**
3. Submit → you'll see "Registered, awaiting admin approval"
4. The facility is saved with `status: "pending"` — cannot log in yet

---

### Step 3 — Admin Approves the Blood Lab
1. Log in as admin (`admin@bbms.com` / `bbms@admin`) → redirected to `/admin`
2. Go to **Verification** in the sidebar
3. Find the Blood Lab with status "pending" → click **Approve**
4. The facility's `status` changes to "approved" in the database

---

### Step 4 — Blood Lab Creates a Camp
1. Log in as the Blood Lab → redirected to `/lab`
2. Go to **Camps** in the sidebar
3. Click **Add Camp** → fill in: title, date (must be future), start/end time, venue, city, state, pincode, expected donors
4. Submit → calls `POST /api/blood-lab/camps`
5. Camp is saved with `status: "Upcoming"` by default

---

### Step 5 — Donor Registers and Views Camps
1. Go to `/register/donor` → fill in all fields including blood group, age, weight, address
2. Submit → immediately redirected to `/donor` dashboard (no approval needed)
3. Go to **Blood Camps** in the sidebar
4. Calls `GET /api/donor/camps?status=Upcoming` → the newly created camp appears

---

### Step 6 — Blood Lab Adds Blood Stock
1. Log in as Blood Lab → go to **Inventory** in the sidebar
2. Select blood type (e.g., "O+"), enter quantity (e.g., 50), select action "Add Stock"
3. Click **Add Units** → calls `POST /api/blood-lab/blood/add`
4. A Blood document is created/updated with `bloodLab: <labId>`

---

### Step 7 — Blood Lab Records a Donor Donation
1. Log in as Blood Lab → go to **Donors** in the sidebar
2. Search for a donor by name, email, or phone
3. Click **Donate** next to a donor → a modal appears
4. Confirm blood group, quantity, optional remarks → click **Confirm Donation**
5. Calls `POST /api/blood-lab/donors/donate/:donorId`
6. Donor's `donationHistory` is updated, `lastDonationDate` is set, blood stock increases

---

### Step 8 — Register and Approve a Hospital
1. Go to `/register/facility`, select **Facility Type: Hospital**
2. Admin approves it (same as Step 3)

---

### Step 9 — Hospital Requests Blood from Lab
1. Log in as Hospital → go to **Blood Requests** in the sidebar
2. Select the Blood Lab from the dropdown, choose blood type, enter units
3. Click **Send Blood Request** → calls `POST /api/hospital/blood/request`
4. A BloodRequest document is created with `status: "pending"`

---

### Step 10 — Blood Lab Accepts the Request
1. Log in as Blood Lab → go to **Requests** in the sidebar
2. See the pending request from the hospital → click **Accept**
3. Calls `PUT /api/blood-lab/blood/requests/:id` with `{ action: "accept" }`
4. Lab's stock is reduced, hospital's stock is increased, request status → "accepted"

---

### Step 11 — Hospital Views Their Inventory
1. Log in as Hospital → go to **Inventory** in the sidebar
2. Calls `GET /api/hospital/blood/stock`
3. The accepted blood units now appear in the hospital's inventory

---

## 9. API Reference

### Auth Routes (`/api/auth`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/register` | None | Register donor or facility |
| POST | `/login` | None | Login any user type |
| GET | `/profile` | protect | Get current user profile |

### Donor Routes (`/api/donor`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/profile` | protectDonor | Get donor profile + donation history |
| PUT | `/profile` | protectDonor | Update donor profile |
| GET | `/camps` | protectDonor | List blood camps (filterable by status) |
| GET | `/history` | protectDonor | Paginated donation history |
| GET | `/stats` | protectDonor | Dashboard stats (total donations, eligibility) |

### Blood Lab Routes (`/api/blood-lab`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/dashboard` | protectFacility | Dashboard stats + recent camps |
| GET | `/history` | protectFacility | Activity + login history |
| POST | `/camps` | protectFacility | Create a blood camp |
| GET | `/camps` | protectFacility | List lab's camps |
| PUT | `/camps/:id` | protectFacility | Update a camp |
| PATCH | `/camps/:id/status` | protectFacility | Update camp status |
| DELETE | `/camps/:id` | protectFacility | Delete a camp |
| POST | `/blood/add` | protectFacility | Add blood units to stock |
| POST | `/blood/remove` | protectFacility | Remove blood units from stock |
| GET | `/blood/stock` | protectFacility | Get current blood stock |
| GET | `/blood/requests` | protectFacility | Get all hospital requests |
| PUT | `/blood/requests/:id` | protectFacility | Accept or reject a request |
| GET | `/donors/search` | protectFacility | Search donors by name/email/phone |
| POST | `/donors/donate/:id` | protectFacility | Record a donor donation |
| GET | `/donations/recent` | protectFacility | Recent donation stats |

### Hospital Routes (`/api/hospital`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/dashboard` | protectFacility | Dashboard stats + inventory + requests |
| POST | `/blood/request` | protectFacility | Request blood from a lab |
| GET | `/blood/requests` | protectFacility | Get hospital's blood requests |
| GET | `/blood/stock` | protectFacility | Get hospital's blood inventory |
| GET | `/history` | protectFacility | Activity history |
| GET | `/donors` | protectFacility | Browse donor directory |
| POST | `/donors/:id/contact` | protectFacility | Log a contact attempt |

### Admin Routes (`/api/admin`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/dashboard` | protect | System-wide stats |
| GET | `/facilities` | protect | List all facilities |
| PUT | `/facility/approve/:id` | protect | Approve a facility |
| PUT | `/facility/reject/:id` | protect | Reject a facility |
| GET | `/donors` | None | List all donors |

### Facility Routes (`/api/facility`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/labs` | protectFacility | Get all approved blood labs (used by hospitals) |

---

## 10. How to Run the Project

### Prerequisites
- Node.js 18+
- MongoDB (local or Atlas)
- npm or yarn

### Backend Setup
```bash
cd backend
cp .env.example .env
# Edit .env: set MONGO_URI and JWT_SECRET

npm install
node seedAdmin.js    # Create the admin account (one time)
npm start            # or: node server.js
# Server runs on http://localhost:5000
```

### Frontend Setup
```bash
cd frontend
# Create .env file:
echo "VITE_API_URL=http://localhost:5000/api" > .env
echo "VITE_WEBSITE_NAME=BloodBank" >> .env

npm install
npm run dev
# App runs on http://localhost:5173
```

### Docker (Alternative)
```bash
# From project root
docker-compose up --build
```

---

## 11. How to Test Everything from the Frontend

### Test 1: Admin Login
- URL: `/login`
- Email: `admin@bbms.com`, Password: `bbms@admin`
- Expected: redirected to `/admin` dashboard with system stats

### Test 2: Register a Blood Lab
- URL: `/register/facility`
- Fill all fields, select Facility Type: **Blood Lab**
- Expected: success message "awaiting admin approval"
- Verify: try logging in as the lab → should be blocked with "awaiting approval" message

### Test 3: Admin Approves the Lab
- Log in as admin → Sidebar: **Verification**
- Find the lab with "pending" status → click **Approve**
- Expected: status changes to "approved" in the table

### Test 4: Blood Lab Login and Dashboard
- Log in as the approved blood lab
- Expected: redirected to `/lab`, dashboard shows 0 camps, 0 stock, 0 donors

### Test 5: Create a Blood Camp
- Sidebar: **Camps** → click **Add Camp**
- Fill: title, future date, start/end time, venue, city, state, 6-digit pincode, expected donors
- Expected: camp appears in the list with status "Upcoming"

### Test 6: Add Blood Stock
- Sidebar: **Inventory**
- Select blood type "O+", quantity 50, action "Add Stock" → click **Add Units**
- Expected: "O+" row appears in the inventory table with 50 units

### Test 7: Register a Donor
- URL: `/register/donor`
- Fill all fields including blood group, age (18–65), weight (min 45kg), address
- Expected: immediately redirected to `/donor` dashboard

### Test 8: Donor Views Blood Camps
- Log in as donor → Sidebar: **Blood Camps**
- Expected: the camp created in Test 5 appears with "Upcoming" status

### Test 9: Register and Approve a Hospital
- URL: `/register/facility`, select Facility Type: **Hospital**
- Admin approves it (same as Test 3)

### Test 10: Hospital Requests Blood
- Log in as hospital → Sidebar: **Blood Requests**
- Select the blood lab, choose blood type "O+", enter 10 units → **Send Blood Request**
- Expected: request appears in history with status "pending"

### Test 11: Blood Lab Accepts the Request
- Log in as blood lab → Sidebar: **Requests**
- Find the pending request → click **Accept**
- Expected: request status changes to "accepted"
- Verify: lab's O+ stock decreased by 10

### Test 12: Hospital Checks Inventory
- Log in as hospital → Sidebar: **Inventory**
- Expected: O+ blood with 10 units appears in the hospital's inventory

### Test 13: Blood Lab Records a Donor Donation
- Log in as blood lab → Sidebar: **Donors**
- Search for the donor by name → click **Donate**
- Confirm blood group, quantity 1 → **Confirm Donation**
- Expected: success toast, donor's last donation date updated

### Test 14: Donor Checks Donation History
- Log in as donor → Sidebar: **Donation History**
- Expected: the donation recorded in Test 13 appears in the list

### Test 15: Update Camp Status
- Log in as blood lab → Sidebar: **Camps**
- Click the action menu (⋮) on a camp → **Mark as Ongoing**
- Expected: status badge changes to "Ongoing"

---

## 12. Key Design Decisions

### Single Facility Model for Hospitals and Labs
Both hospitals and blood labs use the same `Facility` Mongoose model, differentiated by the `facilityType` field. This simplifies auth (one middleware handles both) and avoids duplicating schema fields like address, operating hours, and history.

### Embedded Donation History vs. Separate Collection
Donor donation history is stored as an embedded array inside the Donor document. This makes profile fetches fast (single query) but requires MongoDB aggregation for paginated history queries. The `getDonorHistory` controller uses `$unwind` + `$skip` + `$limit` for this.

### JWT Role-Based Routing
After login, the backend returns a `redirect` field (e.g., `/donor`, `/lab`, `/hospital`, `/admin`). The frontend uses this to navigate to the correct dashboard. The `DashboardLayout` component double-checks the role on every page load by calling `/api/auth/profile` and comparing the returned role against the expected `userRole` prop.

### Blood Stock Ownership
The `Blood` model has two optional fields: `bloodLab` and `hospital`. A pre-save hook enforces that exactly one is set. This allows a single collection to track stock for both entity types, with separate indexes for efficient querying.

### 90-Day Donation Cooldown
Enforced in two places:
1. **Backend** (`markDonation`): checks `lastDonationDate` before recording a new donation
2. **Frontend** (`BloodLabDonor`): the "Donate" button is disabled if the donor donated within 3 months
3. **Donor model**: a virtual field `isEligible` computes this for display in the donor's own profile

### Facility Approval Gate
Facilities (hospitals and labs) cannot log in until an admin approves them. The login controller checks `user.status` and returns HTTP 403 with a descriptive message if the status is "pending" or "rejected". The frontend login page handles these specific messages and shows appropriate UI feedback.
