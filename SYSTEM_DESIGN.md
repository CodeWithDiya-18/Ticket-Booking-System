System Design Write-up
1. System Overview

The Ticket Booking System is a full-stack web application designed for booking seats for movies and events. The system provides role-based access for customers, organisers, and administrators. Customers can browse events, view a visual seat map, select seats, complete bookings, cancel bookings, and join a waitlist when an event is sold out. Organisers can create and manage events and view booking and revenue information, while administrators manage venues, seat layouts, and system-level data.

The system consists of a frontend for event and seat interaction, a backend API for business logic and authentication, and a database for storing users, events, seats, bookings, and waitlist information. The seat map maintains an individual status for every seat: Available, Held, or Booked. This follows the assignment requirement that seat status be stored per show and rendered as a visual grid.

2. Seat Hold and TTL Mechanism

When a customer selects one or more available seats, the backend places a temporary hold on those seats before checkout. A configurable Time-To-Live (TTL) is associated with every hold, with 10 minutes used as the default value.

During the hold period, the selected seats are marked as Held and cannot be selected or booked by another customer. The hold record stores the customer associated with the hold and its expiry time.

A scheduled expiry mechanism periodically checks held seats. If the TTL expires before the customer completes the booking, the system automatically changes the seats from Held back to Available. The frontend can then refresh the seat map and display the newly available seats. This prevents abandoned checkouts from permanently blocking inventory.

3. Concurrency Protection

Concurrency protection is required to prevent two customers from successfully holding or booking the same seat at the same time.

The backend treats seat-hold and booking operations as atomic database operations. Before a seat is held, its current status is checked to ensure that it is still available. The status is then changed within the same protected operation.

Similarly, during booking confirmation, the backend verifies that the selected seats are still held by the requesting customer and have not expired. If another customer has already changed the seat state, the operation fails rather than creating a duplicate booking.

Therefore, even when two customers attempt to select the same seat simultaneously, only one request can successfully acquire the seat. This satisfies the requirement that simultaneous attempts for the same seat must not both succeed.

4. Waitlist Auto-Assignment

When all seats in a particular category, such as Premium, are unavailable, customers can join a waitlist for that specific category.

The waitlist follows a first-in-first-out (FIFO) approach. Each entry stores the event, customer, seat category, queue status, and creation time.

When a confirmed booking is cancelled and a seat becomes available, the system checks the waitlist for that event and category. The first eligible customer in the queue receives an offer for the available seat.

This avoids wasting seats released through last-minute cancellations and provides customers with an opportunity to obtain tickets even after an event initially becomes sold out.

5. Time-Limited Waitlist Offer

The waitlist offer is not permanent. When a customer receives an available seat, the system creates a temporary offer with an expiry time.

For example, the customer may receive an email containing a booking link and a 10-minute time limit. During this period, the customer can complete the booking and obtain the ticket.

If the customer does not complete the booking before the offer expires, the offer is marked as expired and the system automatically selects the next customer in the waitlist. This ensures that an available seat is continuously reallocated instead of remaining unused.
