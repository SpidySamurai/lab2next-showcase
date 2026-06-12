# Features

What the platform does today, in production. Roadmap items are marked as such; nothing here is aspirational.

## Live in production

| Area | What it does |
| --- | --- |
| **Self-serve signup** | 14-day trial, no credit card, email verification. The account does not exist until the email is verified. |
| **Orders** | Capture with catalog autocomplete, order history, states (received / in process / completed) |
| **Patient records** | Digital clinical history, demographics, previous orders |
| **Exam catalog** | 155+ preloaded exams classified per NOM-007, lab-editable with a visual builder for formulas and reference ranges |
| **Exam packages** | Grouped exams with configurable prorated pricing |
| **Samples** | Registration and traceability per order |
| **Results portal (QR)** | Unique signed token per order, web access without an app, secure public view |
| **WhatsApp sharing** | Staff sends the portal link from the order in one click |
| **PDF reports** | Generated from the order, classic or advanced template depending on plan |
| **Appointments** | Calendar with per-branch capacity control |
| **Referring physicians** | Physician catalog linked to orders |
| **Multi-branch** | Branch count gated by plan |
| **Multi-user with roles** | Per-branch roles (reception, technician, admin) with granular claims |
| **Operational dashboard** | Daily KPIs: orders, pending work, operation status |
| **Analytics** | Revenue, collections and turnaround times per branch, exportable to Excel |
| **Billing** | Stripe subscriptions, quota enforcement, self-serve upgrade and downgrade |
| **Email invitations** | Token-based user invites to the laboratory |
| **Full auth** | Login, registration, password recovery, email confirmation |

## In the app

| Operational dashboard | Exam catalog |
| --- | --- |
| ![Dashboard](../.github/readme/dashboard.png) | ![Catalog](../.github/readme/catalog.png) |

| Order creation wizard | Result capture, automatic high flag |
| --- | --- |
| ![Order wizard](../.github/readme/order-wizard.png) | ![Result capture](../.github/readme/results-capture.png) |

## Self-serve onboarding

A lab registers, gets the preloaded catalog, configures branches and roles through guided onboarding, and captures its first order in under an hour. Watch the real flow: [registration tutorial, 94s](../.github/readme/registration-tutorial.mp4).

| Landing | Signup to first order in under an hour |
| --- | --- |
| ![Landing](../.github/readme/landing-hero.png) | ![Onboarding](../.github/readme/landing-onboarding.png) |

| Product modules | The whole system on mobile |
| --- | --- |
| ![Modules](../.github/readme/landing-modules.png) | ![Mobile](../.github/readme/mobile.png) |

## Roadmap

| Feature | Status |
| --- | --- |
| CFDI 4.0 patient invoicing | Planned |
| Automatic appointment reminders | Planned |
| Automatic WhatsApp delivery | Planned |
| Sales / cash register cut | In progress |
| Persistent quotes | In progress |
| Analyzer interfacing (HL7 / ASTM) | Planned |
| Public API | Planned |
| White label | Planned |
