# fleet
Fleet Management System (Node.js + Express + Supabase)
✅ Mandatory Folder Structure
root/
│
├── models/
│   ├── user.model.md
│   ├── vehicle.model.md
│   └── trip.model.md
│
├── routes/
│   ├── user.routes.js
│   ├── vehicle.routes.js
│   ├── trip.routes.js
│   └── analytics.routes.js
│
├── controllers/
│   ├── user.controller.js
│   ├── vehicle.controller.js
│   ├── trip.controller.js
│   └── analytics.controller.js
│
├── middlewares/
│   ├── logger.middleware.js
│   ├── rateLimiter.middleware.js
│   └── notFound.middleware.js
│
├── config/
│   └── supabaseClient.js
│
├── logs.txt
├── .env
├── index.js
└── package.json

0️⃣ Setup
✅ Install Dependencies
npm init -y
npm i express dotenv @supabase/supabase-js

✅ .env
PORT=3000
SUPABASE_URL=YOUR_URL
SUPABASE_KEY=YOUR_ANON_KEY

✅ config/supabaseClient.js
const { createClient } = require("@supabase/supabase-js");
require("dotenv").config();

const supabase = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_KEY);

module.exports = supabase;

1️⃣ Supabase Schema (Tables + Constraints)

Create tables in Supabase Table Editor with proper constraints + relationships.

✅ Table: users
Column	Type	Constraints
id	uuid	PK, default uuid_generate_v4()
name	text	not null
email	text	unique, not null
password	text	not null
role	text	not null (customer/owner/driver only)
created_at	timestamptz	default now()
✅ Role Validation

Create CHECK constraint

(role IN ('customer','owner','driver'))

✅ Table: vehicles
Column	Type	Constraints
id	uuid	PK
name	text	not null
registration_number	text	unique, not null
allowed_passengers	int4	not null, check > 0
isAvailable	boolean	default true
driver_id	uuid	FK → users(id), nullable
rate_per_km	numeric	not null, check > 0
owner_id	uuid	FK → users(id), not null
created_at	timestamptz	default now()
✅ Table: trips
Column	Type	Constraints
id	uuid	PK
customer_id	uuid	FK → users(id), not null
vehicle_id	uuid	FK → vehicles(id), not null
start_date	date	not null
end_date	date	not null
location	text	not null
distance_km	numeric	not null, check > 0
passengers	int4	not null
tripCost	numeric	default 0
isCompleted	boolean	default false
created_at	timestamptz	default now()
2️⃣ Express Server Setup
✅ index.js
const express = require("express");
require("dotenv").config();

const userRoutes = require("./routes/user.routes");
const vehicleRoutes = require("./routes/vehicle.routes");
const tripRoutes = require("./routes/trip.routes");
const analyticsRoutes = require("./routes/analytics.routes");

const loggerMiddleware = require("./middlewares/logger.middleware");
const notFoundMiddleware = require("./middlewares/notFound.middleware");

const app = express();
app.use(express.json());

// logger middleware (for every request)
app.use(loggerMiddleware);

app.use("/users", userRoutes);
app.use("/vehicles", vehicleRoutes);
app.use("/trips", tripRoutes);
app.use("/", analyticsRoutes);

// 404 undefined routes
app.use(notFoundMiddleware);

app.listen(process.env.PORT, () => {
  console.log(`Server running on port ${process.env.PORT}`);
});

3️⃣ Middleware
✅ middlewares/logger.middleware.js
const fs = require("fs");

module.exports = (req, res, next) => {
  const log = `${req.method} ${req.originalUrl} ${new Date().toISOString()}\n`;

  fs.appendFile("logs.txt", log, (err) => {
    if (err) console.log("Log error:", err);
  });

  next();
};

✅ middlewares/rateLimiter.middleware.js (ONLY for create vehicle)

Max 3 requests per minute per IP

const requestMap = new Map();

module.exports = (req, res, next) => {
  const ip = req.ip;
  const now = Date.now();

  if (!requestMap.has(ip)) requestMap.set(ip, []);
  const timestamps = requestMap.get(ip);

  // keep only last 60 seconds
  const updated = timestamps.filter((t) => now - t < 60000);
  updated.push(now);
  requestMap.set(ip, updated);

  if (updated.length > 3) {
    return res.status(429).json({ message: "Too many requests. Try after 1 minute." });
  }

  next();
};

✅ middlewares/notFound.middleware.js
module.exports = (req, res) => {
  res.status(404).json({ message: "This Request Is Not Found" });
};

4️⃣ User Module
✅ routes/user.routes.js
const express = require("express");
const router = express.Router();
const { signup } = require("../controllers/user.controller");

router.post("/signup", signup);

module.exports = router;

✅ controllers/user.controller.js
const supabase = require("../config/supabaseClient");

exports.signup = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;

    if (!name || !email || !password || !role)
      return res.status(400).json({ message: "All fields are required" });

    const validRoles = ["customer", "owner", "driver"];
    if (!validRoles.includes(role))
      return res.status(400).json({ message: "Invalid role" });

    const { data: existingUser } = await supabase
      .from("users")
      .select("id")
      .eq("email", email)
      .single();

    if (existingUser) return res.status(409).json({ message: "Email already exists" });

    const { data, error } = await supabase.from("users").insert([
      { name, email, password, role },
    ]).select();

    if (error) return res.status(500).json({ message: error.message });

    res.status(201).json({ message: "User registered successfully", user: data[0] });
  } catch (err) {
    res.status(500).json({ message: "Server error", error: err.message });
  }
};

5️⃣ Vehicle Module (Owner Only)
Expected Routes

✅ POST /vehicles/add/
✅ PATCH /vehicles/assign-driver/:vehicleId
✅ GET /vehicles/:vehicleId

✅ routes/vehicle.routes.js
const express = require("express");
const router = express.Router();
const rateLimiter = require("../middlewares/rateLimiter.middleware");
const {
  addVehicle,
  assignDriver,
  getVehicleById,
} = require("../controllers/vehicle.controller");

router.post("/add", rateLimiter, addVehicle);
router.patch("/assign-driver/:vehicleId", assignDriver);
router.get("/:vehicleId", getVehicleById);

module.exports = router;

✅ controllers/vehicle.controller.js
const supabase = require("../config/supabaseClient");

exports.addVehicle = async (req, res) => {
  try {
    const { name, registration_number, allowed_passengers, rate_per_km, owner_id } =
      req.body;

    if (!name || !registration_number || !allowed_passengers || !rate_per_km || !owner_id)
      return res.status(400).json({ message: "All fields are required" });

    // Check owner role
    const { data: owner } = await supabase
      .from("users")
      .select("id, role")
      .eq("id", owner_id)
      .single();

    if (!owner) return res.status(404).json({ message: "Owner not found" });
    if (owner.role !== "owner")
      return res.status(403).json({ message: "Only owners can create vehicles" });

    const { data, error } = await supabase.from("vehicles").insert([
      { name, registration_number, allowed_passengers, rate_per_km, owner_id },
    ]).select();

    if (error) return res.status(500).json({ message: error.message });

    res.status(201).json({ message: "Vehicle created", vehicle: data[0] });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};

exports.assignDriver = async (req, res) => {
  try {
    const { vehicleId } = req.params;
    const { driver_id } = req.body;

    if (!driver_id) return res.status(400).json({ message: "driver_id required" });

    // validate driver
    const { data: driver } = await supabase
      .from("users")
      .select("id, role")
      .eq("id", driver_id)
      .single();

    if (!driver) return res.status(404).json({ message: "Driver not found" });
    if (driver.role !== "driver")
      return res.status(403).json({ message: "Only driver role can be assigned" });

    const { data, error } = await supabase
      .from("vehicles")
      .update({ driver_id })
      .eq("id", vehicleId)
      .select();

    if (error) return res.status(500).json({ message: error.message });

    res.status(200).json({ message: "Driver assigned", vehicle: data[0] });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};

exports.getVehicleById = async (req, res) => {
  try {
    const { vehicleId } = req.params;

    const { data, error } = await supabase
      .from("vehicles")
      .select("*")
      .eq("id", vehicleId)
      .single();

    if (!data) return res.status(404).json({ message: "Vehicle not found" });
    if (error) return res.status(500).json({ message: error.message });

    res.status(200).json({ vehicle: data });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};

6️⃣ Trip Module (Customer Only)
Expected Routes

✅ POST /trips/create/
✅ PATCH /trips/update/:tripId
✅ GET /trips/:tripId
✅ DELETE /trips/delete/:tripId

✅ routes/trip.routes.js
const express = require("express");
const router = express.Router();

const {
  createTrip,
  updateTrip,
  getTrip,
  deleteTrip,
  endTrip,
} = require("../controllers/trip.controller");

router.post("/create", createTrip);
router.patch("/update/:tripId", updateTrip);
router.get("/:tripId", getTrip);
router.delete("/delete/:tripId", deleteTrip);

// end trip special route
router.patch("/end/:tripId", endTrip);

module.exports = router;

✅ controllers/trip.controller.js
const supabase = require("../config/supabaseClient");

exports.createTrip = async (req, res) => {
  try {
    const {
      customer_id,
      vehicle_id,
      start_date,
      end_date,
      location,
      distance_km,
      passengers,
    } = req.body;

    if (
      !customer_id ||
      !vehicle_id ||
      !start_date ||
      !end_date ||
      !location ||
      !distance_km ||
      !passengers
    )
      return res.status(400).json({ message: "All fields are required" });

    // validate customer
    const { data: customer } = await supabase
      .from("users")
      .select("id, role")
      .eq("id", customer_id)
      .single();

    if (!customer) return res.status(404).json({ message: "Customer not found" });
    if (customer.role !== "customer")
      return res.status(403).json({ message: "Only customers can create trips" });

    // validate vehicle availability & capacity
    const { data: vehicle } = await supabase
      .from("vehicles")
      .select("id, isAvailable, allowed_passengers")
      .eq("id", vehicle_id)
      .single();

    if (!vehicle) return res.status(404).json({ message: "Vehicle not found" });

    if (vehicle.isAvailable === false)
      return res.status(400).json({ message: "Vehicle not available" });

    if (passengers > vehicle.allowed_passengers)
      return res.status(400).json({ message: "Passengers exceed allowed limit" });

    // create trip
    const { data: tripData, error: tripError } = await supabase
      .from("trips")
      .insert([
        {
          customer_id,
          vehicle_id,
          start_date,
          end_date,
          location,
          distance_km,
          passengers,
        },
      ])
      .select();

    if (tripError) return res.status(500).json({ message: tripError.message });

    // set vehicle unavailable
    await supabase
      .from("vehicles")
      .update({ isAvailable: false })
      .eq("id", vehicle_id);

    res.status(201).json({ message: "Trip created", trip: tripData[0] });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};

exports.getTrip = async (req, res) => {
  try {
    const { tripId } = req.params;

    const { data, error } = await supabase
      .from("trips")
      .select("*")
      .eq("id", tripId)
      .single();

    if (!data) return res.status(404).json({ message: "Trip not found" });
    if (error) return res.status(500).json({ message: error.message });

    res.status(200).json({ trip: data });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};

exports.updateTrip = async (req, res) => {
  try {
    const { tripId } = req.params;
    const updates = req.body;

    const { data, error } = await supabase
      .from("trips")
      .update(updates)
      .eq("id", tripId)
      .select();

    if (!data || data.length === 0)
      return res.status(404).json({ message: "Trip not found" });

    if (error) return res.status(500).json({ message: error.message });

    res.status(200).json({ message: "Trip updated", trip: data[0] });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};

exports.deleteTrip = async (req, res) => {
  try {
    const { tripId } = req.params;

    const { data: trip } = await supabase
      .from("trips")
      .select("vehicle_id")
      .eq("id", tripId)
      .single();

    if (!trip) return res.status(404).json({ message: "Trip not found" });

    await supabase.from("trips").delete().eq("id", tripId);

    // restore vehicle availability
    await supabase
      .from("vehicles")
      .update({ isAvailable: true })
      .eq("id", trip.vehicle_id);

    res.status(200).json({ message: "Trip deleted successfully" });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};

exports.endTrip = async (req, res) => {
  try {
    const { tripId } = req.params;

    const { data: trip } = await supabase
      .from("trips")
      .select("id, vehicle_id, distance_km, isCompleted")
      .eq("id", tripId)
      .single();

    if (!trip) return res.status(404).json({ message: "Trip not found" });
    if (trip.isCompleted) return res.status(400).json({ message: "Trip already ended" });

    const { data: vehicle } = await supabase
      .from("vehicles")
      .select("rate_per_km")
      .eq("id", trip.vehicle_id)
      .single();

    const cost = Number(trip.distance_km) * Number(vehicle.rate_per_km);

    // update trip completed
    const { data: updatedTrip } = await supabase
      .from("trips")
      .update({ isCompleted: true, tripCost: cost })
      .eq("id", tripId)
      .select();

    // set vehicle available
    await supabase
      .from("vehicles")
      .update({ isAvailable: true })
      .eq("id", trip.vehicle_id);

    res.status(200).json({ message: "Trip ended", trip: updatedTrip[0] });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};

7️⃣ Analytics Endpoint
✅ Expected Route

✅ GET /analytics

✅ routes/analytics.routes.js
const express = require("express");
const router = express.Router();
const { getAnalytics } = require("../controllers/analytics.controller");

router.get("/analytics", getAnalytics);

module.exports = router;

✅ controllers/analytics.controller.js
const supabase = require("../config/supabaseClient");

exports.getAnalytics = async (req, res) => {
  try {
    const { count: totalCustomers } = await supabase
      .from("users")
      .select("*", { count: "exact", head: true })
      .eq("role", "customer");

    const { count: totalOwners } = await supabase
      .from("users")
      .select("*", { count: "exact", head: true })
      .eq("role", "owner");

    const { count: totalDrivers } = await supabase
      .from("users")
      .select("*", { count: "exact", head: true })
      .eq("role", "driver");

    const { count: totalVehicles } = await supabase
      .from("vehicles")
      .select("*", { count: "exact", head: true });

    const { count: totalTrips } = await supabase
      .from("trips")
      .select("*", { count: "exact", head: true });

    res.status(200).json({
      totalCustomers,
      totalOwners,
      totalDrivers,
      totalVehicles,
      totalTrips,
    });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};


✅ This uses Supabase DB count queries (NO JS loops).

8️⃣ Schema Documentation Files (models/*.md)
✅ models/user.model.md
# User Model

## Table Name: users

### Columns
- id: UUID (PK, default uuid_generate_v4())
- name: TEXT NOT NULL
- email: TEXT UNIQUE NOT NULL
- password: TEXT NOT NULL
- role: TEXT NOT NULL CHECK (role IN ('customer','owner','driver'))
- created_at: TIMESTAMPTZ DEFAULT now()

### Relationships
- users.id referenced by vehicles.owner_id
- users.id referenced by vehicles.driver_id
- users.id referenced by trips.customer_id

✅ models/vehicle.model.md
# Vehicle Model

## Table Name: vehicles

### Columns
- id: UUID (PK)
- name: TEXT NOT NULL
- registration_number: TEXT UNIQUE NOT NULL
- allowed_passengers: INT NOT NULL CHECK (allowed_passengers > 0)
- isAvailable: BOOLEAN DEFAULT true
- driver_id: UUID NULL REFERENCES users(id)
- rate_per_km: NUMERIC NOT NULL CHECK (rate_per_km > 0)
- owner_id: UUID NOT NULL REFERENCES users(id)
- created_at: TIMESTAMPTZ DEFAULT now()

### Relationships
- Owner (users.role='owner') -> Vehicles
- Driver (users.role='driver') -> Vehicle

✅ models/trip.model.md
# Trip Model

## Table Name: trips

### Columns
- id: UUID (PK)
- customer_id: UUID NOT NULL REFERENCES users(id)
- vehicle_id: UUID NOT NULL REFERENCES vehicles(id)
- start_date: DATE NOT NULL
- end_date: DATE NOT NULL
- location: TEXT NOT NULL
- distance_km: NUMERIC NOT NULL CHECK (distance_km > 0)
- passengers: INT NOT NULL
- tripCost: NUMERIC DEFAULT 0
- isCompleted: BOOLEAN DEFAULT false
- created_at: TIMESTAMPTZ DEFAULT now()

### Relationships
- Customer (users.role='customer') -> Trips
- Vehicle -> Trips
