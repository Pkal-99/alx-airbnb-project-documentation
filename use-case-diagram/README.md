## Air BnB System Use Case Diagram ##
This diagram will illustrate the interactions between different types of users (Actors) and the system for crucial functionalities.

* I. Actors: These are the entities that interact with the system.

    - Guest (Primary Actor for booking, searching, reviewing)

    - Host (Primary Actor for listing, managing properties)

    - Admins (Represents automated processes or external integrations, like Payment Gateway)

* II. Use Cases (Key System Functionalities): These are the specific functions the system performs in response to an actor's request. We'll group them by logical areas.

* Group 1: User & Authentication

    Register Account

    Log In

    Log Out

    Manage Profile

    Reset Password

* Group 2: Property Management (Host Specific)

    Create Listing

    Manage Listing (Includes update, view, delete)

    View Host Bookings

    Manage Listing Availability & Pricing

* Group 3: Booking & Search (Guest Specific)

    Search Listings

    View Listing Details

    Check Property Availability

    Book Property

    View Guest Bookings

    Cancel Booking

    Submit Review

* Group 4: Payments

    Process Payment (Included by "Book Property")

    Process Refund (Included by "Cancel Booking")

    Manage Payouts (Host Specific, may be initiated by the System/Host)

!Air Bnb Use case Diagram (https://github.com/Pkal-99/alx-airbnb-project-documentation/blob/main/use-case-diagram/AIR_BnB_Use_Case_Diagram.png)
