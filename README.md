# Family Task & Chore Manager

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source household coordination platform that combines deep chore scheduling, allowance tracking, and family financial literacy in a single cohesive experience.

Family Task & Chore Manager is a purpose-built household coordination tool for families with children. It addresses a persistent friction point — chores going unnoticed, allowances tracked inconsistently, and responsibilities unevenly distributed — by giving parents transparent visibility into who has done what while giving children clear, age-appropriate feedback on their contributions.

---

## Why Family Task & Chore Manager?

- **Existing apps force a choice between depth and breadth.** Tools like Chorsee offer rich scheduling but no financial product; Greenlight and BusyKid offer real-money cards but shallow chore scheduling. No incumbent combines both well.
- **General task managers miss family-specific needs.** Professional tools lack age-appropriate UI, allowance ledgers, parental approval flows, and reward systems that suit children.
- **Incumbents have dated UX or feature bloat.** Homey is widely criticised as visually dated; Fami risks overwhelming users with overlapping features. Modern, focused design is an opportunity.
- **Pricing is high for multi-child households.** Greenlight charges $4.99–$14.98/month and Chorly $9/month or $49/year — a free, open-source alternative is a meaningful disruption.
- **No open-source player has traction.** Every reviewed solution is a proprietary commercial app, leaving the market without a community-governed option.

---

## Key Features

### Chore Scheduling and Assignment

- One-off and recurring chores with daily, weekly, monthly cadences
- Rotation and alternating assignment between family members
- "Up for grabs" chores available to whoever completes them first
- Photo proof submission by child with parent verification
- Subtasks, categories, and per-chore reminders

### Family Profiles and Roles

- Separate parent (admin/oversight) and child (task and reward) views
- Role-based permissions appropriate to each profile
- Age-progressive UI suitable for children at different developmental stages
- Dual-parent approval for chore completion and reward release

### Rewards and Allowance

- Points-based reward system with parent approval before release
- Flexible reward types: points, virtual cash ledger, or custom rewards
- Allowance ledger tracking earned, pending, and paid balances per child
- Streaks, badges, and progress visualisation to sustain engagement

### AI-Augmented Coordination

- Natural language chore creation (e.g. "clean the bathroom every Saturday, split between the kids")
- AI-generated age-appropriate chore suggestions based on child age and household context
- Household workload balance dashboard tracking per-member contribution over time
- AI-assisted onboarding to reduce setup friction for new families

### Cross-Platform Sync

- iOS and Android applications
- Web interface for parent management on desktop
- Real-time sync across all family members' devices
- Home-screen widget for ambient task awareness and quick completion

---

## AI-Native Advantage

AI in this project is not a thin wrapper on a chatbot. It is used to parse natural-language chore descriptions into structured recurring schedules, recommend age-appropriate chores based on each child's developmental stage and seasonal context, monitor workload fairness across family members, and personalise financial literacy content based on a child's actual allowance history. Photo-proof verification can be automated via vision models, reducing parent approval friction while preserving oversight.

---

## Tech Stack & Deployment

The project targets iOS and Android as primary platforms with a parallel web interface for desktop parent use. Real-time cross-device sync, push notifications with configurable quiet hours, and offline support with background reconciliation are core infrastructure requirements. Optional integrations with banking-as-a-service providers (for real prepaid card linkage) and smart home APIs (Apple Screen Time, Google Family Link) are planned for later phases. COPPA Rule (effective 2026-04-22) and GDPR-K (Article 8) compliance is foundational, with minimal child data collection and verifiable parental consent flows.

---

## Market Context

The category is moderately competitive but no single incumbent occupies the union of deep chore scheduling and full financial literacy. Pricing among incumbents ranges from free freemium tiers (Homey, OurHome, Fami) to $4.99–$14.98/month (Greenlight) and ~$5.99/month (FamZoo). Tody alone reports over 1 million active users as of late 2025, demonstrating consumer demand for chore-first tools. Primary buyers are parents of children aged 5–17, with strong adoption potential among neurodivergent-friendly households underserved by current visual and routine designs. Candidate complexity is rated 3/10 with high domain availability and medium demand.

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
