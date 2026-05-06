# Standards & API Reference

> Project: Family Task & Chore Manager · Generated: 2026-05-04

---

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- Relevant to the secure storage of children's personal data, parental account credentials, and financial ledger data. Compliance with this standard supports COPPA and GDPR-K obligations around data security.

**ISO/IEC 27701:2019 — Privacy Information Management (extension to ISO 27001)**
- URL: https://www.iso.org/standard/71670.html
- Defines requirements for a Privacy Information Management System (PIMS). Relevant for demonstrating accountability for children's data handling and cross-jurisdiction compliance (COPPA + GDPR-K).

**ISO/IEC 29101:2018 — Privacy Architecture Framework**
- URL: https://www.iso.org/standard/45124.html
- Provides a framework for integrating privacy into system design. Applicable when architecting child profiles, parental consent flows, and data minimisation for users under 13.

**ISO 8583 — Financial Transaction Message Format**
- URL: https://www.iso.org/standard/31628.html
- The underlying message standard for card transactions on Visa/Mastercard networks. Relevant if the app integrates a prepaid debit card through a banking-as-a-service provider.

---

### W3C & IETF Standards

**RFC 5545 — Internet Calendaring and Scheduling Core Object Specification (iCalendar)**
- URL: https://www.rfc-editor.org/rfc/rfc5545
- Defines the `.ics` / iCalendar data format used to represent and exchange events, to-dos (VTODO), and recurring schedules. The VTODO component is directly relevant to representing chore tasks with due dates and recurrence rules. Essential for any calendar export/import feature.

**RFC 4791 — Calendaring Extensions to WebDAV (CalDAV)**
- URL: https://datatracker.ietf.org/doc/html/rfc4791
- Defines the CalDAV protocol for accessing and managing calendars over HTTP/WebDAV, based on iCalendar format. Relevant for syncing family chore schedules with Apple Calendar, Google Calendar, or third-party calendar clients.

**RFC 5546 — iCalendar Transport-Independent Interoperability Protocol (iTIP)**
- URL: https://www.rfc-editor.org/rfc/rfc5546
- Defines methods for scheduling iCalendar components (REQUEST, REPLY, CANCEL). Useful for inviting family members to chore assignments or scheduling recurring tasks with acceptance workflows.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard for delegated authorisation. Required for "Sign in with Google / Apple" flows and for third-party integrations (Google Calendar, Apple Screen Time) that access family accounts on behalf of users.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Defines JWT for securely transmitting claims between parties. Used for session tokens in mobile app authentication, particularly for distinguishing parent and child account roles within the same family group.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the `Link` header for REST API hypermedia linking. Relevant to REST API design for paginating chore history, task lists, and allowance transaction records.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Authentication layer built on top of OAuth 2.0. Required for "Sign in with Apple" and "Sign in with Google" and for any identity federation between parent and child accounts. Supports claims needed to model family relationships in the identity layer.

**W3C Push API**
- URL: https://www.w3.org/TR/push-api/
- Web standard for push notifications in progressive web apps. Relevant if a web companion app for parents is implemented alongside the native mobile apps.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 / 3.2**
- URL: https://spec.openapis.org/oas/v3.2.0.html
- The industry standard for documenting REST APIs. Version 3.2 (released September 2025) adds structured tag nesting, streaming media type support (SSE, JSON Lines), native QUERY method support, and OAuth 2.0 Device Authorization Flow. An OpenAPI-described backend enables auto-generated SDKs for third-party integrations and AI agent tool use.

**JSON Schema (draft-07 / 2020-12)**
- URL: https://json-schema.org/
- Standard for validating and documenting JSON data structures. Used to define the schema of chore objects, family member profiles, allowance transaction records, and reward definitions in both API contracts and local storage schemas.

**iCalendar VTODO Extensions — CalConnect CC/11001:2024**
- URL: https://standards.calconnect.org/cc/cc11001-2024.html
- Task extensions to iCalendar that update and extend the VTODO component for improved status tracking, scheduling, and process-control use cases. More precise than raw RFC 5545 VTODO for representing complex chore states (assigned, in-progress, pending-approval, complete).

**Firebase Cloud Messaging (FCM) / APNs**
- FCM: https://firebase.google.com/docs/cloud-messaging
- APNs: https://developer.apple.com/documentation/usernotifications
- The de-facto push notification infrastructure for Android (FCM) and iOS (APNs). FCM proxies through APNs for iOS, providing a single API targeting both platforms. Push payload size limit is 4096 bytes over the FCM APNs interface. Essential for chore due reminders and completion notifications.

**WebSockets (RFC 6455)**
- URL: https://datatracker.ietf.org/doc/html/rfc6455
- Standard for persistent bidirectional communication channels over TCP. Relevant for real-time sync of chore completion status across family member devices without polling.

---

### Security & Authentication Standards

**COPPA Rule (16 CFR Part 312) — Children's Online Privacy Protection Rule**
- URL: https://www.ecfr.gov/current/title-16/chapter-I/subchapter-C/part-312
- FTC 2025 final rule: https://www.federalregister.gov/documents/2025/04/22/2025-05904/childrens-online-privacy-protection-rule
- Mandatory U.S. compliance for any app collecting personal data from children under 13. 2025 amendments (effective April 22, 2026) require: verifiable parental consent before data collection; formal information security programme with a designated security coordinator and annual risk assessments; expanded personal information definitions (including biometric and government-issued identifiers); prohibition on monetising children's data for targeted advertising; and stricter data retention limits. Consent verification methods now include face-match ID, knowledge-based authentication, digital signatures, and video/call-in confirmation.

**GDPR Article 8 / UK GDPR — Children's Data Protection**
- URL: https://gdpr-info.eu/art-8-gdpr/
- EU and UK requirement for parental consent for processing personal data of children under 16 (member states may lower to 13). Applies to any European family using the app. Data minimisation, purpose limitation, and the right to erasure are key obligations.

**OWASP Mobile Application Security Verification Standard (MASVS)**
- URL: https://mas.owasp.org/MASVS/
- Industry-standard security requirements for mobile apps. Particularly relevant for securing child account data, preventing unauthorised account access, and protecting the allowance ledger from tampering.

**NIST SP 800-63B — Digital Identity Guidelines: Authentication and Lifecycle Management**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- Guidance on authentication assurance levels (AAL). Relevant for classifying parent account authentication (password + MFA) versus child account authentication (PIN or biometric only).

**Apple FamilyControls / Screen Time API**
- URL: https://developer.apple.com/documentation/familycontrols
- Apple's restricted API for parental control integrations on iOS/iPadOS. Requires special entitlement approval from Apple. Enables the screen time reward integration (unlocking device time upon chore completion). Device monitoring is iOS-specific.

**Google Family Link**
- URL: https://families.google/familylink/
- Google's parental supervision platform for Android. Allows parents to manage app permissions, screen time, and device settings from a parent account. No public third-party API for deep integration — chore apps cannot directly query or control Family Link programmatically. Integration would require users to manually configure Family Link alongside the chore app.

**PCI DSS (Payment Card Industry Data Security Standard)**
- URL: https://www.pcisecuritystandards.org/
- Required if the app processes or stores payment card data directly. If a banking-as-a-service provider (Stripe Treasury, Marqeta, Unit) is used, PCI scope is largely delegated to the BaaS provider, but the app must still satisfy SAQ A or SAQ A-EP requirements for the payment flow.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://spec.modelcontextprotocol.io/
- Anthropic's open protocol for connecting AI models to data sources and tools via a standardised client-server architecture. Relevant for exposing a Family Task & Chore Manager MCP server that allows AI assistants (Claude, etc.) to query family chore state, create or complete tasks, and retrieve allowance balances on behalf of authenticated users. Enables conversational AI interfaces: "Mark Jake's bathroom chore as done" or "What chores does Emma have this week?"

---

## Similar Products — Developer Documentation & APIs

### Google Tasks API

- **Description:** Google's task management service integrated with Google Workspace. Provides CRUD operations for task lists and individual tasks (title, due date, status, parent task for subtasks). REST-based with OAuth 2.0 authentication.
- **API Documentation:** https://developers.google.com/tasks/reference/rest
- **SDKs/Libraries:** Google API Client Libraries for Python, JavaScript (Node.js), Java, Go, Ruby, .NET
- **Developer Guide:** https://developers.google.com/tasks/quickstart
- **Standards:** REST/JSON, OpenAPI-compatible, OAuth 2.0
- **Authentication:** OAuth 2.0 (Google identity)

---

### Google Calendar API

- **Description:** Full-featured calendar and event management API. Supports creating, reading, updating, and deleting events and recurring series. Supports reminders, attendees, and resource booking. Relevant as a sync target for family chore schedules.
- **API Documentation:** https://developers.google.com/calendar/api/v3/reference
- **SDKs/Libraries:** Google API Client Libraries (Python, JavaScript, Java, Go, Ruby, .NET, PHP)
- **Developer Guide:** https://developers.google.com/calendar/api/guides/overview
- **Standards:** REST/JSON, iCalendar (RFC 5545) import/export, CalDAV, OAuth 2.0
- **Authentication:** OAuth 2.0

---

### Apple EventKit / CalDAV

- **Description:** Apple's framework for accessing calendar and reminder data on iOS and macOS. Reminders app uses VTODO for task items. CalDAV access allows third-party apps to sync with iCloud Calendar and Reminders.
- **API Documentation:** https://developer.apple.com/documentation/eventkit
- **SDKs/Libraries:** EventKit (native Swift/ObjC framework); CalDAV over HTTP for server-side sync
- **Developer Guide:** https://developer.apple.com/documentation/eventkit/accessing_the_event_store
- **Standards:** CalDAV (RFC 4791), iCalendar (RFC 5545), OAuth 2.0 (for iCloud access)
- **Authentication:** Apple ID / OAuth 2.0 for iCloud CalDAV

---

### Stripe Treasury (Banking-as-a-Service)

- **Description:** Stripe's embedded finance platform for building financial accounts, money movement, and card products. Enables non-bank apps to offer FDIC-insured accounts, ACH transfers, and virtual/physical debit cards. Relevant for implementing an allowance wallet with real-money delivery.
- **API Documentation:** https://docs.stripe.com/treasury
- **SDKs/Libraries:** Stripe SDKs (Python, Node.js, Ruby, Java, Go, .NET, PHP)
- **Developer Guide:** https://docs.stripe.com/treasury/tutorials
- **Standards:** REST/JSON, OpenAPI 3.x, Regulation E (US), PCI DSS (delegated to Stripe)
- **Authentication:** Stripe API Keys + OAuth 2.0 (Connect)

---

### Marqeta Card Issuing API

- **Description:** Marqeta is a modern card issuing platform used by Greenlight, Square, and Klarna. Provides programmatic issuance of physical and virtual Mastercard/Visa prepaid cards, real-time transaction controls, and spend notifications. The most flexible BaaS option for building custom child allowance card features.
- **API Documentation:** https://www.marqeta.com/docs/developer-guides/overview
- **SDKs/Libraries:** REST API with official client libraries in Python, Java, Node.js; OpenAPI spec available
- **Developer Guide:** https://www.marqeta.com/docs/developer-guides/getting-started
- **Standards:** REST/JSON, OpenAPI, Mastercard/Visa network compliance, PCI DSS (Level 1 certified)
- **Authentication:** HTTP Basic Auth (API key + secret); OAuth 2.0 for partner integrations

---

### Apple Screen Time API (FamilyControls)

- **Description:** Apple's restricted framework for parental supervision and screen time management on iOS 16+. Enables apps to view app usage, set limits, and block or allow specific apps and categories. Requires special entitlement from Apple — not available to all developers. Relevant for screen-time reward integration.
- **API Documentation:** https://developer.apple.com/documentation/familycontrols
- **SDKs/Libraries:** FamilyControls (Swift only, iOS/iPadOS)
- **Developer Guide:** https://developer.apple.com/documentation/deviceactivity
- **Standards:** Apple proprietary; requires Apple developer entitlement approval
- **Authentication:** Apple ID family group membership; restricted entitlement

---

### Firebase Cloud Messaging (FCM)

- **Description:** Google's cross-platform push notification service for Android and iOS (via APNs bridge). Used for reliable, battery-efficient delivery of chore due reminders, completion alerts, and parent approval requests. Free at scale for most consumer apps.
- **API Documentation:** https://firebase.google.com/docs/cloud-messaging
- **SDKs/Libraries:** Firebase SDKs (iOS Swift, Android Kotlin/Java, Flutter, React Native, Unity, C++)
- **Developer Guide:** https://firebase.google.com/docs/cloud-messaging/ios/get-started
- **Standards:** HTTP/JSON (v1 API), APNs (RFC 2616 + Apple proprietary), OAuth 2.0 (service account)
- **Authentication:** Firebase service account (OAuth 2.0 JWT)

---

### Nylas Calendar & Tasks API

- **Description:** Unified calendar and email API aggregating Google Calendar, Microsoft Outlook/Exchange, and iCloud Calendar behind a single REST interface. Simplifies cross-provider calendar sync for apps that cannot maintain separate integrations per provider.
- **API Documentation:** https://developer.nylas.com/docs/api/
- **SDKs/Libraries:** Nylas SDKs (Python, Ruby, Node.js, Java, Kotlin)
- **Developer Guide:** https://developer.nylas.com/docs/the-basics/
- **Standards:** REST/JSON, iCalendar (RFC 5545), OAuth 2.0, OpenAPI-documented
- **Authentication:** OAuth 2.0 (provider-specific delegated access via Nylas hosted auth)

---

## Notes

**Banking-as-a-Service Regulatory Landscape (2025–2026):** The BaaS sector is undergoing regulatory tightening in the US following the 2024 Synapse collapse. Apps building allowance card features should prefer BaaS partners with direct bank relationships (e.g. Marqeta, Unit, Treasury Prime) over intermediary BaaS middlemen. Compliance with Regulation E (Electronic Fund Transfer Act) is required for prepaid card products.

**COPPA Compliance Deadline:** The 2025 COPPA amendments become enforceable from April 22, 2026. Any app targeting users under 13 — or mixed-audience apps that may reach under-13 users — must complete compliance before this date or risk FTC enforcement action.

**Apple FamilyControls Entitlement:** Apple restricts the FamilyControls API to selected developers. Applications must apply for the `com.apple.developer.family-controls` entitlement through the Apple developer portal. Approval is not guaranteed and the review process can take weeks, making this a deployment risk for the smart home reward integration feature.

**MCP Opportunity:** No reviewed competitor exposes an MCP server. Building a first-party MCP server for the Family Task & Chore Manager would enable AI assistant integrations (Claude, Siri Shortcuts, third-party AI apps) before competitors, creating a meaningful distribution advantage.
