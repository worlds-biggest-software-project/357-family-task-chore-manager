# Family Task & Chore Manager — Feature & Functionality Survey

> Candidate #357 · Researched: 2026-05-04

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Homey | Mobile (iOS/Android) | Free (≤3 users) + Premium subscription | https://www.homeyapp.net/ |
| OurHome | Mobile + Web | Free + Premium | http://ourhomeapp.com/ |
| Tody | Mobile (iOS/Android) | Free + ~$30/year subscription | https://todyapp.com/ |
| Greenlight | Mobile (iOS/Android) | $4.99–$14.98/month | https://greenlight.com/ |
| BusyKid | Mobile (iOS/Android) | $4/month (card); free app | https://busykid.com/ |
| Chorsee | Mobile (iOS/Android) | Free (≤10 chores) + $39.99/year | https://chorsee.com/ |
| FamZoo | Mobile + Web | ~$5.99/month | https://famzoo.com/ |
| Fami | Mobile (iOS/Android) | Free + Premium | https://apps.apple.com/us/app/fami-organize-family-routines/id6445904876 |
| KidKarma | Mobile (iOS/Android) | Freemium | https://kidkarma.app/ |
| Chorly | Mobile (iOS/Android) | $9/month or $49/year | https://apps.apple.com/us/app/chorly-family-chores-app/id6479267582 |
| Levelty | Mobile (iOS/Android) | Freemium | https://levelty.app/ |

---

## Feature Analysis by Solution

### Homey

**Core features**
- Assign recurring chores (daily, weekly, monthly) or one-time tasks
- Differentiate between responsibilities (unpaid) and paid jobs
- Allowance tracking with bank transfer (US) or cash payment marking
- IOU tracking and savings jars with financial goals
- Family chat for in-app communication
- Push notifications when chores are due or completed
- Cross-device sync across all family members

**Differentiating features**
- Free tier for up to 3 family accounts
- Savings jar system with named goals tied to allowance balance
- Family chat natively integrated alongside chore management

**UX patterns**
- Separate parent and child views — children see their task list and reward progress
- Notification-driven engagement loop: chore due → complete → parent notified → allowance updated
- 2018-era interface design; reviewers note it feels dated by 2025 standards

**Integration points**
- Bank transfer for allowance (US-only ACH)
- No documented third-party API or webhooks

**Known gaps**
- No offline support
- UI widely criticised as dated
- No integration with smart home or parental control platforms
- Chore photo proof not available

**Licence / IP notes**
- Proprietary commercial app; no open-source components disclosed

---

### OurHome

**Core features**
- Assign tasks to one person, multiple people, or rotating basis
- Due dates, repeating schedules, reminders, and late penalties
- Link tasks to weekly allowance, screen time, or family holidays
- Shared family calendar with monthly mobile-optimised view
- Grocery/shopping list with swipe-to-complete at the store
- iOS, Android, and web support

**Differentiating features**
- Late penalty system deducting points or allowance for missed chores
- Screen-time rewards: completing tasks unlocks device time
- Linking chores to a family holiday goal as a collective incentive

**UX patterns**
- Points-based economy across family members
- Calendar-centric view to see upcoming obligations in monthly context
- Progressive disclosure: simple task creation with optional advanced scheduling

**Integration points**
- No documented public API
- Premium tier gates rewards management, calendar events, and shopping list

**Known gaps**
- Reported bugginess in 2025 user reviews
- Premium paywall on core organisational features (rewards, calendar events)
- No financial literacy curriculum or savings features

**Licence / IP notes**
- Proprietary; freemium with premium subscription

---

### Tody

**Core features**
- Condition-based tracking: visual "indicator method" (green → yellow → red) rather than fixed calendar
- Frequency-based scheduling (e.g. "every 7 days") without rigid date-locking
- Comprehensive household chore library (dusting, linen changing, deep cleaning, pet care, maintenance)
- Assign chores to household members with per-person workload summary
- FairShare feature (2025) balancing labour distribution algorithmically
- Gamification: compete against "Dusty" animated dustball; earn points

**Differentiating features**
- Condition-based rather than calendar-based scheduling is the core innovation — chores float based on urgency, not arbitrary dates
- FairShare algorithm for equitable labour distribution across partners
- Strong engagement from neurodivergent users and children due to visual indicators and gaming mode
- Over 1 million active users as of late 2025

**UX patterns**
- Visual urgency bars replace date/time reminders — reduces decision fatigue
- Gamification is opt-in, not forced on all family members
- Tidy category organisation (rooms, zones, task types)

**Integration points**
- Cross-device sync via subscription ($30/year)
- No documented public API

**Known gaps**
- Primarily adult/household-oriented; weak children's interface and allowance features
- No financial features — purely operational chore management
- No family calendar integration

**Licence / IP notes**
- Proprietary; free tier + subscription for sync

---

### Greenlight

**Core features**
- Chore list creation with per-chore allowance amounts
- Allowance automation (weekly, biweekly, monthly)
- Each child (up to 5) receives their own Mastercard prepaid debit card
- Parental spending controls: category blocking, spend limits, real-time notifications
- Savings goals with named targets (bike, phone, trip)
- "Level Up" financial education game built into the app
- FDIC-insured accounts up to $250,000; Mastercard Zero Liability Protection

**Differentiating features**
- Real-money debit card tightly coupled to chore completion — the clearest path from chore to spending in the market
- Integrated financial education game embedded alongside real money management
- Bank-grade security and FDIC insurance for family financial peace of mind

**UX patterns**
- Parent dashboard showing all children's spending in real-time
- Child-facing app showing balance, savings goals, and chores in age-appropriate UI
- Onboarding focused on financial concepts: earn, save, spend, give

**Integration points**
- Mastercard network for in-person and online spending
- No documented third-party API for external integrations

**Known gaps**
- Chore tracking is secondary to the financial product — limited scheduling depth (no rotation, no alternating)
- Expensive for families with more than 1-2 children ($4.99–$14.98/month)
- No family calendar; no household coordination features beyond chores

**Licence / IP notes**
- Proprietary commercial fintech product

---

### BusyKid

**Core features**
- Parent-preset chore chart with age-appropriate suggestions
- Automatic allowance direct deposit every Friday (or 1st & 15th)
- Auto-allowance mode (no chores required, set-and-forget)
- Save, spend, give, and invest splits on earned allowance
- Charitable giving from a curated list of organisations within the app
- Fractional stock investment from as little as $10 per transaction
- Visa prepaid card for ages 5–17, accepted anywhere Visa is accepted
- Both parents can approve chores and money movement

**Differentiating features**
- Stock market investment directly from the app — the only major chore app offering real investing
- Charitable giving with in-app charity selection
- Dual-parent approval for both chore completion and financial transactions

**UX patterns**
- Weekly allowance cycle creates a predictable rhythm for children
- Four-bucket money model (save/spend/give/invest) teaches financial literacy through visual allocations
- Parent notification for all financial movements before they execute

**Integration points**
- Visa network via prepaid card
- No documented public API or webhooks

**Known gaps**
- Age-appropriate chore suggestions are preset — limited custom scheduling depth
- No rotation or fairness-balancing features
- No family calendar or household coordination beyond chores/money

**Licence / IP notes**
- Proprietary; $4/month card fee (billed annually)

---

### Chorsee

**Core features**
- Flexible scheduling: daily, weekly, monthly, specific days, alternating, rotating between members
- "Up for grabs" chores — available to whoever completes them first
- Colour-coded chores with custom icons, notes, categories, and subtasks
- Photo proof requirement per chore
- Reward system: Allowance, Points, or No Rewards per chore
- Home screen widget for quick task checking without opening app
- Reminders per chore

**Differentiating features**
- Alternating and "up for grabs" scheduling modes are rare in the category
- Photo proof verification per chore with configurable enforcement
- Home-screen widget for ambient task awareness
- Laser focus on chore management without feature bloat

**UX patterns**
- Simple, clean UI praised for not overcomplicating chore entry
- Widget reduces friction: complete tasks without entering the app
- Progressive complexity: basic chore → add rotation, subtasks, photo proof

**Integration points**
- No documented API or third-party integrations
- iOS and Android only (no web app)

**Known gaps**
- 10-chore limit on free tier frustrates larger families
- No family calendar
- No financial product (no real-money allowance delivery)
- iOS/Android only — no web access for parents on desktop

**Licence / IP notes**
- Proprietary; $39.99/year

---

### FamZoo

**Core features**
- Chore and odd-job tracking tied to allowance payments
- Automated allowance delivery on configurable schedules
- Prepaid Mastercard cards for each family member
- IOU accounts for children not yet ready for cards
- Parent-paid compound interest to teach savings incentives
- Loans, reimbursements, and parental loan repayment tracking
- Spend/save/give budget splits
- Charitable giving tracking

**Differentiating features**
- Parent-paid compound interest is a unique financial literacy teaching tool
- Loan tracking between parent and child — models real-world borrowing
- IOU accounts bridge families where a child is too young for a card
- Pioneer of the chore-linked allowance model; widest financial literacy feature set

**UX patterns**
- Finance-first UX: the app leads with accounts and balances rather than task lists
- Designed for ongoing family financial education over years, not quick wins
- Notification-driven approval flow for all transactions

**Integration points**
- Mastercard network
- No documented public API

**Known gaps**
- Older UI compared to modern competitors
- Chore scheduling features are basic compared to Chorsee or OurHome
- No gamification or visual engagement for children
- No family calendar or household coordination

**Licence / IP notes**
- Proprietary; ~$5.99/month

---

### Fami

**Core features**
- Chore chart with task assignment, deadlines, and completion tracking
- Allowance linking to chore completion
- AI-powered chore suggestions tailored to household
- AI event creation via voice: speak to create calendar events
- Shared family calendar
- AI meal planner with quiz-based household preference setup
- Shopping list
- AI assists parents in account setup more quickly

**Differentiating features**
- Most AI-integrated chore app currently available: AI chores, AI calendar entry, AI meal planning
- Broadest feature set in the "family super-app" category
- Voice-based event creation is unique in this market segment

**UX patterns**
- All-in-one family hub: calendar, chores, meals, shopping in a single app
- AI onboarding reduces setup friction for new families
- Child-facing view shows chores with reward progress

**Integration points**
- No documented public API
- Shared calendar presumably syncs across devices via app accounts

**Known gaps**
- "Feature bloat" risk: complex onboarding and many users using only a fraction of tools
- No real-money prepaid card integration
- AI features are basic by 2026 LLM standards — suggestions rather than deep personalisation

**Licence / IP notes**
- Proprietary; free with optional premium

---

### KidKarma

**Core features**
- Karma point system: children earn points for completing chores
- Age-appropriate task suggestions based on child's age
- Custom rewards redeemable for points
- Task completion requests sent by child for parent approval
- Positive reinforcement focused — no punishment/penalty model
- Habit tracking and streaks

**Differentiating features**
- Entirely positive-reinforcement philosophy — no penalties, only rewards
- Age-appropriate chore guide built into the platform (detailed by age group)
- Focus on habit building over short-term compliance

**UX patterns**
- Child-initiated completion requests shift agency to children
- Reward redemption creates a negotiation moment with clear rules
- Simple, game-like interface with karma theming

**Integration points**
- No documented API or financial integrations

**Known gaps**
- No financial product or real-money allowance
- No family calendar or household coordination
- Limited scheduling complexity (basic recurring chores)
- No photo proof verification

**Licence / IP notes**
- Proprietary; freemium model

---

### Chorly

**Core features**
- Household chore planning and assignment
- Children earn points/rewards for completed chores
- Photo proof upload by child to confirm chore completion
- Goal-setting for children to encourage consistent habits
- Family teamwork framing

**Differentiating features**
- Family teamwork framing rather than individual competition
- Goal-setting linked to reward milestones

**UX patterns**
- Simple parent/child dual-view design
- Photo proof is standard rather than optional
- Goal progress as visual motivation for children

**Integration points**
- No documented API or financial integrations

**Known gaps**
- No financial product
- Limited scheduling options compared to Chorsee or OurHome
- No calendar integration

**Licence / IP notes**
- Proprietary; $9/month or $49/year

---

### Levelty

**Core features**
- Gamified chore chart for children
- Award systems and progress tracking
- Competitive elements (leaderboards, challenges)
- Daily task management with game-like progression

**Differentiating features**
- Strongest gamification framing — explicitly designed as a game experience
- Community forum and resources for parents on chore management

**UX patterns**
- Game-first metaphors applied throughout chore interface
- Reward milestones as "levels" in a progression system

**Integration points**
- No documented API or financial integrations

**Known gaps**
- No financial product or real-money allowance
- No detailed public feature documentation
- No family calendar

**Licence / IP notes**
- Proprietary; freemium

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Recurring chore scheduling (daily, weekly, monthly)
- Separate parent and child views with role-based permissions
- Push notifications for due and completed chores
- Points or allowance rewards linked to chore completion
- Parent approval before reward release
- Cross-device sync in real time
- iOS and Android support

### Differentiating Features
- Real-money prepaid card tightly linked to chore earnings (Greenlight, BusyKid, FamZoo)
- Condition-based scheduling using urgency indicators rather than fixed dates (Tody)
- Alternating and "up for grabs" chore assignment (Chorsee)
- AI-generated chore suggestions and voice-based calendar entry (Fami)
- Stock market investment from earned allowance (BusyKid)
- Parent-paid compound interest as savings incentive (FamZoo)
- Home-screen widget for ambient task completion (Chorsee)
- FairShare labour-balance algorithm (Tody)
- Positive-reinforcement-only reward model (KidKarma)
- Financial education game embedded alongside real money management (Greenlight)

### Underserved Areas / Opportunities
- No major app combines deep chore scheduling with a full financial literacy curriculum — most choose one or the other
- Smart home integration (screen time unlocking, smart plug control) is absent from all reviewed apps
- Natural language chore entry ("clean the bathroom every Saturday, rotate between the kids") is not available outside basic voice input in Fami
- Household health dashboard showing overall family workload balance is not available in any reviewed app
- Cross-platform (iOS + Android + web) parity is lacking in most apps
- Offline support is largely absent despite relevance for travel or connectivity gaps
- Age-progressive onboarding that evolves the child's experience as they grow (from star stickers at age 5 to real budgeting at age 14) is not implemented in any single app
- Neurodivergent-friendly UX (visual cues, reduced text, predictable routines) is a significant gap — only Tody addresses this incidentally
- Parental workload equity (balancing chores between two parents, not just children) is addressed only partially by Tody's FairShare

### AI-Augmentation Candidates
- Chore generation and scheduling: AI parsing natural language parent input to create full recurring chore schedules
- Age-appropriate task recommendation: AI matching chores to children's developmental stage and seasonal context
- Fairness monitoring: AI detecting workload imbalance and proactively suggesting redistribution
- Completion detection: using phone camera AI to verify photo proof of chore completion (autonomous verification)
- Personalised financial literacy content: AI tutoring children on saving, budgeting, and spending concepts based on their actual allowance history
- Routine optimisation: AI suggesting the most efficient chore cadence based on household size, season, and completion history

---

## Legal & IP Summary

All reviewed solutions are proprietary commercial applications. No open-source chore or family allowance app with significant market traction was identified. Financial features (prepaid cards, bank transfers) are subject to banking regulation (Regulation E, FDIC insurance requirements, Mastercard/Visa network rules) which create significant IP and licensing moats for incumbents. The 2025 COPPA Rule amendments (effective April 22, 2026) impose material new compliance requirements on apps collecting data from children under 13, including expanded definitions of personal information, mandatory information security programmes, and new parental consent verification methods. GDPR-K (Article 8) applies in the EU for children up to age 16. No patents on specific features were identified in public searches; the main barriers are regulatory compliance and network effects rather than patented technology.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Recurring chore scheduling with daily, weekly, monthly, and rotation/alternating options
- Separate parent and child profiles with role-based permissions
- Points-based reward system with parent approval before release
- Push notifications for chore due reminders and completion alerts
- Real-time sync across iOS and Android devices
- Photo proof submission by child with parent verification

**Should-have (v1.1)**
- Natural language chore creation (AI parsing "clean the bathroom every Saturday, split between the kids")
- AI-generated age-appropriate chore suggestions based on child age and household context
- Household workload balance dashboard showing per-member contribution over time
- Web interface for parent management on desktop
- Home-screen widget for quick task checking and completion
- Flexible reward types: points, cash allowance (virtual ledger), or custom rewards

**Nice-to-have (backlog)**
- Integration with banking-as-a-service provider for real prepaid card linked to earned allowance
- Smart home integration: screen time unlocking (Apple Screen Time API, Google Family Link) when chores complete
- Financial literacy curriculum: age-staged lessons embedded alongside the allowance system
- Condition-based scheduling mode (urgency indicator rather than fixed date)
- Offline mode with background sync when connection restores
- Family shared calendar combining chores with events and appointments
