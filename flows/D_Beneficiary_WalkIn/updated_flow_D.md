
# Flow [D]: [Beneficiary_WalkIn] — Updated
## Who this person is
This person is a beneficiary of a policy holder who has NOT pre-booked a doctor consultation or lab visit and has walked into the clinic. They are NOT the policy holder themselves — they are listed under the policy holder's policy. They may or may not have their own Visit account.

**Key constraint in this flow:** OPD benefits and sponsored coverages (including AHC packages) are **not bookable through the kiosk**. The kiosk only facilitates direct-pay bookings. Benefit-covered bookings must be made through the Visit app or the insurer's app. This is the same rule as Flow C (Policy Holder Walk-in) applied to the beneficiary persona.

## How they arrived at the kiosk (trigger)

They walk in to the clinic. Because they are a beneficiary (not the policy holder), they have two ways to identify themselves at the kiosk — via their own registered mobile number, or via the policy holder's mobile number.

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

### S2: [User Identification Screen]
- What the user sees:
   
   
     "Enter your mobile number" — since its a common screen an we do not know if the person walking in is the policy holder or the beneficiary.
     

#### S2a: [Identification via own mobile number]
- What the user sees:
    Dialog box with the title — enter mobile number here.
    After patient enters the mobile number, shows the next screen to enter OTP.
- What system does in background:
    Calls the required service to trigger an OTP to this mobile number.
    After the OTP is verified, calls an API to fetch patient details and checks whether this number is registered as a beneficiary under a policy. Lets call this GetBeneficiaryDetails, owned by Visit. This returns the beneficiary's profile and the linked policy holder's details.
    
    If not beneficiary is identified, proceed with @E_NonPolicy_Walkin flow. 
    If identified beneficiary, just welcome the beneficiary

     



### S4: [Select Service to book]
- What the user sees:
A welcome message with Name
    a. Doctor appointment
    b. Lab Tests

    A note below the service options: "You can only book sponsored benefits by using your benefits management app — either the Visit app or your insurer's app."

    Note: AHC packages and benefit-covered OPD services are intentionally not shown here, as they cannot be booked through the kiosk. The kiosk only facilitates direct-pay bookings.

- What the system has done in the background:
    Fetches doctor availability and lab catalog separately when the user selects a service. No benefit eligibility check is run at this stage.
- Action that the user does:
    Clicks on either a or b.


### S5a: [Booking Doctor Appointment]
- What the user sees:
    A list of doctors within the clinic who are available for walk-in booking. Each doctor card shows the doctor's name, specialty, and the consultation fee. A search or filter by specialty is available.
- What the system does in the background:
    Calls an API — /FetchDoctorListing — not sure who owns this between TatvaCare and Visit. This returns the doctor names, profiles, specialty, and full-price consultation fee (no benefit deduction applied).
- Action that user takes:
    Picks any doctor and proceeds to slot selection.

### S5b: [Doctor Slot Selection]
- What the user sees:
    Against the selected doctor, sees the next available appointment slots displayed as a time grid or list (e.g., 10:00 AM, 10:30 AM, 11:00 AM). Slots already taken are greyed out and not selectable.
- What the system has done:
    Against selected doctor, has fetched the available slots via an API — /FetchDoctorSlots, owned by Visit. Returns available time windows for today.
- Action user does:
    Picks a slot and continues for booking.

### S5c: [Payment Calculation and Confirmation]
- What the user sees:
    Sees the final screen confirming the doctor and time slot selected. Shows the full consultation fee to be paid. When booking via kiosk, only direct payment is collected — no benefits or wallet deductions are applied.

    The same footnote is displayed again: "You can only book sponsored benefits by using your benefits management app — either the Visit app or your insurer's app."

- What the system is doing in the background:
    Creates the cart with the order — selected doctor, time slot, and full-price consultation fee. No call to /FetchBenefitEligibility is made.
- Action user takes:
    Clicks on CTA to pay now and proceed.

### S5d: [Payment and Checkout]
- What the user sees:
    QR code to scan and make the payment.
- What the system has done in the background:
    Hit the payment service and generated QR.
- Action that user takes:
    Makes payment.

### S5e: [Appointment Confirmation and Check-in]
- What user sees:
    Confirmation screen with token number and estimated wait time. Also shows a secondary CTA — "Book Another Service" — which allows the user to book an additional appointment or test for themselves.
- What system does:
    Once user pays, checks in the user, blocks the appointment slot, creates the order in the respective tables, stores the financial transaction, generates a queue token for this user, and triggers WhatsApp confirmation and invoice. If the user taps "Book Another Service," the session state is preserved (user remains identified as this beneficiary) and the flow restarts from S4 (Select Service). Since only one beneficiary is in context in this flow, there is no return to a beneficiary selection screen.


### S6a: [Lab Test Catalog]
- What the user sees:
    A catalog of lab tests available at the clinic, browsable by category. Each test shows the name, a brief description, and the full price (no benefit deduction applied), along with an "Add" button. A floating cart icon at the bottom shows the count of tests added and the running total. User can also search for a specific test by name.

    The catalog is organised as follows:
    - Lab Tests and Profiles
        - Recommended Packages
        - Individual tests and profiles
    - Xray
    - ECG
    - USG
    - Treadmill test

    Lab Tests and Profiles is an expandable chevron section. Xray, ECG, USG, and Treadmill test are individual lines with a plus button.

- What the system does in the background:
    Calls /FetchLabCatalog, owned by Visit. Returns the full list of tests the clinic offers, including full pricing, category grouping, preparation instructions, and any time-of-day or fasting constraints. No benefit eligibility check is applied.
- Action that user takes:
    Taps "Add" on one or more tests/packages to add them to cart. Once done, taps the next button to proceed to time slot selection.

### S6b: [Timeslot Selection]
- What the user sees:
    Next available timeslots for the selected tests. If the cart contains tests with conflicting preparation requirements (e.g., a fasting test and a post-meal test), the system communicates that two separate lab visits will need to be scheduled and presents two separate slot pickers.
    Also flags if any test in the cart has a time-of-day constraint that cannot be met today (e.g., fasting test but it is already 2 PM — show: "This test requires fasting — the next available slot is tomorrow morning").
- What the system does in the background:
    Calls an API or scheduling service — ownership between TatvaCare and Visit to be confirmed — which returns available time slots. This service also applies preparation rules: if the cart has tests requiring conflicting conditions, it splits the booking into separate time slots and communicates this to the UI. The UI only needs to present this information and manage the UX.
- Action that user takes:
    Selects the time slot(s) (multiple if preparation conditions require separate visits) and clicks on next.

### S6c: [Cart Review and Preparation Instructions]
- What the user sees:
    A list of all tests added to the cart with individual prices and a combined total (full price, no benefit deduction). Below each test that has preparation requirements, shows the prep instruction (e.g., "12-hour fasting required for FBS and Lipid Profile"). User can remove any test from the cart here. CTA to confirm and proceed to payment.
- What the system has done in the background:
    Assembled the cart from user selections. Fetched preparation instructions from the /FetchLabCatalog response. Computed total full-price amount. No call to /FetchBenefitEligibility is made.
- Action user takes:
    Reviews selection, removes any unwanted tests, then taps "Confirm and Proceed."

### S6d: [Payment and Checkout]
- What the user sees:
    QR code to scan and make the payment.
- What the system has done in the background:
    Hit the payment service and generated QR.
- Action that user takes:
    Makes payment.

### S6e: [Lab Booking Confirmation and Check-in]
- What user sees:
    Confirmation screen showing the list of ordered tests, a token number for the lab queue, and instructions on where to head (e.g., "Please proceed to the Sample Collection area — our team will assist you. Your results will be shared on WhatsApp."). Also informs the user that a WhatsApp message with the order summary and invoice has been sent. Also shows a secondary CTA — "Book Another Service" — which allows the user to book an additional appointment or test for themselves.
- What system does:
    Once payment is done, creates the lab order, adds the patient to the lab queue, generates a queue token, triggers WhatsApp confirmation with order details, invoice, and preparation instructions, and stores the financial transaction. If the user taps "Book Another Service," the session state is preserved (user remains identified as this beneficiary) and the flow restarts from S4 (Select Service).




## Error / edge case screens

### Identification
- **S2a — mobile number not found in Visit system** — the beneficiary's own number entered in S2a does not exist in Visit. Show a message: "We couldn't find this number in our system. Try using your policy holder's mobile number instead, or speak to our concierge." Direct the user to try S2b or go to the concierge. New patient registration is handled in Flow E.
- **S2a — mobile number belongs to a policy holder, not a beneficiary** — if GetBeneficiaryDetails returns a policy holder profile, redirect the user to Flow C (Policy Holder Walk-in) with a message: "It looks like you're the policy holder — let's take you to the right flow."
- **S2a — mobile number found but not linked to an active policy** — GetBeneficiaryDetails returns a profile but with no active policy. Show a message: "No active policy was found for this account." Direct the user to the concierge.
- **S2b — policy holder's mobile number not found** — the number entered in S2b does not exist in Visit. Show an error message asking the user to verify the number or speak to the concierge.
- **S2b — OTP wrong 3x** — Do not provide more attempts and ask user to speak to concierge. Since the OTP goes to the policy holder's phone, note that the beneficiary should contact the policy holder if the OTP was not received.
- **S3b — beneficiary does not see their name in the list** — user cannot find their name in the list shown in S3b. The "My name is not here" option directs the user to the concierge for manual verification. ⚠️ DECISION: minimum data requirements for manual registration — flag for resolution in `user-flows.md`.

### Policy and Eligibility
- **Beneficiary exists in policy list but has no UHID or Visit account** — when S3b lists beneficiaries from GetBeneficiaries, a name may appear on the policy but the corresponding person may not yet have a UHID or an active Visit account. Selecting such a beneficiary must be blocked with a message: "This beneficiary has not yet been registered in the system. Please speak to our concierge." Do not silently proceed and fail at order creation.
- **User asks about sponsored/benefit-covered services at the kiosk** — the beneficiary sees the "You can only book sponsored benefits via your benefits management app" note and asks for assistance. The kiosk cannot process benefit bookings. Show a concierge-directed message: "For benefit-covered bookings, please use your Visit app or your insurer's app, or ask our concierge for help getting started."

### Doctor Booking (S5)
- **No doctors available today** — /FetchDoctorListing returns an empty list (clinic holiday, all doctors on leave, or system misconfiguration). S5a must show an informative empty state: "No doctors are available for walk-in booking today." Provide a back CTA to return to S4 and optionally book a lab test instead. Do not show a blank list.
- **All slots full for a doctor** — /FetchDoctorSlots returns no available slots for the selected doctor (walk-in quota exhausted or fully booked for the day). S5b must show a "No slots available today for this doctor" message and provide a back button to choose a different doctor. ⚠️ DECISION: walk-in quota per doctor — flag for resolution in `user-flows.md`.
- **Doctor slot taken between selection and payment** — user picks a slot in S5b but by the time they reach S5d to pay, another patient has booked that slot. System must detect this at order creation and return the user to S5b with a message: "That slot is no longer available — please choose another time."
- **Timeslot conflict with another booking in the same session** — user has already confirmed a doctor booking at 11:00 AM earlier in this session and then tries to book a lab test at the same time. The system should detect the overlap at S6b (timeslot selection) and warn the user: "You already have a booking at this time — please choose a different slot."

### Lab Tests (S6)
- **Lab not available today** — /FetchLabCatalog returns an empty catalog (lab closed, phlebotomist absent, all tests unavailable). S6a must show a clear "Lab services are unavailable today" message with a CTA to return to S4. Must not show a blank catalog.
- **Test unavailable today** — a specific test in the catalog is not available today (reagent out of stock, equipment under maintenance). The test should be shown as greyed out/unavailable in S6a so the patient cannot add it to cart. Must not allow selection and fail silently at checkout.
- **Test requires a doctor's requisition** — certain tests (e.g., some radiology or specialised pathology) may require a doctor's prescription or EMR order before they can be performed at the clinic. If such a test is present, mark it clearly in the catalog as "requires doctor's prescription" (S6a) or block it at cart review (S6c) with a message directing the user to obtain a requisition from the doctor first.
- **Fasting or time-of-day constraint for same-day booking** — user is booking a fasting test at a time when same-day collection is no longer feasible (e.g., it is 2 PM and the test requires 12-hour overnight fasting). The slotting service or catalog API should return this constraint. The kiosk must communicate clearly: "This test requires fasting — the next available slot for this test is tomorrow morning." Do not silently allow the booking to proceed to payment for a slot that cannot be honoured.
- **Duplicate test order** — patient selects a test that the doctor has already ordered for them via the EMR today. System should detect the duplicate at cart review (S6c) and warn the user before allowing payment.
- **Empty cart** — user removes all tests from cart in S6c. The "Confirm and Proceed" CTA must be disabled and the user should be nudged back to the catalog to add tests.

### Payment and Session (all service types)
- **Payment QR expiry or payment failure** — the QR in S5d / S6d expires before the patient pays, or the payment gateway returns a failure. Show a "Regenerate QR" option. The order must not be created until payment is confirmed — do not create a dangling order on QR generation.
- **Session timeout mid-flow** — user walks away from the kiosk with no interaction. After 60 seconds of inactivity, show a countdown warning. On timeout, reset the session to S1 and discard any in-progress cart. Must not leave a session open indefinitely.
- **Multiple bookings in one session via "Book Another Service"** — in Flow D, "Book Another Service" restarts from S4 (not S3), since the beneficiary has already been identified and there is only one beneficiary in this flow. Each booking completed within the same session must have its own separate token. WhatsApp confirmations should be sent per booking, not as a single merged message.
- **"Book Another Service" for a different beneficiary** — in the S2b path, the policy holder may have entered their number and selected a specific beneficiary in S3b. If a different beneficiary from the same family also needs a booking, they should initiate a new kiosk session. The "Book Another Service" CTA in Flow D is scoped to the identified beneficiary and does not offer switching to another beneficiary. The confirmation screen should note: "To book for another family member, please start a new session at the kiosk."
- **Concurrent booking by policy holder** — while the beneficiary is checking in at the kiosk, the policy holder (or the beneficiary themselves via app) may concurrently book a sponsored service through the Visit or insurer app. This does not affect the kiosk flow since kiosk payments are direct-pay only. However, if the beneficiary later presents the kiosk-booked appointment alongside a benefit-covered appointment, the clinic staff (not the kiosk) will need to reconcile.


## Review Notes

### Decisions flagged (⚠️)
- **Walk-in quota per doctor** — this flow assumes /FetchDoctorSlots returns slots respecting walk-in quotas. The quota allocation rules are unresolved in `user-flows.md` and must be confirmed with TatvaCare/Visit.
- **UHID creation for new patients** — if a beneficiary enters their mobile number and is not found in Visit, Flow E handles registration. This flow redirects to the concierge rather than launching Flow E inline. That handoff point needs confirmation.
- **Minimum data for walk-in registration** — referenced above in S3b edge case. Needs resolution in `user-flows.md`.
- **API ownership (FetchDoctorListing, FetchDoctorSlots, timeslot service)** — ownership between TatvaCare and Visit is unresolved. Prototype can mock these; ownership must be confirmed before build.
