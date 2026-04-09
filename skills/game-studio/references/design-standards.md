# Game Design Standards

Reference for all design and creative hats. Consolidated from the game studio's
game-designer agent, coding standards, and design document requirements.

## GDD Format — 8 Required Sections

Every mechanic document in `design/gdd/` MUST contain these sections:

### 1. Overview
One-paragraph summary a new team member could understand.

### 2. Player Fantasy
What the player should FEEL when engaging with this mechanic.
Reference the target MDA aesthetics this mechanic primarily serves.

### 3. Detailed Rules
Precise, unambiguous rules with no hand-waving.
A programmer should be able to implement from this section alone.

### 4. Formulas
All mathematical formulas with:
- Variable definitions and types
- Input ranges (min/max)
- Example calculations
- Graphs for non-linear curves

### 5. Edge Cases
Unusual or extreme situations:
- Minimum/maximum values
- Zero-division scenarios, overflow behavior
- Degenerate strategies and mitigations

### 6. Dependencies
What other systems this interacts with:
- Data flow direction
- Integration contract (what it provides and requires)

### 7. Tuning Knobs
Configurable values with:
- Intended range
- Category: feel / curve / gate
- Rationale for defaults
- All values in external data files (`assets/data/`), never hardcoded

### 8. Acceptance Criteria
Testable success conditions:
- Functional criteria (does it do the right thing?)
- Experiential criteria (does it FEEL right?)
- What a playtest should validate

## Theoretical Frameworks

### MDA Framework (Hunicke, LeBlanc, Zubek 2004)
Design from the player's emotional experience backward:
- **Aesthetics** (what the player FEELS): Sensation, Fantasy, Narrative, Challenge, Fellowship, Discovery, Expression, Submission
- **Dynamics** (emergent behaviors): patterns arising from mechanics during play
- **Mechanics** (rules we build): formal systems that generate dynamics

Always start with target aesthetics: "What should the player feel?"

### Self-Determination Theory (Deci & Ryan 1985)
Every system should satisfy at least one core need:
- **Autonomy**: Meaningful choices where multiple paths are viable
- **Competence**: Clear skill growth with readable feedback
- **Relatedness**: Connection to characters, players, or game world

### Flow State (Csikszentmihalyi 1990)
- **Sawtooth difficulty**: Tension builds → releases at milestone → re-engages higher
- **Feedback clarity**: Micro (<0.5s), meso (5-15min), macro (session-level)
- **Failure recovery**: Cost proportional to frequency of failure
- **Onboarding**: First 10 min teach through play, not tutorials

### Nested Loop Model
- **Micro-loop** (30s): Intrinsically satisfying core action
- **Meso-loop** (5-15min): Goal-reward cycle
- **Macro-loop** (session): Progression + natural stopping point + reason to return

### Player Motivation Types (Bartle + Quantic Foundry)
- **Achievers**: Progression, collections, mastery markers
- **Explorers**: Discovery, hidden content, systemic depth
- **Socializers**: Cooperative systems, shared experiences
- **Competitors**: PvP, leaderboards, skill expression

## Balancing Methodology

### Mathematical Modeling
- **Power curves**: linear, quadratic, logarithmic, or S-curve
- **DPS equivalence**: Normalize across damage/healing/utility profiles
- **TTK/TTC anchors**: Time-to-kill and time-to-complete as primary tuning targets

### Balance Types
- **Transitive**: A > B > C in cost and power (clear hierarchy)
- **Intransitive**: Rock-paper-scissors (circular counters)
- **Frustra**: Apparent imbalance with hidden counters
- **Asymmetric**: Different capabilities, equal viability

### Tuning Knob Categories
1. **Feel knobs**: Moment-to-moment (attack speed, movement speed). Tuned via playtesting.
2. **Curve knobs**: Progression shape (XP requirements, damage scaling). Tuned via math modeling.
3. **Gate knobs**: Pacing (level requirements, cooldowns). Tuned via session-length targets.

## Economy Design

### Sink/Faucet Model
- Map every faucet (source entering economy) and sink (destination removing)
- Balance over target session length
- Use Gini coefficient targets for wealth distribution health
- Apply pity systems for probabilistic rewards (guarantee within N attempts)

### Ethical Monetization
- No pay-to-win in competitive contexts
- No exploitative psychological dark patterns
- Transparent odds for probabilistic purchases
