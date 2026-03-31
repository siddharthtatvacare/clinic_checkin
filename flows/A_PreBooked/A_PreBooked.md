
# Flow [A]: [Prebooked]
## Who this person is
This person is the policy holder who has pre-booked a doctor consultation or lab visit (either AHC which could be a combination of tests - both pathology and radiology or could be only pathology) 

## How they arrived at the kiosk (trigger)

They walk in to the clinic. They have an option to not visit the Kiosk and do the self checkin on the app directly (which uses geofencing to be sure that they have indeed arrived)

## Screens (in order)

Standard Instructions across the design: 
Use the Visit branding assets - show logo on top left. Use colours and texts of Visit components for all Icons, buttons, chevrons, and other components.
Visit logo is present here https://getvisitapp.com/payload/_next/static/media/visit-logo.03c9c3d8.svg



### S1: [Standard Welcome Screen]
- What the patient/user sees:
  a. Do you have a pre-booked appointment
  b. Are you walking in for an appointment.
- What action they take:
  a. Click on Prebooked- appointment (since this is the persona of the user we are solving for in this flow)

### S2: [User Identification Screen]
- What the patient/user sees:
  a. Message saying  - Please enter your mobile number for us to fetch your details or Scan scan the QR code that was provided to you against your appointment. (You can find your QR in your Visit App)
- Action that the user takes: 
#### Clicks on Enter your mobile number

#### S2a: [Mobile number based user identification]
- What the user sees:
    a. Dialog box with the title - enter your mobile number here
    b.After patient enters the mobile number, show the next screen to enter OTP. 
- What system does in background:
     Calls the required service to trigger an OTP
     After the OTP is verified, calls and API to fetch patient details and then another APO call to fetch orders against the patient for today.#

#### S2b: [QR based user identification]
- What the user sees:
    a. An option to scan the QR with the message to the user to place their mobile screen in front of the iPad camera in a position to be able to scan the QR
- What the system does in background:
    Calls the required service which scans the QR and then calls the API to fetch the patient details then another API to fetch orders against the patient for today. 

### S3: [List of appointments/orders]
- What the user sees:
    a. A "← Back" button (top-left). Tapping it returns the patient to S2 Mobile Number entry — not to OTP. The patient restarts identification from scratch (e.g. if someone else's number was accidentally entered).
    b. Welcome {name}, where {name} is the full name of the patient fetched by calling the API.
    c. The list of the appointments/orders (Dr appointments/ Lab - show relevant icons, appointment type, doctor/test name, and time) for today against this patient with the CTA for check-in. Do not show location (cabin/floor) on this screen — location is communicated on S4 after check-in. Note that the check-in is not at order level but overall into the clinic.
- What action the user takes:
    Clicks on Check-in

### S4: [Queueing and Checkin]
- What the user sees:
 A message telling the user that you have successfully checked in and display the token number of the user. Also communicate to the user that you have received a whatsapp message with the token number and the link to track your queue. 
 Also communicate where the patient must head to avail the service. For eg if there is a consultation appointment and the doctor sits in cabin A, tell the user that head to the waiting area and finish the vitals collection and that your consultation will happen in Cabin A. 

 - What the system does in the background:
 Call the queueing service which checks-in the user into the clinic and generates the token for the user and also basis the intelligence of the queueing system tells the user where to head for availing the service.



## Error / edge case screens
- Patient not found with mobile number or wrong QR scanned - display error message in place of S3 which explains to the user to verify their mobile or QR and also suggests they speak to the concierge if they need more help

- OTP wrong 3x → Do not provide more attempts and ask user to speak to concierge
- **A polciy holder could have pre-booked an appointment for multiple beneficiaries including themselves. Then on Screen 3, must list the appointments by beneficiary names and on Screen 4, provide option to checkin other beneficiaries to loop through the screens**

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
