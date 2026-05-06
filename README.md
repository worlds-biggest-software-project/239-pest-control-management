# Pest Control Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source operations platform for pest control businesses, covering customer management, scheduling, chemical tracking, and compliance reporting.

Pest Control Management is a candidate open-source platform that helps pest control operators run scheduling, routing, technician dispatch, chemical application records, and regulatory reporting in one system. It targets SMB and mid-market operators who today choose between expensive pest-specific incumbents and generic field service tools that lack pest compliance features.

---

## Why Pest Control Management?

- Leading pest-specific platforms (PestPac, FieldRoutes, ServiceTitan) are powerful but priced for mid-market and enterprise, with custom contracts and steep onboarding curves that exclude smaller operators.
- SMB-friendly tools such as Jobber and GorillaDesk are affordable but offer lighter pest-specific compliance reporting, and Jobber lacks chemical tracking entirely.
- All analysed incumbents are proprietary closed-source SaaS with no open-source alternative identified in the mainstream market.
- Public EPA datasets (PPIS, PPLS) and NPMA compliance frameworks are openly documented, making an open-source pest platform feasible without licence conflicts.
- AI-native opportunities such as photo-based pest identification, weather-aware re-treatment scheduling, and chemical-usage intelligence are absent or only rudimentary in current tools.

---

## Key Features

### Scheduling and Routing

- Drag-and-drop scheduling calendar with recurring appointment support
- Route optimisation with proximity-based clustering and map visualisation
- Stop-sequence output for technician daily plans
- Skill- and certification-aware technician assignment (planned)

### Chemical and Compliance Tracking

- Chemical application logging with EPA registration number, active ingredients, dilution rates, application method, quantity, and targeted pests
- Multi-year electronic chemical record retention to meet state requirements
- Exportable compliance reports for state pesticide applicator and EPA requirements
- Trap and device tracking with barcode scanning, status, and activity logging

### Customer and Field Operations

- CRM with full customer interaction and service history
- Customer portal for self-service scheduling, invoice viewing, and payment
- Mobile technician app with offline capability, job completion, signature capture, and in-field payment collection
- Automated SMS and email appointment reminders and follow-ups

### Billing and Integrations

- Invoicing, payment processing, and QuickBooks integration
- Automated review request management (Google, Yelp)
- Public API for third-party and custom integrations

---

## AI-Native Advantage

AI capabilities targeted by this project go beyond the route-optimisation assistance offered by incumbents. Planned features include computer-vision pest species identification from technician-uploaded photos, predictive re-treatment scheduling using property history and weather data, automated generation of EPA-compliant application records from structured job data, chemical-usage intelligence flagging over-application and lower-risk alternatives, and ML-driven churn prediction from service and complaint signals. These capabilities address gaps the feature survey found absent across all mainstream pest control platforms.

---

## Tech Stack & Deployment

The project is expected to support self-hosted and cloud deployment so operators can choose between operational control and managed convenience. Integration with public EPA data services (PPIS, PPLS) is in scope, alongside QuickBooks for accounting, Stripe and similar processors for payments, and a public API following patterns established by Jobber's GraphQL API and ServiceTitan's REST API. A mobile technician app with offline-first sync is a core requirement.

---

## Market Context

The US pest control industry generates over $20 billion in annual revenue and software penetration is high, with over 65% of pest control companies expected to use specialised management software by 2026. Incumbent pricing ranges from $29–$49/month for SMB tools (Jobber, GorillaDesk) up to $200–$800/month for mid-market platforms and custom enterprise contracts (ServiceTitan, PestPac). Primary buyers include pest control business owners, operations managers, compliance officers, and franchise operators managing multi-location fleets.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
