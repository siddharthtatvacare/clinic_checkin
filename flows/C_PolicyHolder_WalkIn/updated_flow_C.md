
# Flow [C]: [PolicyHolder_Walkin]
## Who this person is
This person is a policy holder who has NOT pre-booked a doctor consultation or lab visit and has walked into the clinic.

## How they arrived at the kiosk (trigger)

They walk in to the clinic. They have an option to not visit the Kiosk and do a spot booking or use the visit app and book a service for themselves.

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
    a. Dialog box with the title - enter your mobile number here
    b.After patient enters the mobile number, show the next screen to enter OTP. 

- What system does in background:
     Calls the required service to trigger an OTP
     After the OTP is verified, calls and API to fetch patient details and then another API call to fetch beneficiaries against the user. Lets call this API GetBeneficiaries. This API is owned by visit. This API would return all beneficiaries including beneficiary = self i.e the policy holder. 


### S3: [List of beneficiaries]
- What the user sees:
    List of beneficiaries for whom the user wants to book an appointment.
    
- What action the user takes:
    Clicks on any 1 beneficiary

### S4: [Select Service to book]
- What the user sees:
    a. Doctor appointment
    b. Lab Tests
    Add a note here saying - You can only book sponsored benefits by using your benefits management app i.e either Visit or your insurer's app.


    Clicks on either of a or b


### S5a: [Booking Doctor Appointment]
- What the user sees:
 A List of Doctors within the clinic who are available for booking. 
On clicking any one doctor, shows the available time slots for the doctor.

 - What the system does in the background:
    Calls an API lets assume is called /FetchDoctorListing. Am not sure who will own this api- TatvaCare or Visit. This should return the doctor names and profiles and the pricing against each doctor. 

- Action that user takes:
    Picks any doctor and proceeds with booking.

### S5b: [Doctor Slot Selection]
- What the user sees:
    Against the selected doctor, sees the next available appointment slots
- What the system has done:
    Against selected doctor, has fetched the available slots. Assume an API called /FetchDcotorSlots is called. This is owned by Visit - lets assume 
- Action user does:
    Picks a slot and continues for booking

### S5c: [Payment calculation and confirmation]
- What the user sees:
    Sees the final screen confirming the doctors appointment that needs to be booked. Also shows the payment to be done. When booking via kiosk, we will only collect payment. 

    Once again place the same footnote - You can only book sponsored benefits by using your benefits management app i.e either Visit or your insurer's app.
    
- What the system is doing at the background:
    Creates the cart with the order which includes the timeslot and the payment info.

- Action user takes:
    Clicks on CTA to pay now and proceed.

### S5d: [Payment and checkout]
- What the user sees:
    QR code to scan and make the payment.
- What the system has done in the background:
    Hit the payment service and generated QR. 
- Action that user takes:
    Makes payment.

### S5e: [Appointment confirmation and checkin]
- What user sees:
    Confirmation screen with token number and wait time. Also shows a secondary CTA — "Book Another Service" — which allows the user to book an additional appointment or test for the same or a different beneficiary.
- What system does:
    Once user pays, it needs to checkin the user, block the appointment, create order in the respective tables and store financial transaction information, and also generate token for this user. Also trigger whatsapp confirmation and invoice for the user. If the user taps "Book Another Service," the session state is preserved (user remains identified) and the flow restarts from S3 (Select Beneficiary).


### S6a: [Lab Test Catalog]
- What the user sees:
    A catalog of lab tests available at the clinic, browsable by category (e.g., Blood Tests, Diabetes, Thyroid, Liver Function, Cardiac). Each test shows the name, a brief description, and price, along with an "Add" button. A floating cart icon at the bottom of the screen shows the count of tests added and the running total. User can also search for a specific test by name.

    There could be individual tests and also packages. So the UI should show the following segregations: 

    - Lab Tests and Profiles
        - Recommended Packages
        - Individual tests and profiles
    - Xray
    - ECG
    - USG
    - Treadmill test
    
    There should also be a search bar to search for the lab test. Also while Xray ECG USG and Treadmill test are individual lines with a plus button, lat tests should be a chevron which is expandable. 

- What the system does in the background:
    Calls an API, lets assume is called /FetchLabCatalog, owned by Visit. Returns the full list of tests the clinic offers, including pricing and category grouping.
- Action that user takes:
    Taps "Add" on one or more tests/packages to add them to cart. Once done, taps next button to proceed to time slot booking.

### S6b: [Timeslot]
- What the user sees:
    Next available timeslots.
- What the system does in the background:
    Calls an API or a service which is listing the available time slots. Not sure who owns this between Tatva and Visit.
    This service also fetches necessary information basis rules of tests. For eg, if in the cart there is a test which has a blood test which needs fasting and there is another which is post food - for eg, fasting blood sugar and post prandial, we should ideally inform the user that he needs to make 2 lab bookings - one for the fasting and one for the other test. This information and logic should be handled by the slotting service that we call which is also fetching the time slots. 

    the Kiosk UI only needs to communicate this to the user and manage the UI/UX.


- Action that user takes:
    Selects the time slot(s) (multiple if FBS and post prandial are in cart) and clicks on next

### S6c: [Cart Review and Preparation Instructions]
- What the user sees:
    A list of all tests added to the cart with individual prices and a combined total. Below each test that has preparation requirements, shows the prep instruction (e.g., "12-hour fasting required for FBS and Lipid Profile" This information also should be coming from the scheduling service that we are calling). User can remove any test from the cart here. CTA to confirm and proceed to payment.
- What the system has done in the background:
    Assembled the cart from user selections. Fetched preparation instructions for each selected test from the /FetchLabCatalog response. Computed total price.
- Action user takes:
    Reviews selection, removes any unwanted tests, then taps "Confirm and Proceed."

- Action user takes:
    Clicks on CTA to pay now and proceed.

### S6d: [Payment and Checkout]
- What the user sees:
    QR code to scan and make the payment. 
- What the system has done in the background:
    Hit the payment service and generated QR 
- Action that user takes:
    Makes payment (or is auto-progressed if fully covered).

### S6e: [Lab Booking Confirmation and Check-in]
- What user sees:
    Confirmation screen showing the list of ordered tests, a token number for the lab queue. Also informs the user that a WhatsApp message with the order summary and invoice has been sent. Also, that the message carries a tracking link which will educate the user of where to head next and will help navigate through the clinic.  Also shows a secondary CTA — "Book Another Service" — which allows the user to book an additional appointment or test for the same or a different beneficiary.
- What system does:
    Once payment is done, creates the lab order, adds the patient to the lab queue, generates a queue token, triggers WhatsApp confirmation with order details and invoice, and stores the financial transaction. If the user taps "Book Another Service," the session state is preserved (user remains identified) and the flow restarts from S3 (Select Beneficiary).






## Error / edge case screens

### Identification
- Patient not found with mobile number — display error message in place of S3 which explains to the user to verify their mobile and also suggests they speak to the concierge if they need more help
- OTP wrong 3x → Do not provide more attempts and ask user to speak to concierge
- **Mobile number not found in Visit system (new patient)** — the number entered in S2 does not exist in Visit. Show a message explaining the patient is not registered and direct them to the concierge. New patient registration is handled in Flow E.

### Beneficiary and Policy
- **A policy holder could have booked appointments for multiple beneficiaries including themselves. Then on Screen 3, must list beneficiaries and on Screen 4, provide option to continue booking for other beneficiaries to loop through the screens**
- **Zero payable services available** — `/FetchDoctorListing` and `/FetchLabCatalog` both return empty or entirely unavailable results (clinic holiday, system outage). S4 should show a clear empty state with a message directing the user to the concierge. Must not show a blank screen with no guidance.
- **User asks about sponsored/benefit-covered services at the kiosk** — policy holder sees the "You can only book sponsored benefits via your benefits management app" note and asks for help at the kiosk. The kiosk cannot assist with benefit booking. Show a concierge-directed message clearly stating: "For benefit-covered bookings, please use your Visit app or your insurer's app, or ask our concierge for help getting started."
- **Beneficiary exists in policy but has no Visit account or UHID** — when S3 lists beneficiaries from `GetBeneficiaries`, a beneficiary may be listed by name on the policy but not yet have a UHID or active Visit account. Selecting such a beneficiary before proceeding to a booking must be blocked with a message directing the policy holder to the concierge for manual registration. Do not silently proceed and fail at order creation.

### Doctor Booking (S5)
- **No doctors available today** — `/FetchDoctorListing` returns an empty list (clinic holiday, all doctors on leave, or system misconfiguration). S5a must show an informative empty state: "No doctors are available for walk-in booking today." Do not show a blank list. Provide a CTA to return to S4 and optionally book a lab test instead.
- **All slots full for a doctor** — `/FetchDoctorSlots` returns no available slots for the selected doctor (walk-in quota exhausted or fully booked for the day). S5b must show a "No slots available today for this doctor" message and provide a back button to choose a different doctor. ⚠️ DECISION: walk-in quota per doctor — flag for resolution in `user-flows.md`.
- **Doctor slot taken between selection and payment** — user picks a slot in S5b but by the time they reach S5d to pay, another patient has booked that slot. System must detect this at order creation and return the user to S5b with a message: "That slot is no longer available — please choose another time."
- **Timeslot conflict with another booking in the same session** — user has already booked a doctor at 11:00 AM in this session and then tries to book a lab test at the same time. The system should detect the overlap at S6b (timeslot selection) and warn the user. Show a message: "You already have a booking at this time — please choose a different slot."

### Lab Tests (S6)
- **Lab not available today** — `/FetchLabCatalog` returns an empty catalog (lab closed, phlebotomist absent, all tests unavailable). S6a must show a clear "Lab services are unavailable today" message with a CTA to return to S4. Must not show a blank catalog.
- **Test unavailable today** — a specific test in the catalog is not available today (reagent out of stock, equipment under maintenance). The test should be shown as greyed out/unavailable in S6a so the patient cannot add it to cart. Must not allow selection and then fail silently at checkout.
- **Test requires a doctor's requisition** — certain tests (e.g., some radiology or specialised pathology) may require a doctor's order before they can be performed at the clinic. If such a test is in the catalog, it should either be marked clearly as "requires doctor's prescription" at the catalog level (S6a) or blocked at cart review (S6b) with a message directing the user to get a requisition from the doctor first.
- **Fasting or time-of-day constraint for same-day booking** — user is booking a fasting test (e.g., FBS, Lipid Profile) at a time when same-day collection is no longer feasible (e.g., it is 2 PM and the test requires 12-hour overnight fasting). The slotting service or catalog API should return this constraint, and the UI must communicate clearly: "This test requires fasting — the next available slot for this test is tomorrow morning." Do not silently allow the booking to proceed to payment.
- **Duplicate test order** — patient selects a test that the doctor has already ordered for them via the EMR today. System should detect the duplicate at cart review (S6b) and warn the user before allowing payment.
- **Empty cart** — user removes all tests from cart in S6b. The "Confirm and Proceed" CTA must be disabled and the user should be nudged back to the catalog to add tests.

### Payment and Session (all service types)
- **Payment QR expiry or payment failure** — the QR in S5d / S6d expires before the patient pays, or the payment gateway returns a failure. Show a "Regenerate QR" option. The order must not be created until payment is confirmed — do not create a dangling order on QR generation.
- **Session timeout mid-flow** — user walks away from the kiosk with no interaction. After 60 seconds of inactivity, show a countdown warning. On timeout, reset the session to S1 and discard any in-progress cart. Must not leave a session open indefinitely.
- **Multiple bookings in one session via "Book Another Service"** — each booking completed within the same session must have its own separate token. The confirmation screen for a second booking must not overwrite or merge with the first booking's token. WhatsApp confirmations should be sent per booking, not as a single merged message.
- **"Book Another Service" across beneficiaries** — when the policy holder loops back to S3 to book for a different beneficiary, the new session context (beneficiary, cart, selected service) must be completely fresh. The previous beneficiary's booking details must not persist into the new booking flow.

```


## Review Process

Write **Flow A first**. Once shared, Claude will review for:

1. **Alignment with existing strategy** — cross-check against `user-flows.md` and `integrations.md` for decisions already made (wallet deduction, UHID assignment, queue priority rules)
2. **Gaps** — things the existing docs flag as unresolved decisions that your flow needs to resolve
3. **Contradictions** — e.g., if beneficiary lookup is defined differently than what `integrations.md` describes
4. **Prototype feasibility** — whether a screen/step can be mocked realistically without a real backend

Flow A is the reference flow — B through E will be reviewed against it for consistency.

---

## What Already Exists in Strategy Docs

The existing strategy docs have high-level journey steps and open decisions but **no screen-by-screen kiosk logic**. Your flow files are genuinely new content. Key things already captured elsewhere:

| Topic | Where it lives |
|---|---|
| High-level patient arrival & check-in steps | `user-flows.md` §3a |
| Walk-in queue decision (self-service vs front desk) | `user-flows.md` — marked ⚠️ DECISION |
| Wallet deduction + corporate benefit check | `integrations.md` — owned by Shashvat |
| Symptom collector trigger | `integrations.md` — owned by Siddharth |
| Walk-in quota per doctor | `user-flows.md` — marked ⚠️ DECISION |
| UHID creation for new patients | `user-flows.md` — marked ⚠️ DECISION |
| Minimum data for walk-in registration | `user-flows.md` — marked ⚠️ DECISION |

Your flow files should resolve or explicitly flag each of these where they appear in that flow.
