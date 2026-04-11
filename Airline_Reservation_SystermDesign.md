# Airline Ticket Reservation – System Design Peculiarities

This document focuses on **technical aspects specific to airline reservation systems** that differ from
concert-style ticket booking systems (e.g. Ticketmaster).

It assumes familiarity with concepts already covered in the Ticketmaster design:

- idempotent booking flows
- transactional reservation logic
- retry and failure handling
- high-level distributed architecture

Therefore this document focuses on **airline-specific architecture and algorithms**, particularly:

- yield management
- dynamic pricing curves
- probabilistic overbooking
- inventory synchronization
- distributed scaling considerations

---

# 1. Core Concept: Inventory Is Not Seats

In concert ticket systems:

seat = product

In airline systems:

transportation = product  
seat = attribute

Airlines typically **do not sell specific seats initially**.

Instead they sell **capacity within fare buckets**.

Example inventory:

| Fare Class | Description | Available |
|------------|-------------|----------|
| Y | Economy flexible | 50 |
| M | Economy standard | 40 |
| Q | Economy saver | 20 |
| C | Business | 12 |

Booking consumes **capacity**, not seat rows.

Seat assignment may occur later during:

- booking
- online check-in
- airport check-in
- gate assignment

This eliminates the **seat-level contention problem** common in concert systems.

---

# 2. Inventory Model

Simplified schema:

flights  
flight_id  
departure_time  
aircraft_type  
physical_capacity

fare_inventory  
flight_id  
fare_class  
capacity  
sold  
overbooking_limit

Availability formula:

available = capacity + overbooking_limit - sold

Example:

capacity = 180  
overbooking_limit = 9  
sold = 120

available = 69

Inventory updates are **numeric operations**, not seat-row locks.

---

# 3. Yield Management

Airline systems optimize **revenue**, not simply seat occupancy.

Yield management dynamically adjusts **which fare buckets are available**.

Example allocation:

Total seats = 180

Business: 20  
Economy flexible: 60  
Economy standard: 70  
Economy saver: 30

Cheap fares are gradually restricted as the flight fills.

Example rule:

if remaining_seats < threshold → close cheapest_fare_bucket

Inputs to yield management algorithms:

- historical demand curves
- booking velocity
- days until departure
- seasonal demand
- competitor pricing

Common algorithmic approaches include:

- EMSR (Expected Marginal Seat Revenue)
- demand forecasting models
- statistical revenue optimization
- reinforcement learning pricing models

---

# 4. Dynamic Pricing Curves

Airline ticket prices change continuously.

Simplified pricing model:

price = base_price × demand_multiplier × time_multiplier

Where:

demand_multiplier = f(booking_rate)  
time_multiplier = f(days_to_departure)

Typical pattern:

Early sales → cheaper  
Mid period → moderate  
Late sales → expensive

Pricing engines normally run **outside the reservation transaction** to avoid latency during booking.

---

# 5. Overbooking Model

Airlines intentionally sell more tickets than seats.

Reason: passenger **no-show probability**.

Example:

physical seats = 180  
expected no-show rate = 5%

Allowed overbooking:

180 × 0.05 = 9 seats

Sellable inventory:

180 + 9 = 189

Therefore the system allows:

sold > aircraft_capacity

Operational recovery mechanisms handle situations where more passengers arrive than available seats.

---

# 6. Passenger Name Record (PNR)

Airline bookings revolve around the **PNR object**.

Example structure:

pnr  
pnr_id  
status  
created_at

pnr_passengers  
pnr_id  
passenger_id

pnr_segments  
pnr_id  
flight_id  
fare_class

PNR acts as:

- reservation container
- payment reference
- ticket issuance anchor

Multiple passengers can share one PNR.

---

# 7. Seat Assignment Decoupling

Seat allocation may be delayed.

Two common strategies:

Immediate seat selection  
Passenger chooses seat during booking.

Deferred seat assignment  
Seat assigned during:

online check-in  
airport check-in  
gate assignment

Delaying seat assignment reduces contention during booking.

---

# 8. Global Distribution Systems (GDS)

Airline tickets are often sold through intermediaries.

Major GDS platforms include:

- Sabre
- Amadeus
- Travelport

Architecture:

travel agency  
↓  
GDS  
↓  
airline reservation system

Pricing logic belongs to the **airline**, not the GDS.

Airlines publish fares and inventory to GDS systems.  
Agencies therefore receive **airline-defined base fares**.

Agencies may add:

- service fees
- negotiated corporate discounts
- commissions

---

# 9. Scaling Strategy

Airline reservation systems scale using several mechanisms.

## Partitioning by flight_id

Flights are largely independent.

Systems often shard data using:

partition_key = flight_id

Example:

flight A → shard 1  
flight B → shard 2  
flight C → shard 3

This allows booking operations to run in parallel across flights.

---

## Aggressive Availability Caching

Search traffic is far higher than booking traffic.

Typical ratio:

searches : bookings ≈ 1000 : 1

Architecture:

inventory database  
↓  
availability cache  
↓  
search services

Cache entries include:

- flight
- fare class
- available seats
- price snapshot

Caches are refreshed via:

- booking events
- scheduled refresh
- reconciliation jobs

---

## Asynchronous Pricing Calculations

Pricing computation can be expensive.

Typical pipeline:

demand data  
↓  
pricing engine  
↓  
fare updates  
↓  
fare database / cache

Reservation services simply read the latest price snapshot.

---

# 10. Batch Reconciliation Processes

Because multiple systems participate in the booking ecosystem,
temporary inconsistencies can appear.

Possible causes:

- failed transactions
- network failures
- stale GDS caches
- partial outages

Reconciliation jobs periodically repair mismatches.

Example process:

for each flight:
actual_sold = count(tickets)
if actual_sold != inventory.sold:
correct inventory

---

# 11. Reconciliation in Airline Reservation Systems

Reconciliation ensures **all systems agree on inventory and ticket state**.

Systems involved:

- airline reservation system
- GDS platforms
- ticketing systems
- payment systems

Typical reconciliation checks:

ticket issued but inventory not decremented  
inventory decremented but ticket not issued  
duplicate tickets  
GDS inventory mismatch

These jobs usually run:

- nightly
- after system failures
- during financial settlement

Goal:

inventory state == tickets actually issued

---

# 12. Complete Reservation Flow (Agency → GDS → Airline)

This section describes the **full booking transaction**, focusing on the reservation stage
(not search).

## Step 1 — Passenger selects itinerary

The customer uses a **travel agency system** (OTA or corporate booking tool).

The agency communicates with the **GDS** to obtain itinerary options and pricing.

Passenger chooses:

- flight
- fare class
- passenger information

The agency submits a **reservation request to the GDS**.

---

## Step 2 — GDS forwards reservation request

The GDS forwards the request to the **airline reservation system**.

Request contains:

- flight_id
- fare_class
- passenger list
- booking channel
- fare code

At this moment no payment has occurred yet.

---

## Step 3 — Airline creates provisional PNR

The airline reservation system performs:

availability check  
fare validation  
inventory allocation

If successful:

- inventory is decremented
- a **PNR is created**
- reservation status = HOLD / RESERVED

The PNR identifier is returned to the GDS.

---

## Step 4 — GDS confirms reservation to agency

The GDS stores the PNR reference and returns confirmation to the agency.

At this stage:

reservation exists  
ticket is not issued yet

This stage is often called:

PNR creation / booking confirmation

---

## Step 5 — Payment and ticketing

Payment responsibility depends on the channel.

### Agency payment model

Passenger pays the agency.

Agency later settles payment with airline via clearing systems
(e.g. BSP or ARC).

### Airline payment model

Passenger pays airline directly.

In this case the airline system performs payment authorization with the payment gateway.

---

## Step 6 — Ticket issuance

Once payment is confirmed:

airline reservation system issues an **electronic ticket**

Ticket information includes:

ticket_number  
passenger  
flight segment  
fare

Ticket status becomes:

TICKETED

The airline updates the GDS with ticket details.

---

## Step 7 — GDS synchronization

The GDS updates its internal records:

PNR status → ticketed  
inventory counters updated

This allows all agencies using the GDS to see updated availability.

---

## Step 8 — Downstream system updates

Additional systems may be updated:

- departure control system
- loyalty program systems
- accounting systems
- revenue management systems

These updates typically occur asynchronously through event streams.

---

# Summary

Airline reservation systems differ from concert ticket systems primarily in that:

- inventory is managed as **capacity pools**
- pricing and yield optimization are core components
- bookings pass through **multi-party ecosystems (agency → GDS → airline)**
- reconciliation processes ensure consistency across distributed systems
- seat allocation is often **decoupled from booking**