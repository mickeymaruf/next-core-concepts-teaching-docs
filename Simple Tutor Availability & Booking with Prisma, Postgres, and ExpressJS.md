# Simple Tutor Availability & Booking with Prisma, Postgres, and ExpressJS

---

# Core Idea

* **Tutor creates time slots** (availability)
* **Student books one slot**
* **Slot becomes unavailable once booked**

No calendars, no recurrence, no over-engineering.

---

## Prisma Schema (Minimal)

### 1️⃣ User

```prisma
model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  password  String
  role      Role
  createdAt DateTime @default(now())

  tutorProfile TutorProfile?
  bookings     Booking[] @relation("StudentBookings")
}
```

```prisma
enum Role {
  STUDENT
  TUTOR
  ADMIN
}
```

---

### 2️⃣ TutorProfile

```prisma
model TutorProfile {
  id        Int      @id @default(autoincrement())
  userId    Int      @unique
  bio       String?
  price     Int

  user        User          @relation(fields: [userId], references: [id])
  slots       Availability[]
  bookings    Booking[]     @relation("TutorBookings")
}
```

---

### 3️⃣ Availability (Time Slots)

Each row = **one bookable slot**

```prisma
model Availability {
  id        Int      @id @default(autoincrement())
  tutorId   Int
  startTime DateTime
  endTime   DateTime
  isBooked  Boolean  @default(false)

  tutor TutorProfile @relation(fields: [tutorId], references: [id])
  booking Booking?
}
```

---

### 4️⃣ Booking

```prisma
model Booking {
  id           Int      @id @default(autoincrement())
  studentId    Int
  tutorId      Int
  slotId       Int      @unique
  status       BookingStatus @default(CONFIRMED)
  createdAt    DateTime @default(now())

  student User          @relation("StudentBookings", fields: [studentId], references: [id])
  tutor   TutorProfile  @relation("TutorBookings", fields: [tutorId], references: [id])
  slot    Availability  @relation(fields: [slotId], references: [id])
}
```

```prisma
enum BookingStatus {
  CONFIRMED
  COMPLETED
  CANCELLED
}
```

---

# Flow (Straightforward)

## 🧑‍🏫 Tutor: Set Availability

Tutor submits time slots:

```json
[
  { "startTime": "2026-02-01T10:00:00Z", "endTime": "2026-02-01T11:00:00Z" },
  { "startTime": "2026-02-01T11:00:00Z", "endTime": "2026-02-01T12:00:00Z" }
]
```

### Express Route

```js
POST /api/tutor/availability
```

### Logic

```js
await prisma.availability.createMany({
  data: slots.map(slot => ({
    tutorId,
    startTime: slot.startTime,
    endTime: slot.endTime
  }))
})
```

✔ Tutor just creates rows
✔ No conflict logic needed (keep it simple)

---

## 👨‍🎓 Student: View Available Slots

```js
GET /api/tutors/:id/availability
```

```js
await prisma.availability.findMany({
  where: {
    tutorId,
    isBooked: false,
    startTime: { gt: new Date() }
  }
})
```

---

## 👨‍🎓 Student: Book a Slot

```js
POST /api/bookings
```

```json
{
  "slotId": 12
}
```

### Booking Logic (Transaction)

```js
await prisma.$transaction(async (tx) => {
  const slot = await tx.availability.findUnique({
    where: { id: slotId }
  })

  if (!slot || slot.isBooked) {
    throw new Error("Slot not available")
  }

  await tx.booking.create({
    data: {
      studentId,
      tutorId: slot.tutorId,
      slotId: slot.id
    }
  })

  await tx.availability.update({
    where: { id: slot.id },
    data: { isBooked: true }
  })
})
```

✔ Prevents double booking
✔ Very easy to reason about

---

## ❌ Cancel Booking (Optional)

```js
PATCH /api/bookings/:id/cancel
```

```js
await prisma.$transaction(async (tx) => {
  const booking = await tx.booking.update({
    where: { id },
    data: { status: "CANCELLED" }
  })

  await tx.availability.update({
    where: { id: booking.slotId },
    data: { isBooked: false }
  })
})
```

---

# Why This Is the Simplest Possible Design

* ❌ No recurring schedules
* ❌ No complex availability rules
* ❌ No overlapping logic
* ✔ One slot = one booking
* ✔ Easy queries
* ✔ Easy UI

This is **perfect for SkillBridge MVP** and scores well in reviews.

---

# Auto generating Daily Hourly Slots for Tutor Availability

---

Short answer: **yes — but the system can do it for them.**

Tutor should **NOT manually create every slot**.

Here’s the **simplest clean approach** that still feels good to use.

---

## Best Simple Pattern (MVP-friendly)

### Tutor provides:

* **Working days**
* **Daily time range**
* **Slot duration**

Example:

```json
{
  "days": ["MON", "TUE", "WED"],
  "startTime": "09:00",
  "endTime": "17:00",
  "slotDuration": 60
}
```

👉 Backend **auto-generates slots** for the next *N days* (ex: next 14 or 30 days).

---

## How It Works (Plain Logic)

1. Tutor saves availability rule
2. Backend:

   * Loops days
   * Splits time range into slots
   * Inserts rows into `Availability`
3. Students book slots
4. Booked slots are locked

No calendar syncing. No cron jobs. No magic.

---

## Minimal Prisma Additions

### Availability stays the same

```prisma
model Availability {
  id        Int      @id @default(autoincrement())
  tutorId   Int
  startTime DateTime
  endTime   DateTime
  isBooked  Boolean  @default(false)
}
```

### Optional (but helpful): AvailabilityRule

```prisma
model AvailabilityRule {
  id           Int   @id @default(autoincrement())
  tutorId      Int
  days         String // "MON,TUE,WED"
  startTime    String // "09:00"
  endTime      String // "17:00"
  slotDuration Int    // minutes
}
```

(You can also skip this table and generate slots immediately.)

---

## Slot Generation (Very Simple JS)

```js
function generateSlots(rule, daysAhead = 14) {
  const slots = []

  for (let i = 0; i < daysAhead; i++) {
    const date = new Date()
    date.setDate(date.getDate() + i)

    const day = date.toLocaleString('en-US', { weekday: 'short' }).toUpperCase()
    if (!rule.days.includes(day)) continue

    let start = new Date(`${date.toDateString()} ${rule.startTime}`)
    const end = new Date(`${date.toDateString()} ${rule.endTime}`)

    while (start < end) {
      const slotEnd = new Date(start.getTime() + rule.slotDuration * 60000)

      slots.push({
        tutorId: rule.tutorId,
        startTime: start,
        endTime: slotEnd
      })

      start = slotEnd
    }
  }

  return slots
}
```

Then:

```js
await prisma.availability.createMany({ data: slots })
```

---

## UX Result (Clean)

Tutor UI:

* Select days ✔
* Pick start/end time ✔
* Pick duration ✔
* Click **Save**

Student UI:

* Sees real bookable times
* Click → Book

---

## Why This Is the Right Balance

* Tutor effort = **one form**
* Database stays **dead simple**
* Booking logic stays **safe**
* Easy to extend later

---

### TL;DR

* ❌ Tutor should NOT manually add every slot
* ✔ Tutor defines **rules**
* ✔ Backend generates slots automatically
* ✔ Slots are just rows in `Availability`

---

Handling Unbooked Tutor Slots: Should Daily Availability Stay in the Database or Be Removed?

---

Yes — **they stay in the DB**.
That’s the simplest and most correct answer.

Now the *important part* is **how you clean them up** without complexity.

---

## What Happens to Unbooked Slots?

### ✅ They remain in `Availability`

Each slot is just a row like:

```txt
tutorId | startTime | endTime | isBooked=false
```

If nobody books it → **nothing bad happens**.

---

## How You Handle Old / Expired Slots (Very Simple)

### Rule

> **Any slot whose `endTime` is in the past is irrelevant**

You **never show** past slots to students.

```js
where: {
  isBooked: false,
  startTime: { gt: new Date() }
}
```

That alone solves 90% of the concern.

---

## Do You Delete Old Slots?

### Option 1 (Recommended for MVP): ❌ Don’t delete

* Leave them in DB
* Ignore past slots in queries
* Zero maintenance
* Safe and easy

This is totally fine for **thousands of slots**.

---

### Option 2 (Simple Cleanup – Optional)

Run a daily cleanup (cron or manual):

```js
await prisma.availability.deleteMany({
  where: {
    isBooked: false,
    endTime: { lt: new Date() }
  }
})
```

* Keeps DB small
* Not required early on

---

## What About Booked Slots?

❗ **Never delete booked slots**

They are needed for:

* booking history
* reviews
* admin audit

Booked slots stay forever (or archived later).

---

## Key Design Rule (Important)

* **Availability = future options**
* **Booking = historical record**

Once booked:

* Slot → locked
* Booking → source of truth

---

## Scaling Reality (No Overthinking)

Example:

* 1 tutor
* 7 days × 11 hours = 77 slots
* 1,000 tutors → 77,000 rows

Postgres handles this **easily**.

---

## TL;DR

* ✔ Yes, slots stay in DB
* ✔ Past unbooked slots are ignored
* ❌ No need to delete immediately
* 🧹 Optional cleanup later
* ❗ Booked slots are never deleted

This is **normal industry practice**, not a hack.

---

# How Platforms Like Airbnb and Calendly Manage Availability and Time Slots

---

Absolutely! Let’s do this **cleanly**, Airbnb/Calendly style — using **availability rules + exceptions**, minimal DB rows, and **on-demand slot generation**.

---

# 1️⃣ Prisma Schema (Postgres)

```prisma
model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  password  String
  role      Role
  tutorProfile TutorProfile?
  bookings     Booking[]
  createdAt DateTime @default(now())
}

enum Role {
  STUDENT
  TUTOR
  ADMIN
}

model TutorProfile {
  id        Int      @id @default(autoincrement())
  userId    Int      @unique
  bio       String?
  price     Int
  user      User      @relation(fields: [userId], references: [id])
  bookings  Booking[]
  rules     AvailabilityRule[]
}

model AvailabilityRule {
  id           Int      @id @default(autoincrement())
  tutorId      Int
  days         String   // e.g., "MON,TUE,WED"
  startTime    String   // e.g., "10:00"
  endTime      String   // e.g., "18:00"
  slotDuration Int      // in minutes
  tutor        TutorProfile @relation(fields: [tutorId], references: [id])
  exceptions   AvailabilityException[]
}

model AvailabilityException {
  id        Int      @id @default(autoincrement())
  ruleId    Int
  date      DateTime // The day that is blocked
  tutor     AvailabilityRule @relation(fields: [ruleId], references: [id])
}

model Booking {
  id        Int      @id @default(autoincrement())
  studentId Int
  tutorId   Int
  startTime DateTime
  endTime   DateTime
  status    BookingStatus @default(CONFIRMED)
  createdAt DateTime @default(now())
  student   User      @relation(fields: [studentId], references: [id])
  tutor     TutorProfile @relation(fields: [tutorId], references: [id])
}

enum BookingStatus {
  CONFIRMED
  COMPLETED
  CANCELLED
}
```

---

# 2️⃣ How Slots Are Generated (Backend)

We **don’t store slots**, we **compute them dynamically** when a student requests availability.

```js
// utils/generateSlots.js
function generateSlots(rule, exceptions, daysAhead = 7) {
  const slots = [];
  const daysOfWeek = rule.days.split(','); // ["MON","TUE"]

  for (let i = 0; i < daysAhead; i++) {
    const date = new Date();
    date.setDate(date.getDate() + i);
    const dayName = date.toLocaleString('en-US', { weekday: 'short' }).toUpperCase();

    if (!daysOfWeek.includes(dayName)) continue;

    // Skip if this date is in exceptions
    if (exceptions.some(e => e.date.toDateString() === date.toDateString())) continue;

    let [startHour, startMin] = rule.startTime.split(':').map(Number);
    let [endHour, endMin] = rule.endTime.split(':').map(Number);

    let start = new Date(date);
    start.setHours(startHour, startMin, 0, 0);

    const end = new Date(date);
    end.setHours(endHour, endMin, 0, 0);

    while (start < end) {
      const slotEnd = new Date(start.getTime() + rule.slotDuration * 60000);
      slots.push({ startTime: new Date(start), endTime: new Date(slotEnd) });
      start = slotEnd;
    }
  }

  return slots;
}
```

---

# 3️⃣ Express Route Example

```js
// GET /api/tutors/:id/available-slots
app.get("/api/tutors/:id/available-slots", async (req, res) => {
  const tutorId = parseInt(req.params.id);

  // 1️⃣ Fetch rules + exceptions
  const rules = await prisma.availabilityRule.findMany({
    where: { tutorId },
    include: { exceptions: true }
  });

  // 2️⃣ Fetch booked slots in the next 7 days
  const now = new Date();
  const weekLater = new Date();
  weekLater.setDate(now.getDate() + 7);

  const bookings = await prisma.booking.findMany({
    where: {
      tutorId,
      startTime: { gte: now, lte: weekLater },
      status: "CONFIRMED"
    }
  });

  // 3️⃣ Generate slots
  let slots = [];
  for (let rule of rules) {
    const ruleSlots = generateSlots(rule, rule.exceptions, 7);

    // Remove slots already booked
    const availableSlots = ruleSlots.filter(s =>
      !bookings.some(b => b.startTime.getTime() === s.startTime.getTime())
    );

    slots.push(...availableSlots);
  }

  res.json(slots);
});
```

---

# 4️⃣ Booking Route

```js
// POST /api/bookings
app.post("/api/bookings", async (req, res) => {
  const { studentId, tutorId, startTime, endTime } = req.body;

  // 1️⃣ Check if slot is free
  const conflict = await prisma.booking.findFirst({
    where: { tutorId, startTime: new Date(startTime), status: "CONFIRMED" }
  });

  if (conflict) return res.status(400).json({ error: "Slot already booked" });

  // 2️⃣ Create booking
  const booking = await prisma.booking.create({
    data: { studentId, tutorId, startTime: new Date(startTime), endTime: new Date(endTime) }
  });

  res.json(booking);
});
```

---

# ✅ Key Advantages

1. **DB stays tiny** – no thousands of pre-generated slots
2. **Dynamic** – supports exceptions easily
3. **Bookings remain single source of truth**
4. **Easy to scale** – just generate next 7 days of slots on request

---
