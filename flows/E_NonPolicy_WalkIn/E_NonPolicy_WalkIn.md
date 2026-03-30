
# Flow [E]: [NonPolicy_WalkIn]
## Who this person is
This person has walked into the clinic and has no corporate or insurance policy linked to their Visit account. They fall into one of two sub-types:

- **Registered retail user** — they have an existing Visit account under their own mobile number, but with no policy/benefit attached. They pay entirely out-of-pocket.
- **Unregistered user** — they have never signed up on Visit and have no record in TatvaPractice. They must be registered before they can book any service.

In both cases, available services are Doctor Consultation and Lab Tests only. AHC packages are not available to non-policy users.

## How they arrived at the kiosk (trigger)

They walk in to the clinic. They have no prior booking, no policy, and may or may not have the Visit app. The kiosk is their entry point into the system.

## Screens (in order)

Standard Instructions across the design:
Use the Visit branding assets - show logo on top left. Use colours and texts of Visit components for all Icons, buttons, chevrons, and other components.
Visit logo is present here https://getvisitapp.com/payload/_next/static/media/visit-logo.03c9c3d8.svg



### S1: [Standard Welcome Screen]
- What the patient/user sees:
  a. Do you have a pre-booked appointment
  b. Are you walking in for an appointment.
- What action they take:
  a. Click on Walking in for an appointment.

### S2: [User Identification — Mobile Number Entry]
- What the user sees:
    Dialog box with the title — enter your mobile number here.
    After the patient enters the mobile number, shows the next screen to enter OTP.
- What system does in background:
    Calls the required service to trigger an OTP to this mobile number.
    After the OTP is verified, calls an API to look up the patient in Visit. Lets call this GetPatientProfile, owned by Visit.
    Two outcomes:
    a. Number found with an existing Visit account → load profile and proceed to S3 (registered user path).
    b. Number not found → proceed to S2a (registration path for new user).

#### S2a: [Registration — Capture Patient Details]
This screen is shown only for unregistered users whose mobile number was not found in Visit.

- What the user sees:
    A registration form with the following fields:
    a. Full Name (required)
    b. Date of Birth (required)
    c. Gender (required)
    d. Email address (optional)
    Mobile number is pre-filled from S2 and cannot be edited.
    A CTA to submit and create account.
- What system does in background:
    No background action yet — waits for user to fill and submit the form.
- Action user takes:
    Fills in details and taps "Create Account."

#### S2b: [Registration — Account and Record Creation]
- What the user sees:
    A brief loading/confirmation screen: "Setting up your account, please wait..." followed by a success message: "Your account has been created. Welcome to Visit Clinic, [Name]."
- What system does in background:
    Calls a Visit API to create a new retail patient account with the submitted details. Lets assume this is called CreatePatientAccount, owned by Visit.
    Simultaneously triggers a TatvaPractice API to create a new patient record in the EMR and generate a UHID for this patient. This UHID is the unique identifier that will be used across all clinic visits. Lets assume this is called CreatePatientRecord, owned by TatvaPractice (Siddharth).
    ⚠️ DECISION: Minimum data required for walk-in registration. Is DOB + gender mandatory upfront or can it be captured later by the nurse/receptionist?
- Action user takes:
    Proceeds to S3 after the account is confirmed.

### S3: [Select Service to Book]
- What the user sees:
    a. Doctor Appointment
    b. Lab Tests
    Note: AHC packages are not shown to non-policy users.
- What the system has done in the background:
    For registered users — loaded the patient profile from GetPatientProfile.
    For newly registered users — profile is freshly created from S2a/S2b.
    Hits a Visit API, lets assume FetchServices, against this patient. For non-policy users this returns only Doctor and Lab services.
- Action that the user does:
    Clicks on either Doctor Appointment or Lab Tests.


### S4a: [Booking Doctor Appointment]
- What the user sees:
    A list of doctors within the clinic who are available for booking.
    On clicking any one doctor, shows the available time slots for the doctor.
- What the system does in the background:
    Calls an API lets assume is called /FetchDoctorListing. Am not sure who will own this api — TatvaCare or Visit. This should return the doctor names and profiles.
- Action that user takes:
    Picks any doctor.

### S4b: [Doctor Slot Selection]
- What the user sees:
    Against the selected doctor, sees the next available appointment slots.
- What the system has done:
    Against selected doctor, has fetched the available slots. Assume an API called /FetchDoctorSlots is called. This is owned by Visit — lets assume.
- Action user does:
    Picks a slot and continues for booking.

### S4c: [Payment Confirmation]
- What the user sees:
    Final screen confirming the doctor appointment to be booked. Shows the full consultation fee — there is no wallet or policy benefit to apply for this user, so the full amount is payable out-of-pocket. Shows the doctor name, slot, and total amount to pay.
- What the system is doing in the background:
    Fetches the consultation fee for the selected doctor. No benefit eligibility check is needed since this user has no policy.
- Action user takes:
    Clicks on CTA to pay now and proceed.

### S4d: [Payment and Checkout]
- What the user sees:
    QR code to scan and make the payment.
- What the system has done in the background:
    Hit the payment service and generated QR.
- Action that user takes:
    Makes payment.

### S4e: [Appointment Confirmation and Check-in]
- What user sees:
    Confirmation screen with token number and wait time. Also shows a secondary CTA — "Book Another Service" — which allows the user to book an additional appointment or lab test for themselves.
- What system does:
    Once user pays, checks in the user, blocks the appointment, creates order in the respective tables, stores the financial transaction, and generates a token. Triggers WhatsApp confirmation and invoice for the user. If the user taps "Book Another Service," the session state is preserved (user remains identified) and the flow restarts from S3 (Select Service).


### S5a: [Lab Test Catalog]
- What the user sees:
    A catalog of lab tests available at the clinic, browsable by category (e.g., Blood Tests, Diabetes, Thyroid, Liver Function, Cardiac). Each test shows the name, a brief description, and price, along with an "Add" button. A floating cart icon at the bottom of the screen shows the count of tests added and the running total. User can also search for a specific test by name.
- What the system does in the background:
    Calls an API, lets assume is called /FetchLabCatalog, owned by Visit. Returns the full list of tests the clinic offers, including pricing and category grouping.
- Action that user takes:
    Taps "Add" on one or more tests to add them to cart. Once done, taps the cart icon or a "Review Cart" CTA to proceed.

### S5b: [Cart Review and Preparation Instructions]
- What the user sees:
    A list of all tests added to the cart with individual prices and a combined total. Below each test that has preparation requirements, shows the prep instruction (e.g., "12-hour fasting required for FBS and Lipid Profile"). User can remove any test from the cart here. CTA to confirm and proceed to payment.
- What the system has done in the background:
    Assembled the cart from user selections. Fetched preparation instructions for each selected test from the /FetchLabCatalog response. Computed total price.
- Action user takes:
    Reviews selection, removes any unwanted tests, then taps "Confirm and Proceed."

### S5c: [Payment Confirmation]
- What the user sees:
    Final screen confirming the selected tests and the full amount to pay. There is no wallet or policy benefit to apply for this user — the full price of all selected tests is payable out-of-pocket.
- What the system is doing in the background:
    No benefit eligibility check needed. Computes total price from the cart.
- Action user takes:
    Clicks on CTA to pay now and proceed.

### S5d: [Payment and Checkout]
- What the user sees:
    QR code to scan and make the payment.
- What the system has done in the background:
    Hit the payment service and generated QR.
- Action that user takes:
    Makes payment.

### S5e: [Lab Booking Confirmation and Check-in]
- What user sees:
    Confirmation screen showing the list of ordered tests, a token number for the lab queue, and instructions on where to head (e.g., "Please proceed to the Sample Collection area — our phlebotomist will assist you. Your results will be shared on WhatsApp within 24–48 hours."). Also informs the user that a WhatsApp message with the order summary and invoice has been sent. Also shows a secondary CTA — "Book Another Service" — which allows the user to book an additional doctor appointment or lab test for themselves.
- What system does:
    Once payment is done, creates the lab order, adds the patient to the lab queue, generates a queue token, triggers WhatsApp confirmation with order details and invoice, and stores the financial transaction. If the user taps "Book Another Service," the session state is preserved (user remains identified) and the flow restarts from S3 (Select Service).


## Error / edge case screens

### Identification
- **OTP wrong 3x** — Do not provide more attempts and ask user to speak to concierge.

### Registration (S2a / S2b)
- **Mobile number already registered in Visit** — if the number entered turns out to exist in Visit after all (race condition or user error), do not create a duplicate account. Load the existing profile and proceed to S3 as a registered user.
- **Registration form submitted with missing required fields** — show inline validation errors on S2a for each missing required field (name, DOB, gender). Do not allow form submission until all required fields are filled.
- **TatvaPractice record creation fails (CreatePatientRecord API error)** — if the UHID creation in TP fails, the user cannot be booked into the clinic system. Show an error message and direct the user to the concierge. Do not proceed to S3 with a partial registration. ⚠️ DECISION: Should the Visit account still be created even if TP record creation fails, or should both be rolled back?
- **Newly registered user with no Visit account** — WhatsApp confirmations and invoices should still be sent to the registered mobile number even if the user has not downloaded the Visit app. Confirmations must not be app-only.

### Booking
- **No doctors available today** — /FetchDoctorListing returns an empty list (all doctors unavailable or slots fully booked). S4a should show a clear empty state with a message suggesting the user come back at another time or speak to the concierge.
- **Doctor slot taken between selection and payment** — user picks a slot in S4b but by the time they reach S4d to pay, the slot has been taken. System must detect this at order creation and return the user to S4b with a message: "That slot is no longer available — please choose another time."
- **Test unavailable today** — a test in the catalog is not available today (phlebotomist absent, reagent out of stock). The test should be shown as greyed out/unavailable in S5a so the patient cannot add it to cart.
- **Empty cart** — user removes all tests from cart in S5b. The "Confirm and Proceed" CTA must be disabled and the user should be nudged back to the catalog to add tests.
- **Duplicate test order** — patient selects a test that the doctor has already ordered for them via the EMR today. System should detect the duplicate at cart review (S5b) and warn the user before allowing payment.

### Payment and Session
- **Payment QR expiry or payment failure** — QR in S4d / S5d expires or payment fails. Show a "Regenerate QR" option. The order must not be created until payment is confirmed.
- **Session timeout mid-flow** — user walks away from the kiosk with no interaction. After 60 seconds of inactivity, show a countdown warning. On timeout, reset the session to S1 and discard any in-progress cart. Note: for a newly registered user whose account was created in S2b, the account persists — only the current booking cart is discarded.
- **Multiple bookings in one session via "Book Another Service"** — each booking must have its own separate token and WhatsApp confirmation. The loop restarts at S3 since there is no beneficiary selection in this flow.
