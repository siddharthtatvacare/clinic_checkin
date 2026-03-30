
# Flow [D]: [Beneficiary_WalkIn]
## Who this person is
This person is a beneficiary of a policy holder who has NOT pre-booked a doctor consultation or lab visit and has walked into the clinic. They are NOT the policy holder themselves — they are listed under the policy holder's policy. They may or may not have their own Visit account.

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
    Two options to identify themselves:
    a. "Enter your own mobile number" — if they have a Visit account registered under their own number
    b. "Enter your policy holder's mobile number" — if they do not have their own Visit account or prefer to look up via the policy holder

#### S2a: [Identification via own mobile number]
- What the user sees:
    Dialog box with the title — enter your mobile number here.
    After patient enters the mobile number, shows the next screen to enter OTP.
- What system does in background:
    Calls the required service to trigger an OTP to this mobile number.
    After the OTP is verified, calls an API to fetch patient details and checks whether this number is registered as a beneficiary under a policy. Lets call this GetBeneficiaryDetails, owned by Visit. This returns the beneficiary's profile and the linked policy holder's details.
    If the number belongs to a policy holder (not a beneficiary), redirect the user to Flow C.

#### S2b: [Identification via policy holder's mobile number]
- What the user sees:
    Dialog box with the title — enter your policy holder's mobile number.
    After the number is entered, shows the next screen to enter OTP.
    Note: the OTP is sent to the policy holder's phone. The beneficiary must coordinate with the policy holder to receive it.
- What system does in background:
    Calls the required service to trigger an OTP to the policy holder's mobile number.
    After the OTP is verified, calls an API to fetch all beneficiaries under this policy. Lets call this GetBeneficiaries, owned by Visit. This returns all beneficiaries listed under the policy.

### S3: [Beneficiary Confirmation]

#### S3a: [After S2a — own mobile number]
- What the user sees:
    Welcome [Beneficiary Name]. A confirmation message showing: "You are a beneficiary under [Policy Holder Name]'s policy." A CTA to confirm and proceed to booking.
- What system has done in background:
    Loaded the beneficiary's profile and linked policy holder name from the GetBeneficiaryDetails response.
- Action user takes:
    Confirms their identity and proceeds to S4.

#### S3b: [After S2b — policy holder's mobile number]
- What the user sees:
    A list of all beneficiaries under the policy holder's policy. The user is asked to tap their own name to confirm who they are.
- What system has done in background:
    Loaded all beneficiaries from the GetBeneficiaries response.
- Action user takes:
    Taps their own name from the list to confirm their identity and proceeds to S4.

### S4: [Select Service to book]
- What the user sees:
    a. Doctor appointment
    b. Lab Tests
    c. Available AHCs (if any)
- What the system has done in the background:
    Against the identified beneficiary, hits a Visit API, lets assume is called FetchServices. Basis this, the doctor appointment, LabTests and Available AHCs if any are displayed.
- Action that the user does:
    Clicks on either of a, b, or c.


### S5a: [Booking Doctor Appointment]
- What the user sees:
    A List of Doctors within the clinic who are available for booking.
    On clicking any one doctor, shows the available time slots for the doctor.
- What the system does in the background:
    Calls an API lets assume is called /FetchDoctorListing. Am not sure who will own this api — TatvaCare or Visit. This should return the doctor names and profiles.
- Action that user takes:
    Picks any doctor.

### S5b: [Doctor Slot Selection]
- What the user sees:
    Against the selected doctor, sees the next available appointment slots.
- What the system has done:
    Against selected doctor, has fetched the available slots. Assume an API called /FetchDoctorSlots is called. This is owned by Visit — lets assume.
- Action user does:
    Picks a slot and continues for booking.

### S5c: [Payment calculation and confirmation]
- What the user sees:
    Sees the final screen confirming the doctors appointment that needs to be booked. Also shows the payment to be done. Some or all of this service could be sponsored.
- What the system is doing at the background:
    Fetching the benefits (assume this is called /FetchBenefitEligibility — this is owned by Visit) to identify whether or not the appointment can be fulfilled through wallet or is sponsored and thereby calculates any payable amount that user has to pay.
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
    Confirmation screen with token number and wait time. Also shows a secondary CTA — "Book Another Service" — which allows the user to book an additional appointment or test for themselves.
- What system does:
    Once user pays, it needs to checkin the user, block the appointment, create order in the respective tables and store financial transaction information, and also generate token for this user. Also trigger whatsapp confirmation and invoice for the user. If the user taps "Book Another Service," the session state is preserved (user remains identified) and the flow restarts from S4 (Select Service) — since there is only one beneficiary in this flow, there is no need to loop back to a beneficiary selection screen.


### S6a: [Lab Test Catalog]
- What the user sees:
    A catalog of lab tests available at the clinic, browsable by category (e.g., Blood Tests, Diabetes, Thyroid, Liver Function, Cardiac). Each test shows the name, a brief description, and price, along with an "Add" button. A floating cart icon at the bottom of the screen shows the count of tests added and the running total. User can also search for a specific test by name.
- What the system does in the background:
    Calls an API, lets assume is called /FetchLabCatalog, owned by Visit. Returns the full list of tests the clinic offers, including pricing and category grouping.
- Action that user takes:
    Taps "Add" on one or more tests to add them to cart. Once done, taps the cart icon or a "Review Cart" CTA to proceed.

### S6b: [Cart Review and Preparation Instructions]
- What the user sees:
    A list of all tests added to the cart with individual prices and a combined total. Below each test that has preparation requirements, shows the prep instruction (e.g., "12-hour fasting required for FBS and Lipid Profile"). User can remove any test from the cart here. CTA to confirm and proceed to payment.
- What the system has done in the background:
    Assembled the cart from user selections. Fetched preparation instructions for each selected test from the /FetchLabCatalog response. Computed total price.
- Action user takes:
    Reviews selection, removes any unwanted tests, then taps "Confirm and Proceed."

### S6c: [Payment Calculation and Benefit Application]
- What the user sees:
    Final screen confirming the selected tests and the payment to be done. Some or all of the tests could be covered under the beneficiary's policy — shows breakdown of: original amount, wallet/policy-covered amount, and final payable amount.
- What the system is doing in the background:
    Calls /FetchBenefitEligibility (owned by Visit) to check whether the selected tests are covered for this beneficiary and calculates the final payable amount.
- Action user takes:
    Clicks on CTA to pay now and proceed.

### S6d: [Payment and Checkout]
- What the user sees:
    QR code to scan and make the payment. If the tests are fully covered by the policy/wallet, this screen shows a zero-payment confirmation and proceeds automatically.
- What the system has done in the background:
    Hit the payment service and generated QR (or auto-proceeded if no payment is due).
- Action that user takes:
    Makes payment (or is auto-progressed if fully covered).

### S6e: [Lab Booking Confirmation and Check-in]
- What user sees:
    Confirmation screen showing the list of ordered tests, a token number for the lab queue, and instructions on where to head (e.g., "Please proceed to the Sample Collection area — our phlebotomist will assist you. Your results will be shared on WhatsApp within 24–48 hours."). Also informs the user that a WhatsApp message with the order summary and invoice has been sent. Also shows a secondary CTA — "Book Another Service" — which allows the user to book an additional appointment or test for themselves.
- What system does:
    Once payment is done, creates the lab order, adds the patient to the lab queue, generates a queue token, triggers WhatsApp confirmation with order details and invoice, and stores the financial transaction. If the user taps "Book Another Service," the session state is preserved (user remains identified) and the flow restarts from S4 (Select Service).


### S7: [AHC Package Booking]
Note: S7 handles the AHC (Annual Health Check) path, selected from S4 when the user taps "Available AHCs." AHC packages are pre-defined and typically include a combination of pathology tests, and may include a doctor consultation. After selecting a package, the user can also add individual tests on top of the package before proceeding to payment.

#### S7a: [AHC Package Listing]
- What the user sees:
    A list of available AHC packages for the identified beneficiary. Each package card shows: package name (e.g., "Basic Health Checkup", "Comprehensive Health Package"), a condensed list of what is included (e.g., 15 tests + doctor consultation), estimated visit duration, and package price. Some packages may be available at no cost or at a discounted rate based on the beneficiary's policy.
- What the system does in the background:
    Calls an API, lets assume /FetchAHCPackages — owned by Visit. Returns AHC packages available for this beneficiary, with pricing, inclusions, and any policy-linked eligibility.
- Action user takes:
    Selects a package to view full details.

#### S7b: [AHC Package Details and Add-ons]
- What the user sees:
    Full breakdown of the selected package — complete list of included tests (pathology, radiology if applicable), whether a doctor consultation is included, estimated visit duration (e.g., "This typically takes 2–3 hours"), and any preparation instructions (fasting requirements, etc.). Below the package details, shows a section "Add more tests" which surfaces individual tests from the lab catalog (similar to S6a) that the user can add on top of the AHC package. A cart summary at the bottom shows the AHC package + any add-on tests selected, with a running total.
- What the system has done in the background:
    Loaded the full package detail from /FetchAHCPackages. Also called /FetchLabCatalog to surface add-on tests.
- Action user takes:
    Optionally adds individual tests, then taps "Review Cart" to proceed.

#### S7c: [Cart Review — AHC Package + Add-ons]
- What the user sees:
    A consolidated cart showing: the AHC package (with its included tests listed) and any additional individual tests added. Shows individual prices and a combined total. Prep instructions for any tests that require preparation (both from the package and add-ons) are shown here. User can remove add-on tests (but not tests within the AHC package). CTA to confirm and proceed to payment.
- What the system has done in the background:
    Assembled cart with the AHC package and any add-ons. Fetched prep instructions for all items. Computed total price.
- Action user takes:
    Reviews and confirms cart, taps "Confirm and Proceed."

#### S7d: [AHC Payment Confirmation]
- What the user sees:
    Final confirmation screen showing the AHC package + any add-ons and payment breakdown (original price, policy-covered amount, final payable amount). The AHC package cost may be fully or partially covered under the beneficiary's policy; add-on tests are calculated separately for eligibility.
- What the system is doing in the background:
    Calls /FetchBenefitEligibility to check AHC package coverage and any add-on test coverage for this beneficiary and computes the total payable amount.
- Action user takes:
    Clicks on CTA to pay now and proceed.

#### S7e: [AHC Payment and Checkout]
- What the user sees:
    QR code to scan and make the payment. If fully covered by the policy/wallet, shows a zero-payment confirmation and proceeds automatically.
- What the system has done:
    Hit the payment service and generated QR (or auto-proceeded if no payment is due).
- Action user takes:
    Makes payment (or is auto-progressed if fully covered).

#### S7f: [AHC Booking Confirmation and Visit Guidance]
- What user sees:
    Confirmation screen with the AHC package (and any add-ons) booked, a visit token, and step-by-step guidance on how the visit will proceed (e.g., "Step 1 — Head to Sample Collection for your blood tests. Step 2 — Proceed to Radiology for your scan. Step 3 — Your doctor consultation is in Cabin [X]."). Also informs the user that a WhatsApp message with the full AHC schedule, prep instructions, and receipt has been sent. Also shows a secondary CTA — "Book Another Service" — which allows the user to book an additional appointment or test for themselves.
- What system does:
    Creates the AHC order (including add-ons), enrolls the patient into the relevant queues (lab queue + doctor queue if consultation is included), generates a visit token, triggers WhatsApp confirmation with the schedule and invoice, and stores the financial transaction. If the user taps "Book Another Service," the session state is preserved (user remains identified) and the flow restarts from S4 (Select Service).


## Error / edge case screens

### Identification
- **S2a — mobile number not found in Visit system** — the beneficiary's own number entered in S2a does not exist in Visit. Direct the user to try their policy holder's mobile number (S2b) or speak to the concierge. New patient registration is handled in Flow E.
- **S2a — mobile number belongs to a policy holder, not a beneficiary** — if GetBeneficiaryDetails returns a policy holder profile, redirect the user to Flow C (Policy Holder Walk-in) with a message: "It looks like you're the policy holder — let's take you to the right flow."
- **S2b — policy holder's mobile number not found** — the number entered in S2b does not exist in Visit. Show an error message asking the user to verify the number or speak to the concierge.
- **S2b — OTP wrong 3x** — Do not provide more attempts and ask user to speak to concierge. Note: since the OTP goes to the policy holder's phone, the beneficiary should call the policy holder if they have not received the OTP.
- **S3b — beneficiary does not see their name in the list** — user cannot find their name in the list shown in S3b. Show a "My name is not here" option which directs the user to speak to the concierge.

### Policy and Eligibility
- **Beneficiary's own number found but not linked to an active policy** — GetBeneficiaryDetails returns a profile but with no active policy. Show a message explaining no active policy was found for this account and direct the user to the concierge.
- **Zero services available for this beneficiary** — FetchServices returns nothing (benefit limits exhausted or policy inactive). S4 should show a clear empty state with an explanation rather than a blank list, and suggest the user speak to the concierge.
- **Benefit exhausted between selection and payment** — by the time the user reaches the payment screen, a concurrent booking has reduced or exhausted the wallet/benefit. The system must re-call /FetchBenefitEligibility at the point of payment and recalculate the payable amount. Must not silently charge the old calculated amount.

### Lab Tests (S6)
- **Duplicate test order** — patient selects a test that the doctor has already ordered for them via the EMR today. System should detect the duplicate at cart review (S6b) and warn the user before allowing payment.
- **Test unavailable today** — a test in the catalog is not available today (phlebotomist absent, reagent out of stock). The test should be shown as greyed out/unavailable in S6a so the patient cannot add it to cart.
- **Empty cart** — user removes all tests from cart in S6b. The "Confirm and Proceed" CTA must be disabled and the user should be nudged back to the catalog to add tests.

### AHC (S7)
- **No AHC packages available for this beneficiary** — /FetchAHCPackages returns empty. S7a must show an informative empty state with a suggestion to book individual lab tests instead or speak to the concierge.
- **AHC already completed or already booked this year** — system should flag this on S7a before the user proceeds, rather than letting them go through the full flow and hit a failure at payment.
- **Add-on test duplicates a test already in the AHC package** — in S7b, user tries to add an individual test that is already included in the selected AHC package. System should block the addition and show a message: "This test is already included in your selected package."
- **Radiology slot unavailable** — selected AHC package includes radiology but no slots are available today. System should communicate this at S7a or S7b — either block the package or show a clear message that the radiology component will need to be separately scheduled, before the user pays.

### Payment and Session (all service types)
- **Doctor slot taken between selection and payment** — user picks a slot in S5b but by the time they reach S5d to pay, the slot has been taken. System must detect this at order creation and return the user to S5b with a message: "That slot is no longer available — please choose another time."
- **Payment QR expiry or payment failure** — QR in S5d / S6d / S7e expires or payment fails. Show a "Regenerate QR" option. The order must not be created until payment is confirmed.
- **Session timeout mid-flow** — user walks away from the kiosk with no interaction. After 60 seconds of inactivity, show a countdown warning. On timeout, reset the session to S1 and discard any in-progress cart.
- **"Book Another Service" loop** — in Flow D, the "Book Another Service" CTA restarts from S4 (not S3), since the beneficiary has already been identified and there is only one beneficiary in this flow. Each booking in the same session must have its own separate token and its own WhatsApp confirmation.
