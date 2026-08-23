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

4. Ticket Creation and Booking

The ticket creation process validates the booking request, checks the selected event date, and determines whether the event uses reserved seating or general admission.

For general-admission events, the requested number of tickets is atomically deducted from the available ticket count. For reserved-seating events, the selected seats are validated against the configured seat map and are atomically bound to the generated ticket IDs.

Each attendee receives a unique ticket ID. The ticket stores information such as the buyer's name, email, event date, payment status, ticket type, seat information, location, language, and ticket status.

The system also supports additional attendees in a single booking. Separate ticket IDs are generated for the main customer and each additional attendee.

If any operation fails after capacity has been reserved, the system releases the reserved capacity to prevent tickets or seats from becoming permanently unavailable.

5. Payment and Idempotency

The system supports paid and unpaid ticket flows. For online payments, a Stripe PaymentIntent ID can be associated with the booking.

To prevent duplicate ticket creation caused by repeated or replayed requests, the system checks whether a PaymentIntent has already been consumed. An atomic mark_intent_used operation acts as the race-safe protection against duplicate processing.

The payment intent is consumed before tickets are issued. If another request has already consumed the same payment intent, the newly reserved capacity is rolled back and the request is rejected.

The current implementation uses the PaymentIntent primarily for replay/idempotency protection. Full server-side Stripe webhook signature and payment-status verification is identified as a future enhancement.

6. Reserved Seat Binding

For events with reserved seating, the selected seat IDs are first validated against the event location's seat map.

The number of selected seats must match the number of attendees represented by the booking. Unknown or invalid seats are rejected.

During ticket creation, the selected seats are atomically mapped to the generated ticket IDs. If any selected seat has already been claimed by another customer, the booking fails with a seats_taken response and the transaction is rolled back.

Each stored ticket therefore contains its associated seat ID and human-readable seat label.

7. Ticket Persistence and Transaction Rollback

After capacity and seat allocation have been successfully validated, ticket records are persisted.

The implementation follows a rollback strategy for failures occurring during ticket creation. If ticket generation, seat binding, payment idempotency handling, or persistence fails after capacity has been claimed, the system releases the reserved seats or restores the general-admission availability.

This prevents capacity leakage, where seats or tickets could otherwise remain unavailable even though no valid ticket was created.

8. QR Code and Ticket Generation

Every ticket is assigned a unique ticket ID which is also encoded into a QR code.

The QR code is generated in memory using the qrcode library and embedded directly into the generated PDF. The implementation does not require a separate persistent PNG file for the QR code.

Two PDF formats are supported:

- Standard A4 ticket for public/customer use.
- Simple A5 ticket for box-office printing.

The generated ticket contains information such as the customer's name, event date, time, seat, location, ticket ID, QR code, and usage instructions.

The ticket can also be generated in English or German based on the language stored with the booking.

9. Secure Ticket PDF Access

The ticket PDF endpoint is protected using a per-ticket HMAC-based token.

Instead of exposing a ticket PDF solely through its ticket ID, the system derives an additional token from the application's secret authentication key and the ticket ID.

The token is compared using a timing-safe comparison. This prevents users from simply enumerating sequential ticket IDs and accessing other customers' PDF tickets.

The PDF endpoint also uses safe path handling to prevent path traversal attacks.

10. Email Ticket Delivery

After a ticket has been successfully persisted, the system sends the ticket to the customer's email address.

The email contains the ticket information, QR code, ticket ID, and the generated PDF as an attachment.

The system validates the email address and rejects addresses containing carriage-return or line-feed characters to reduce the risk of SMTP header injection.

Importantly, email delivery is treated as a post-booking operation. If the email server fails, the already-created ticket is not deleted or rolled back. The failure is logged while the booking remains valid.

11. Ticket Cancellation

The system provides both administrator cancellation and customer self-service cancellation.

Cancellation is implemented as an idempotent active-to-cancelled state transition. Once a ticket has been cancelled, another cancellation request does not release capacity or issue another refund.

For reserved-seating events, cancellation releases the exact seat associated with the ticket. For general-admission events, the available-ticket count is incremented.

The cancellation also reverses the corresponding ticket-sale statistics.

12. Automatic Refund Processing

For tickets purchased through Stripe, cancellation can trigger a full refund using the stored Stripe PaymentIntent.

The system sends the PaymentIntent to Stripe's refund endpoint and stores the resulting refund ID when the refund succeeds.

If the refund fails, the failure is recorded and reported so that the refund can be handled manually.

This process is protected by the idempotent cancellation state, preventing repeated cancellation requests from repeatedly releasing capacity or attempting to process the same ticket as a new cancellation.

13. Self-Service Cancellation Deadline

Customers can cancel their tickets through a self-service cancellation link included in the ticket email.

Self-service cancellation is restricted to a fixed deadline of 24 hours before the event starts. After this deadline, cancellation is restricted to administrative handling at the box office.

The deadline is calculated using the event date and event time. Tickets for which the event date or time cannot be properly interpreted are not offered the self-service cancellation option.

14. Ticket Audit Trail

Ticket operations are recorded through the ticket's access-attempt history.

Cancellation operations append an audit record containing information such as the operation type, cancellation status, timestamp, actor/scanner, reason, and refund ID.

This provides an internal history of important ticket operations and supports administrative tracking and troubleshooting.

15. Internationalization

The system supports English and German ticket content.

The buyer's selected language is stored with the ticket at purchase time and is subsequently used when generating the ticket PDF and email.

If the requested language is missing or unsupported, the system falls back to English.

16. Error Handling

The backend validates incoming requests before performing booking operations.

Examples of validation include:

- Missing event date.
- Invalid event date.
- Invalid ticket quantity.
- Zero or negative ticket quantity.
- Unknown seat IDs.
- Incorrect number of selected seats.
- Already-used PaymentIntent.
- Duplicate ticket ID.
- Unauthorized API requests.

The system returns appropriate HTTP status codes such as 400 for invalid input, 401 for unauthorized requests, 403 for forbidden PDF access, 409 for conflicts such as unavailable seats or reused payments, and 500 for unexpected server-side failures.

17. Overall Booking Flow

The overall booking workflow can be summarized as:

Customer selects event
        ↓
Selects ticket quantity / seats
        ↓
Seat hold created for reserved seating
        ↓
Payment completed if required
        ↓
Backend validates request
        ↓
Capacity / seats atomically claimed
        ↓
PaymentIntent idempotency checked
        ↓
Unique ticket IDs generated
        ↓
Seats bound to ticket IDs
        ↓
Ticket records persisted
        ↓
QR code and PDF generated
        ↓
Ticket emailed to customer
        ↓
Ticket available for QR validation at entry

For cancellation:

Customer/Admin requests cancellation
        ↓
Ticket existence and active status checked
        ↓
Ticket marked as cancelled
        ↓
Seat released / availability restored
        ↓
Statistics reversed
        ↓
Stripe refund processed if applicable
        ↓
Cancellation recorded in audit trail
