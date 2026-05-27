# Attempt

## 1) Requirements (me)

### Initial round

1. Needs a management system object
2. Management system tracks parking and unparking of vechicles.
3. Be able to generate Parking tickets when vechicles arrrive
4. Fee calculation when vehicles leave
5. Multiple floors per parking Lot

### 1A) Clarification of requirements (me):

1. Are each floor uniform in lay out, do the vechicles move through the floors and the order matter. And for the rest of the fees, and vechicle space preference functionally the same?
2. What's the fee calculation function?
3. Are all the vechicles the same and if not does the class of the vechicle change the functional behavior of the management system towards the vechicles?

### 1B) (interviewer)

1. Floors do not have to be uniform. You can assume each floor has its own collection of spots.
2. Vehicle type matters. A bike, car, and truck may need different spot types, so your design should account for compatibility.
3. Yes, generate a ticket on entry and use it during exit to calculate the fee.
4. Prefer assigning the nearest available compatible spot.
5. Keep the design extensible so adding new vehicle or fee types does not force large changes.

That is enough clarification for now. Proceed by naming the core entities and the main APIs before going deeper into class relationships.

### 2) Domain Model:

1. I'm assuming that we'll probably have a ParkingLotManagement system which has a one to many relationship with with each ParkingFloor

2. I should also have a Vehicle Abstraction which enforces the behavior that all Bikes and Cars and Trucks entities which inherits from it have the forced implementation of entry if there's a space on one of the floors, exit which calls fee implementation of the ParkingLotManagement.

3. I should have a ticket which is one to one with Vehicle Abstraction, on entry

Clarification what does it mean finding nearest available compatible spot? is this via min(floor where empty compatible spot) or within floor there's ordering too?

### 2A) (Interviewer)

Directionally correct, but your domain model is still missing one core entity: `ParkingSpot`. Without that, you cannot represent compatibility, occupancy, or "nearest available compatible spot."

A second critique: I would not put `entry` and `exit` behavior on `Vehicle`. The vehicle is mostly domain data plus type identity. The orchestration belongs to the parking lot management side, because parking and unparking require shared state lookup, spot assignment, ticket creation, and fee calculation.

For your ticket assumption, "one active ticket per parked vehicle" is a reasonable constraint for this round. Just be careful not to overstate permanent one-to-one identity across all time; it is one ticket per parking session.

For "nearest available compatible spot," assume a deterministic ordering chosen by the system. It can be floor-first then spot-order within floor, or any other consistent ordering. You do not need to optimize the exact metric yet. The important part in step 2 is that the model must support:

1. spot compatibility by vehicle type
2. spot occupancy state
3. enough location or ordering information to define "nearest"

Continue by refining your domain model to include the minimal core entities and each entity's responsibility only. Start with `ParkingLot`, `ParkingFloor`, `ParkingSpot`, `Vehicle`, and `Ticket`. Then tell me which one owns spot assignment.


### 2B)

1. In a sense I previously should have defined state space clearly right instead of jumping into to relationships, separating the order avoids confusion.

3. ParkingLot should own ParkingSpot assignment, since the edge case of a parking floor being full could cause Parkingfloor not being able to assign a spot since this is inter floor check.

### 2C) (Interviewer)

Yes, that is a better correction. Separating state and invariants from relationships usually makes the domain model much cleaner.

Your ownership call is reasonable for this round: `ParkingLot` can own spot assignment because the allocation decision spans multiple floors. That said, be precise about what `ParkingFloor` still owns. A floor should still own its local collection of spots and expose enough information for the lot-level allocator to query or reserve from it.

One critique: you are still speaking mostly in terms of ownership, but step 2 should also make the entities themselves explicit. Right now the missing gap is not the allocator decision anymore; it is the responsibility boundary of each core entity.

For this round, define the minimal responsibility of:

1. `ParkingLot`
2. `ParkingFloor`
3. `ParkingSpot`
4. `Vehicle`
5. `Ticket`

Keep each one to one sentence. Do not move into methods yet unless the responsibility is unclear without them.

### 2D)

1. ParkingLot should keep track of the fullness of the parking floors and allocation of the ParkingSpot

2. Parkingfloor should track it's internal state of parking spots, when allocated by ParkingLot

3. ParkingSpot should track which vehicle is within the ParkingSpot if any.

4. A Vehicle should contain a ticket all all the time within the parkinglot, we don't track outside vehicles hence it should always track the ticket

5. A Ticket should contain just the time of entry of the vehicle, we can use this to track the exit time. of the vehicle.

### 2E) (Interviewer)

This is moving in the right direction. You now have the right core entities and the ownership split is much clearer.

Main comments:

1. `ParkingLot` owning global allocation is fine, but "track fullness" is a bit too derived. The stronger responsibility is that it coordinates parking and unparking across floors and chooses a compatible spot.
2. `ParkingFloor` is fine directionally, but say it owns a collection of spots rather than only "internal state." That is the more concrete domain responsibility.
3. `ParkingSpot` should track both compatibility and occupancy, not only which vehicle is currently there.
4. I would not say a `Vehicle` contains a ticket "all the time." A vehicle exists outside the parking lot too. For this round, it is safer to say the vehicle carries identifying/type information, and during an active parking session it is associated with a ticket.
5. `Ticket` likely needs more than entry time. At minimum, think about whether the system can unpark correctly if the ticket does not identify the parked vehicle or spot.

The biggest remaining gap is `Ticket` and `ParkingSpot`: one is missing enough identity to support exit, and the other is missing enough state to support compatibility.

Refine only those two responsibilities once more, then you can move to step 3 and sketch the main APIs.

### 2F) 

1. ParkingLot and ParkingFloor should just be collections instead of state tracking from previous

2. ParkingSpot should track type of vehicle which can normally park there and occupancy which is either None or vehicle id

3. Vechicle contains identifying information such as vehicle id and vehicle type

4. ticket needs both vehicle id. I don't get why the ticket should track the spot, since the spot carry's it's own occupancy, however, if it's vehicle type for the fees then perhaps.

### 2G) (Interviewer)

Better. `Vehicle` and `ParkingSpot` are now much closer to what I want for this stage.

A few comments:

1. I would not reduce `ParkingLot` and `ParkingFloor` to "just collections." They are containers, but they also define responsibility boundaries. `ParkingLot` coordinates across floors; `ParkingFloor` owns a set of spots on one floor.
2. `ParkingSpot` tracking compatible vehicle type and occupancy is correct.
3. `Vehicle` carrying identity and type is correct for this round.
4. On your ticket question: the reason to include spot identity is not because the spot cannot track occupancy. It is because the ticket is the handle used later during exit. If the system receives a ticket at unpark time, that ticket should give enough information to locate the parked session efficiently and unambiguously.

So the critique is: you are now mostly correct, but your `Ticket` thinking is still slightly under-modeled. Ask yourself what minimum information the exit flow needs if the input is `ticketId`.

That is enough for step 2. Move to step 3 and sketch the main APIs for:

1. park
2. unpark
3. spot lookup or allocation
4. fee calculation entry point

## 3 Relationships:

1. Parking lot one to many Parking floor one to many parking spot. ParkingSpot potentially one to one Vehicle. Ticket and Vehicle one to one. While Ticket to ParkingSpot one to one to track ParkingSpot state resolution.

## 4 Behavior:

There should be 4 main APIs

1. Park - A simple function on the ParkingSpot which changes the occupancy through the given Vehicle item.
2. Unpark - Where the vehicle is removed from ParkingSpot, so also handled in parking spot
3. Spot look up or allocation -  Handled by the ParkingLot which iterates through Parkingfloor which has internal implementation of the nearest empty compatible ParkingSpot.
4. Fee calculation entry point - ParkingLot, perhaps can track total money earnt throughout the day, this resolution is easy if ticket has the spot id, which sets ParkingSpot back to empty. Optional to remove the ticket from Vehicle to be none. hence ParkingLot taking in Vehicle is enough

### 3A) (Interviewer)

The relationship sketch is mostly fine for this round. You have the main containment structure and the temporary session-related links in the right area.

One small correction: be careful with saying `Ticket` and `Vehicle` are strictly one-to-one without qualification. That is only true for an active parking session, not as a permanent domain relationship.

Also keep in mind that step 3 is about ownership and association shape, not yet about how state changes. The relationship model is good enough to move on.

### 4A) (Interviewer)

This is the right step to start naming APIs, but your current ownership is too low-level in a couple of places.

Main comments:

1. `park` and `unpark` should not be centered on `ParkingSpot` as the main public entry points. A spot can support occupancy changes, but the system-level workflow still belongs higher up because it involves lookup, ticket handling, and fee flow.
2. `spot lookup or allocation` being coordinated by `ParkingLot` is directionally fine.
3. For this round, keep fee handling as a separate concern from "total money earned throughout the day." That metric is secondary and not needed to define the core API.
4. Be careful with "ParkingLot taking in Vehicle is enough" for exit. Your own earlier reasoning suggests the exit flow should be keyed by the parking session handle, not by the vehicle alone.

This is enough to continue. Tighten the API ownership once more, keep the methods at system boundary level, and then move to step 5 invariants.

##