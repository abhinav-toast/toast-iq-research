# Toast IQ Component Implementation Guide

> **Purpose:** Developer-ready spec for building the proposed Toast IQ generative UI components into the prototype. Covers design tokens, component schemas, CSS implementation details, render conditions, and composition rules — everything needed to go from mockup to working prototype.

---

## Part 1 — Design Token Foundation

All component values below reference these tokens. Use them directly; do not substitute with arbitrary values.

### Colors

#### Base ToastWeb
```
Brand Orange:     #FF4C00          (--color-brand-50)
Primary Blue:     #2B4FB9          (--color-primary-75)
Primary Dark:     #002182          (--color-primary-100)
Primary Tint:     #E3F0FB          (--color-primary-0)
Focus Ring:       #558EDD          (--color-focus)

Body Text:        #252525          (--color-rich-black)
Secondary Text:   #666666          (--color-gray-100)
Tertiary Text:    #7A7A7A          (--color-gray-75)
Disabled Text:    #C0C0C0          (--color-gray-50)

Surface White:    #FFFFFF
App BG:           #f0f0f0          (--color-gray-25)
Light BG:         #f8f9fc          ($white-lilac)

Default Border:   rgba(0,0,0,0.12) (--color-darken-12)
Input Border:     #dddde4          ($baby-seal)
Hover Tint:       rgba(0,0,0,0.04) (--color-darken-4)

Error Text:       #D40023          (--color-error-75)
Error BG:         #FFE6E9          (--color-error-0)
Error Border:     #EE6868
Warning Text:     #c54300          (--color-warning-75)
Warning BG:       #FEF6D9          (--color-warning-0)
Success Text:     #006400          (--color-success-75)
Success BG:       #E8F7D4          (--color-success-0)
```

#### Toast IQ — AI-exclusive palette (never use in base ToastWeb UI)
```
IQ Purple:        #7B5DB5          (.bg-sous-75)
IQ Purple Dark:   #513699          (.text-sous-100)
IQ Purple Tint:   #ECE8F4          (.bg-sous-0)
IQ Purple Border: #E0DCEF          (derived, ~rgba(123,93,181,0.25))
IQ Purple Light:  #F5F3FF          (derived surface)
```

#### IQ Gradient Presets
```
Action Card (2-color):
  conic-gradient(from var(--ga) at 50% 50%, #FF4C00, #E35EA3, #FF4C00)

Halo Container (4-color, pastel):
  conic-gradient(from var(--ga) at 50% 50%,
    #FFB093 0deg, #C498E3 120deg, #B1D4FF 240deg, #FFCDC0 360deg)

IQ Badge Pill (4-color, vivid):
  conic-gradient(from 45deg, #ff6128, #8a32c8, #63aaff, #ff9c81)

Memory Card (16-color, full rainbow):
  conic-gradient(from var(--ga),
    #F4B1EF, #B348FF, #9500FF, #1E00FF, #479BFF, #ABD2FF,
    #FF814C, #FF4C00, #FF5006, #FF6220, #2854D6, #388BFF,
    #7D8EF8, #DD55D2, #F4B1EF)
```

### Typography
```
Primary UI font:    Source Sans Pro, system-ui, -apple-system, "Helvetica Neue", Arial
Display/Heading:    Effra, system-ui, -apple-system, "Helvetica Neue", Arial
Monospace:          Menlo, Monaco, Consolas

Body (default):     17px / 24px  weight 400  letter-spacing 0.2px
Caption:            13px / 15px  weight 400  letter-spacing 0.4px
Label/UI text:      14–16px      weight 400–600
Component label:    10–12px      weight 700  text-transform uppercase  letter-spacing 0.4–0.5px
```

### Border Radius
```
Standard (inputs, buttons, cards):  4px
Medium (cards, modals):             6px
Large (panels, AI cards):          12px   ← use for all AI-generated content
Pill (badges, search):             9999px
```

### Shadows
```
Default card:
  inset 0 0 0 1px rgba(0,0,0,0.08),
  0 3px 1px -2px rgba(0,0,0,0.09),
  0 2px 2px 0 rgba(0,0,0,0.06),
  0 1px 5px 0 rgba(0,0,0,0.04)

Focus ring:
  0 0 0 3px #558EDD
```

### Gradient Animation (Required for all animated IQ borders)
```css
/* Register the custom property — must be in :root or @property at top of stylesheet */
@property --ga {
  syntax: '<angle>';
  initial-value: 0deg;
  inherits: false;
}

/* Keyframes */
@keyframes gradient-rotate { to { --ga: 360deg; } }

/* Speed reference */
Idle (ambient):   60s linear infinite
Focused:           8s linear infinite
AI processing:     4s linear infinite
Action card:       3s linear infinite
Loading/streaming: 1.5s linear infinite
```

### Gradient Border Trick
All animated gradient borders use the padding wrapper pattern — never `border`:
```html
<!-- wrapper has gradient background and padding -->
<div class="gradient-wrapper" style="padding:2px; border-radius:14px; background:conic-gradient(...)">
  <!-- inner has white background and is 2px smaller radius -->
  <div class="inner-card" style="background:white; border-radius:12px;">
    content
  </div>
</div>
```

### Ridealong Panel Dimensions
```
Width:    clamp(300px, 25vw, 450px)
Height:   calc(100vh - 4rem)   [4rem = 64px toast navbar]
Top:      4rem
Position: fixed, right: 0
Z-index:  21
Border:   1px solid rgba(0,0,0,0.12) on left side
```

---

## Part 2 — Skill Output Contract

Every skill must return this structure. The UI maps each field to a specific component.

```typescript
interface SkillOutput {
  // Primary outputs
  result: any                        // the skill's main answer
  mode: 'propose' | 'draft' | 'execute'

  // Trust fields (all required; null only if read-only skill)
  provenance: string                 // plain-language source description
  basis: string                      // the assumption the result rests on
  affected_entities: Entity[]        // downstream items touched by this change
  rollback_token: string | null      // null if read-only

  // Progressive disclosure (required two-tier structure)
  output: {
    summary: {
      headline: string               // one line: what the skill found/recommends
      recommendation: string         // the action or finding
      expected_outcome: string       // in plain language
    }
    detail: {
      reasoning: string
      data_sources: string[]
      alternatives: Alternative[]
      affected_entities: Entity[]
      projection_basis: string
    }
  }

  // Contextual fields (present when relevant)
  confidence: 'high' | 'medium' | 'low'
  confidence_reason: string          // "30d sales + 2 supplier quotes"
  alternatives: Alternative[]
  recommendation_timing: {
    sensitivity: 'high' | 'medium' | 'low'
    operator_cadence: 'immediate' | 'monthly' | 'quarterly' | 'annual'
    suggested_action_window: string  // derived, not hardcoded
    urgency_label: string            // generated from above
  }
  data_prerequisites: {
    passed: boolean
    checks: DataCheck[]              // each with status + remediation if failed
  }
}

// Draft output (when mode = 'draft')
interface DraftOutput {
  id: string                         // uuid
  created_by: string
  reviewer_role: 'owner' | 'operator' | 'manager'
  expires_at: string | null          // null = persistent
  scope: {
    type: 'single' | 'all' | 'select'
    location_ids: string[] | null
  }
  status: 'pending_review' | 'approved' | 'rejected' | 'modified'
  downstream_checklist: ChecklistItem[]
}
```

**Field → Component mapping:**
| Field | Component |
|---|---|
| `provenance` | ProvenanceLabel |
| `basis` | AnchoredProjection |
| `affected_entities` | CascadeMap |
| `rollback_token` | ReversibilityIndicator |
| `output.summary` + `output.detail` | DisclosureCard / RecommendationCard |
| `mode: propose` | RecommendationCard |
| `mode: draft` | DraftWorkspace |
| `mode: execute` | ConfirmationFrame + AuditEntry |
| `recommendation_timing` | UrgencyBadge |
| `reviewer_role` | ReviewerAssignment |
| `data_prerequisites.passed = false` | DataHealthCard (replaces RecommendationCard) |
| `confidence` | ConfidenceIndicator |
| `alternatives` | AlternativesPanel |

---

## Part 3 — Infrastructure Skills (Build First)

These three skills must exist before any domain skill. Every domain skill depends on them.

### `data_health_check`
```typescript
// Input
interface DataHealthCheckInput {
  dependencies: DataDependency[]     // e.g. ['recipe_costs', 'sales_history', 'inventory']
}

// Output
interface DataHealthCheckOutput {
  passed: boolean
  checks: {
    dependency: string
    status: 'healthy' | 'stale' | 'missing' | 'unverifiable'
    last_updated: string | null
    staleness_hours: number | null
    gap_description: string | null
    remediation: string | null        // "Upload CSV" | "Connect Sysco integration" | null
  }[]
}
```

### `cascade_mapper`
```typescript
// Input
interface CascadeMapperInput {
  entity_id: string
  entity_type: 'menu_item' | 'modifier_group' | 'modifier' | 'category' | 'bundle'
  proposed_change: {
    field: string                    // e.g. 'price'
    current_value: number | string
    proposed_value: number | string
  }
}

// Output
interface CascadeMapperOutput {
  root_entity: Entity
  affected: {
    entity: Entity
    relationship: string             // 'modifier_group' | 'modifier' | 'bundle' | 'delivery_item'
    current_value: number | string
    proposed_value: number | string
    delta: number | string
    risk_flag: 'arbitrage' | 'price_mismatch' | 'bundle_inconsistency' | null
  }[]
  arbitrage_warnings: {
    description: string
    entities_involved: string[]
    severity: 'high' | 'medium' | 'low'
  }[]
}
```

### `draft_manager`
```typescript
// Input
interface DraftManagerInput {
  change_package: {
    primary_change: Change
    cascade: CascadeMapperOutput
    downstream_checklist: ChecklistItem[]
    scope: DraftScope
    reviewer_role: 'owner' | 'operator' | 'manager'
  }
}

// Output
interface DraftManagerOutput {
  id: string
  status: 'pending_review'
  created_at: string
  reviewer_assignment: {
    role: string
    notified_users: string[]
    notified_at: string
  }
  rollback_token: string
  expires_at: string | null
}
```

---

## Part 4 — Components

Components are organized by trust phase. A lower-phase operator only sees lower-phase components — this is enforced by skill mode, not feature flags.

---

### Phase 1 — Intelligence (Observe)

> Read-only, ambient, zero friction. Never ask for an action.

---

#### SignalCard

**Purpose:** Surfaces an anomaly, trend, or benchmark signal contextually, where the operator already is.

**Render condition:** `mode = propose`, `data_health_check.passed = true`, no action CTA needed.

**Schema:**
```typescript
interface SignalCardProps {
  signal: string               // "Chicken breast cost up 18% in 60 days"
  implication: string          // "Your current margin is below target"
  sparkline_data?: number[]    // optional trend data points
  stats?: { value: string; label: string }[]
  provenance: ProvenanceLabelProps
  expand_target?: string       // id of a DetailPanel to open on click
}
```

**CSS spec:**
```css
.signal-card {
  background: white;
  border-radius: 12px;               /* AI card radius */
  border: 1px solid #E0DCEF;         /* IQ purple border */
  overflow: hidden;
  box-shadow: /* shadows-base */
    inset 0 0 0 1px rgba(0,0,0,0.08),
    0 3px 1px -2px rgba(0,0,0,0.09),
    0 2px 2px 0 rgba(0,0,0,0.06),
    0 1px 5px 0 rgba(0,0,0,0.04);
}

.signal-card-header {
  padding: 10px 14px;
  background: #F5F3FF;               /* IQ tint */
  border-bottom: 1px solid #E0DCEF;
  font-size: 12px;
  font-weight: 700;
  color: #513699;                    /* IQ purple dark */
  display: flex;
  align-items: center;
  gap: 8px;
}

.signal-card-stats {
  display: flex;
  border-top: 1px solid #F0ECFF;
}

.signal-stat {
  flex: 1;
  padding: 8px 10px;
  text-align: center;
  border-right: 1px solid #F0ECFF;
  font-size: 14px;
  font-weight: 800;
  color: #513699;
}

.signal-stat-label {
  font-size: 9px;
  color: #7A7A7A;
  text-transform: uppercase;
  letter-spacing: 0.3px;
  margin-top: 2px;
}
```

---

#### ProvenanceLabel

**Purpose:** Persistent inline attribution for any data-derived number. Never a tooltip — always visible. Load-bearing trust component; cannot be removed for space reasons.

**Schema:**
```typescript
interface ProvenanceLabelProps {
  source: string              // "Sysco invoice #INV-2024-8812"
  verified: boolean
  format: 'compact' | 'expanded'
  // compact: "Sysco invoice · verified"
  // expanded: full plain-language explanation of methodology
}
```

**CSS spec:**
```css
.provenance-label {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 11px;
  color: #7B5DB5;                    /* IQ purple */
  background: #F5F3FF;
  border: 1px solid #E0DCEF;
  border-radius: 9999px;             /* pill */
  padding: 2px 8px;
  font-weight: 500;
  white-space: nowrap;
}

.provenance-label-icon {
  font-size: 10px;
  opacity: 0.8;
}
```

**Behavior:**
- `compact` format shows inline next to the number it annotates
- `expanded` format shows in a popover or expandable row beneath the number
- Always rendered; never hidden behind a "more info" toggle

---

#### DataHealthCard

**Purpose:** Blocks the RecommendationCard when prerequisite data is stale or missing. Agent is being transparent about its own limitations — not punishing the operator.

**Render condition:** `data_prerequisites.passed = false`. Renders **instead of** the RecommendationCard.

**Schema:**
```typescript
interface DataHealthCardProps {
  checks: {
    dependency: string
    status: 'stale' | 'missing' | 'unverifiable' | 'healthy'
    gap_description: string | null
    remediation: string | null       // actionable label for the CTA button
    remediation_href?: string
  }[]
  affected_skill: string             // "Chicken sandwich cost analysis"
}
```

**CSS spec:**
```css
.data-health-card {
  background: white;
  border-radius: 12px;
  border: 1.5px solid #fcd9d2;       /* error border */
  overflow: hidden;
  box-shadow: /* shadows-base */;
}

.data-health-header {
  padding: 10px 14px;
  background: #FFE6E9;               /* error bg */
  border-bottom: 1px solid #fcd9d2;
  display: flex;
  align-items: center;
  gap: 8px;
}

.data-health-title {
  font-size: 13px;
  font-weight: 700;
  color: #D40023;                    /* error text */
  flex: 1;
}

.data-health-badge {
  font-size: 10px;
  font-weight: 700;
  background: #D40023;
  color: white;
  padding: 2px 7px;
  border-radius: 9999px;
}

.data-health-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 14px;
  font-size: 12px;
  color: #666666;
  border-bottom: 1px solid rgba(0,0,0,0.04);
}

.data-health-row.error { color: #D40023; }
.data-health-row.ok    { color: #006400; }

.data-health-action {
  padding: 10px 14px;
  font-size: 12px;
  font-weight: 700;
  color: #D40023;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 4px;
}

.data-health-action:hover { text-decoration: underline; }
```

---

#### BenchmarkBand

**Purpose:** Shows where the operator's current price sits in the local market range. Ambient awareness, no CTA. Reused in Phase 2 as supporting context inside RecommendationCard.

**Schema:**
```typescript
interface BenchmarkBandProps {
  item_name: string
  current_value: number
  market_range: { min: number; max: number }
  unit: string                       // "$/lb" | "$/item" | "%"
  sample_size?: number               // for ProvenanceLabel: n= value
  source?: string
}
```

**CSS spec:**
```css
.benchmark-band {
  background: white;
  border-radius: 8px;
  padding: 12px 14px;
  border: 1px solid rgba(0,0,0,0.12);
}

.benchmark-track {
  position: relative;
  height: 10px;
  background: #f0f0f0;
  border-radius: 5px;
  margin: 8px 0 4px;
}

.benchmark-range {
  position: absolute;
  height: 10px;
  border-radius: 5px;
  background: linear-gradient(90deg, #E8F7D4, #86efac); /* success tint gradient */
}

.benchmark-dot {
  position: absolute;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: #FF4C00;               /* brand orange — "you are here" */
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0,0,0,0.2);
  top: -3px;
  transform: translateX(-50%);
}

.benchmark-labels {
  display: flex;
  justify-content: space-between;
  font-size: 10px;
  color: #7A7A7A;
  margin-top: 4px;
}

.benchmark-you {
  font-size: 11px;
  font-weight: 700;
  color: #FF4C00;
  margin-top: 4px;
}
```

**Position calculation:**
```javascript
// dot left % = (current_value - market_range.min) / (market_range.max - market_range.min) * 100
// clamp to 5–95% so dot stays visible
const dotLeft = Math.min(95, Math.max(5,
  (current - min) / (max - min) * 100
)) + '%'
```

---

### Phase 2 — Recommendation (Propose)

> Complete case presented. Operator reads and decides. No state is modified.

---

#### RecommendationCard

**Purpose:** Primary Phase 2 component. Two-tier disclosure. The animated gradient border signals AI origin. The only action affordance is "Create draft" — never "Apply" or "Approve."

**Render condition:** `mode = propose`, `data_prerequisites.passed = true`.

**Schema:**
```typescript
interface RecommendationCardProps {
  headline: string
  expected_outcome: string
  reasoning: string
  urgency: UrgencyBadgeProps
  projection?: AnchoredProjectionProps
  confidence: ConfidenceIndicatorProps
  benchmark?: BenchmarkBandProps
  cascade_preview?: CascadeMapProps   // compact, not full tree
  alternatives?: AlternativesPanelProps
  provenance: ProvenanceLabelProps[]
  on_create_draft: () => void
}
```

**CSS spec:**
```css
/* Gradient border wrapper — animated */
.rec-card-wrapper {
  padding: 2px;
  border-radius: 14px;
  background: conic-gradient(from var(--ga, 0deg) at 50% 50%,
    #FF4C00, #E35EA3, #FF4C00);
  animation: gradient-rotate 3s linear infinite;
}

/* Inner card */
.rec-card {
  background: white;
  border-radius: 12px;
  overflow: hidden;
}

.rec-card-header {
  padding: 14px 16px;
  background: linear-gradient(135deg, #F5F3FF 0%, white 100%);
  border-bottom: 1px solid #E0DCEF;
  display: flex;
  align-items: flex-start;
  gap: 10px;
}

.rec-card-title {
  font-size: 15px;
  font-weight: 800;
  color: #252525;
  line-height: 1.3;
  flex: 1;
}

.rec-card-body {
  padding: 14px 16px;
  font-size: 13px;
  color: #666666;
  line-height: 1.6;
}

.rec-card-actions {
  display: flex;
  gap: 8px;
  padding: 12px 16px;
  border-top: 1px solid #F0ECFF;
}

.rec-btn-primary {
  flex: 1;
  padding: 8px 12px;
  border-radius: 4px;
  font-size: 14px;
  font-weight: 600;
  background: #2B4FB9;               /* primary blue — create draft */
  color: white;
  border: none;
  cursor: pointer;
  transition: background 150ms cubic-bezier(0.4, 0, 0.2, 1);
}

.rec-btn-primary:hover { background: #002182; }
.rec-btn-primary:focus-visible { box-shadow: 0 0 0 3px #558EDD; outline: none; }

.rec-btn-secondary {
  padding: 8px 12px;
  border-radius: 4px;
  font-size: 14px;
  font-weight: 600;
  background: white;
  color: #252525;
  border: 1px solid rgba(0,0,0,0.12);
  cursor: pointer;
}
```

**Summary tier** (always visible): headline, expected_outcome, UrgencyBadge.
**Detail tier** (expanded on click): reasoning, ProvenanceLabel per source, AnchoredProjection, ConfidenceIndicator, BenchmarkBand, CascadeMap preview, AlternativesPanel.

---

#### AnchoredProjection

**Purpose:** Renders any dollar/percentage projection. Always shows the number **and** the assumption it rests on. Never renders an unanchored number — if basis is unavailable, shows a range with explicit uncertainty bounds.

**Schema:**
```typescript
interface AnchoredProjectionProps {
  value: string                      // "$1,247/month"
  basis: string                      // "Based on 127 units/day avg (last 30 days)"
  delta?: string                     // "+$41/day"
  uncertainty_bounds?: {
    low: string
    base: string
    high: string
  }
  math_expandable: boolean           // "Show the math" expand toggle
}
```

**CSS spec:**
```css
.anchored-projection {
  background: white;
  border-radius: 8px;
  padding: 12px 14px;
  border: 1px solid rgba(0,0,0,0.12);
  margin: 8px 0;
}

.ap-primary-number {
  font-size: 26px;
  font-weight: 900;
  color: #D40023;                    /* error/impact red for negative cost */
  /* use #006400 (success green) for positive outcome projections */
  line-height: 1;
  margin-bottom: 2px;
}

.ap-delta-pill {
  display: inline-block;
  font-size: 11px;
  font-weight: 700;
  color: #D40023;
  background: #FFE6E9;
  padding: 2px 7px;
  border-radius: 9999px;
  border: 1px solid #fcd9d2;
}

.ap-basis {
  font-size: 11px;
  color: #7A7A7A;
  margin-top: 4px;
  line-height: 1.4;
}

/* Confidence range bar */
.ap-range-track {
  height: 6px;
  background: #ECE8F4;               /* IQ tint */
  border-radius: 3px;
  margin: 8px 0 4px;
  overflow: hidden;
}

.ap-range-fill {
  height: 6px;
  border-radius: 3px;
  background: linear-gradient(90deg, #7B5DB5, #E35EA3);
}

.ap-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 9px;
  color: #C0C0C0;
}
```

---

#### ConfidenceIndicator

**Purpose:** Score (High / Medium / Low) with one-line reason. The reason is the trust signal — the score alone is insufficient.

**Schema:**
```typescript
interface ConfidenceIndicatorProps {
  level: 'high' | 'medium' | 'low'
  reason: string                     // "30d sales + 2 supplier quotes"
  filled_dots: number                // 1–5; high=4, medium=3, low=2
}
```

**CSS spec:**
```css
.confidence-indicator {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 12px;
  padding: 8px 0;
}

.confidence-dots {
  display: flex;
  gap: 3px;
}

.confidence-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #E0DCEF;
}

.confidence-dot.filled { background: #7B5DB5; }

.confidence-level {
  font-weight: 700;
  color: #7B5DB5;
  font-size: 12px;
}

.confidence-reason {
  font-size: 11px;
  color: #7A7A7A;
}
```

---

#### CascadeMap

**Purpose:** Visualizes every entity affected by a proposed change before the confirm step. Always rendered upfront — not on expand. P04's modifier incident was a direct consequence of this missing.

**Render condition:** `affected_entities` present in skill output. Required, not optional.

**Schema:**
```typescript
interface CascadeMapProps {
  root_entity: { name: string; type: string }
  affected: {
    entity: { name: string; type: string }
    relationship: string
    current_value: string
    proposed_value: string
    delta: string
    risk_flag: 'arbitrage' | 'price_mismatch' | null
    depth: number                    // 1 = direct child, 2 = grandchild
  }[]
  arbitrage_warnings: {
    description: string
    severity: 'high' | 'medium' | 'low'
  }[]
  compact?: boolean                  // true = list view; false = full tree
}
```

**CSS spec:**
```css
.cascade-map {
  background: white;
  border-radius: 10px;
  border: 1px solid #E0DCEF;
  overflow: hidden;
  margin: 8px 0;
}

.cascade-header {
  padding: 9px 14px;
  background: #F5F3FF;
  border-bottom: 1px solid #E0DCEF;
  font-size: 12px;
  font-weight: 700;
  color: #513699;
  display: flex;
  align-items: center;
  gap: 6px;
}

.cascade-tree {
  padding: 10px 14px;
}

.cascade-node {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  padding: 4px 0;
  font-size: 12px;
  color: #252525;
}

.cascade-node[data-depth="1"] { padding-left: 20px; }
.cascade-node[data-depth="2"] { padding-left: 40px; }

.cascade-impact {
  font-size: 10px;
  font-weight: 700;
  color: #D40023;
  margin-top: 1px;
}

.cascade-warning-banner {
  margin: 8px 14px 12px;
  background: #FEF6D9;
  border: 1px solid #fce9c0;
  border-radius: 6px;
  padding: 8px 12px;
  font-size: 11px;
  color: #c54300;
  font-weight: 600;
}
```

**Rendering rule:** 2 or fewer affected entities → compact list. 3+ entities or arbitrage warnings → full tree, warnings surfaced at top.

---

#### AlternativesPanel

**Purpose:** When the skill generates multiple response paths, renders them as selectable options with projected outcomes. Operator chooses the path to draft. Implements P01's "magic wand" pattern — the agent offers options, not prescriptions.

**Schema:**
```typescript
interface AlternativesPanelProps {
  alternatives: {
    id: string
    title: string
    description: string
    projected_outcome: string
    tag: 'recommended' | 'partial' | 'not_recommended'
    risk: 'safe' | 'neutral' | 'risky'
  }[]
  selected_id: string | null
  on_select: (id: string) => void
}
```

**CSS spec:**
```css
.alternatives-panel {
  background: #faf8f5;
  border: 1px solid rgba(0,0,0,0.12);
  border-radius: 10px;
  overflow: hidden;
  margin: 8px 0;
}

.alternatives-header {
  padding: 9px 14px;
  font-size: 11px;
  font-weight: 700;
  color: #666666;
  border-bottom: 1px solid rgba(0,0,0,0.08);
  background: white;
}

.alternative-item {
  padding: 10px 14px;
  border-bottom: 1px solid rgba(0,0,0,0.06);
  display: flex;
  align-items: flex-start;
  gap: 10px;
  cursor: pointer;
  transition: background 150ms ease;
}

.alternative-item:last-child { border-bottom: none; }
.alternative-item:hover { background: white; }
.alternative-item.selected { background: #F5F3FF; }

/* Radio dot */
.alt-radio {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  border: 2px solid #C0C0C0;
  flex-shrink: 0;
  margin-top: 2px;
  transition: all 150ms ease;
}

.alternative-item.selected .alt-radio {
  border-color: #7B5DB5;
  background: #7B5DB5;
  box-shadow: inset 0 0 0 2px white;
}

.alt-title {
  font-size: 13px;
  font-weight: 700;
  color: #252525;
}

.alt-description {
  font-size: 12px;
  color: #666666;
  margin-top: 2px;
  line-height: 1.4;
}

/* Risk tags */
.alt-tag {
  display: inline-block;
  font-size: 10px;
  font-weight: 700;
  padding: 2px 7px;
  border-radius: 9999px;
  margin-top: 4px;
}

.alt-tag.safe    { background: #E8F7D4; color: #006400; border: 1px solid #bbf7d0; }
.alt-tag.neutral { background: #E3F0FB; color: #2B4FB9; border: 1px solid #bfdbfe; }
.alt-tag.risky   { background: #FFE6E9; color: #D40023; border: 1px solid #fcd9d2; }
```

---

#### UrgencyBadge

**Purpose:** Visual weight derived from the intersection of issue sensitivity × operator's actual cadence. Never a hardcoded string — always generated from `recommendation_timing`.

**Schema:**
```typescript
interface UrgencyBadgeProps {
  level: 'high' | 'medium' | 'low'
  label: string                      // generated from recommendation_timing
  // Examples:
  // high + immediate cadence   → "Act today — $41/day ongoing cost"
  // high + quarterly cadence   → "Consider for Q3 menu review"
  // medium + monthly cadence   → "Review before next pricing cycle"
  // low + any cadence          → "When convenient"
}
```

**CSS spec:**
```css
.urgency-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 10px;
  font-weight: 800;
  padding: 3px 9px;
  border-radius: 9999px;
  white-space: nowrap;
  letter-spacing: 0.2px;
}

.urgency-badge.high {
  background: #FFE6E9;
  color: #D40023;
  border: 1px solid #fcd9d2;
}

.urgency-badge.medium {
  background: #FEF6D9;
  color: #c54300;
  border: 1px solid #fce9c0;
}

.urgency-badge.low {
  background: #E8F7D4;
  color: #006400;
  border: 1px solid #bbf7d0;
}
```

---

### Phase 3 — Draft (Prepare)

> Agent's work product. A complete, persistent change package the operator reviews. Nothing is published until a designated approver acts.

---

#### DraftWorkspace

**Purpose:** Primary Phase 3 surface. Contains the complete change package: the primary change, CascadeMap (full), ScopeSelector, ReviewerAssignment, DownstreamChecklist, SandboxPreview, and approve/reject/modify affordances. Persists across sessions. Shareable with an approver who didn't initiate.

**Render condition:** `mode = draft`, draft record exists in `draft_manager`.

**CSS spec:**
```css
.draft-workspace {
  background: white;
  border-radius: 12px;
  border: 1.5px solid #E0DCEF;
  overflow: hidden;
  box-shadow: /* shadows-base */;
}

.draft-header {
  padding: 12px 16px;
  background: linear-gradient(135deg, #F5F3FF 0%, white 100%);
  border-bottom: 1px solid #E0DCEF;
  display: flex;
  align-items: center;
  gap: 8px;
}

.draft-title {
  font-size: 14px;
  font-weight: 700;
  color: #513699;
  flex: 1;
}

.draft-status-badge {
  font-size: 10px;
  font-weight: 700;
  padding: 3px 8px;
  border-radius: 9999px;
}

.draft-status-badge.pending {
  background: #FEF6D9;
  color: #c54300;
  border: 1px solid #fce9c0;
}

.draft-status-badge.approved {
  background: #E8F7D4;
  color: #006400;
  border: 1px solid #bbf7d0;
}

.draft-body {
  padding: 14px 16px;
}

.draft-field-label {
  font-size: 10px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: #7A7A7A;
  margin-bottom: 4px;
}

.draft-field-value {
  font-size: 14px;
  font-weight: 600;
  color: #252525;
  margin-bottom: 12px;
}

.draft-footer {
  display: flex;
  gap: 8px;
  padding: 12px 16px;
  border-top: 1px solid #F0ECFF;
}

/* Primary: Submit for Review (not "Approve" or "Publish") */
.draft-submit-btn {
  flex: 1;
  padding: 8px 12px;
  border-radius: 4px;
  font-size: 14px;
  font-weight: 600;
  background: #7B5DB5;               /* IQ purple — draft action */
  color: white;
  border: none;
  cursor: pointer;
  transition: background 150ms ease;
}

.draft-submit-btn:hover { background: #513699; }
```

---

#### ScopeSelector

**Purpose:** Set at draft creation. Locked once in review. Scope is visible throughout — the reviewer always knows what they're approving.

**Schema:**
```typescript
interface ScopeSelectorProps {
  value: 'single' | 'all' | 'select'
  location_ids?: string[]            // populated when value = 'select'
  locked: boolean                    // true when draft status = 'pending_review'
  available_locations?: { id: string; name: string }[]
}
```

**CSS spec:**
```css
.scope-selector {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
  margin: 8px 0;
}

.scope-chip {
  padding: 5px 12px;
  border-radius: 9999px;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
  border: 1.5px solid transparent;
  transition: all 150ms ease;
  user-select: none;
}

.scope-chip.selected {
  background: #7B5DB5;
  color: white;
  border-color: #7B5DB5;
}

.scope-chip.unselected {
  background: #F5F3FF;
  color: #7B5DB5;
  border-color: #E0DCEF;
}

.scope-chip.locked {
  cursor: default;
  opacity: 0.8;
}

.scope-chip:focus-visible {
  box-shadow: 0 0 0 3px #558EDD;
  outline: none;
}
```

---

#### ReviewerAssignment

**Purpose:** Routes draft to a role, not a user. Shows who in that role was notified and when. Draft cannot auto-advance — it stays `pending_review` until someone in the role acts.

**Schema:**
```typescript
interface ReviewerAssignmentProps {
  role: string                       // "Owner" | "GM" | "Operations Manager"
  notified_users: { name: string; avatar_initials: string }[]
  notified_at: string
  status: 'pending' | 'approved' | 'rejected'
}
```

**CSS spec:**
```css
.reviewer-assignment {
  display: flex;
  align-items: center;
  gap: 10px;
  background: #F5F3FF;
  border-radius: 8px;
  padding: 10px 14px;
  border: 1px solid #E0DCEF;
  margin: 8px 0;
}

.reviewer-avatar {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: #2B4FB9;
  color: white;
  font-size: 10px;
  font-weight: 800;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.reviewer-name {
  font-size: 13px;
  font-weight: 700;
  color: #252525;
}

.reviewer-role {
  font-size: 11px;
  color: #7A7A7A;
  margin-top: 1px;
}

.reviewer-status {
  margin-left: auto;
  font-size: 10px;
  font-weight: 700;
  padding: 3px 8px;
  border-radius: 9999px;
}

.reviewer-status.pending {
  background: #FEF6D9;
  color: #c54300;
  border: 1px solid #fce9c0;
}

.reviewer-status.approved {
  background: #E8F7D4;
  color: #006400;
  border: 1px solid #bbf7d0;
}
```

---

#### SandboxPreview

**Purpose:** Shows what metrics will look like after the draft is applied. Clearly labeled "Simulated · not live." P01's explicit request — eliminates the fear of publishing something unexpected.

**Schema:**
```typescript
interface SandboxPreviewProps {
  simulation_basis: string           // "Based on 127 units/day avg (last 30 days)"
  metrics: {
    label: string
    current: number
    projected: number
    target?: number
    unit: string                     // "%" | "$" | "units"
    direction: 'higher_is_better' | 'lower_is_better'
  }[]
}
```

**CSS spec:**
```css
/* Gradient border wrapper — always present, non-animated */
.sandbox-wrapper {
  background: white;
  border-radius: 12px;
  border: 1.5px solid #7B5DB5;
  overflow: hidden;
  margin: 8px 0;
  box-shadow: 0 0 0 3px rgba(123,93,181,0.1); /* IQ purple halo */
}

.sandbox-header {
  padding: 10px 14px;
  background: linear-gradient(90deg, #F5F3FF, white);
  border-bottom: 1px solid #E0DCEF;
  display: flex;
  align-items: center;
  gap: 8px;
}

.sandbox-title {
  font-size: 12px;
  font-weight: 700;
  color: #7B5DB5;
  flex: 1;
}

.sandbox-tag {
  font-size: 9px;
  font-weight: 700;
  background: #F5F3FF;
  color: #7B5DB5;
  border: 1px solid #E0DCEF;
  padding: 2px 7px;
  border-radius: 9999px;
}

.sandbox-body { padding: 12px 14px; }

.sandbox-bar-row { margin-bottom: 10px; }

.sandbox-bar-labels {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  margin-bottom: 3px;
}

.sandbox-bar-labels .bar-name { color: #666666; font-weight: 600; }
.sandbox-bar-labels .bar-val  { font-weight: 800; color: #7B5DB5; }
/* Override for current (bad) value */
.sandbox-bar-labels .bar-val.bad { color: #D40023; }
/* Override for target */
.sandbox-bar-labels .bar-val.target { color: #006400; }

.sandbox-track {
  height: 8px;
  background: #ECE8F4;
  border-radius: 4px;
  overflow: hidden;
}

.sandbox-fill {
  height: 8px;
  border-radius: 4px;
  background: linear-gradient(90deg, #7B5DB5, #E35EA3);
}

/* "bad" fill for current state bar */
.sandbox-fill.bad { background: #D40023; }
/* "good" fill for target state bar */
.sandbox-fill.target { background: #006400; }

.sandbox-footer {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px;
  border-top: 1px solid #F0ECFF;
  font-size: 11px;
  color: #7A7A7A;
}

.sandbox-apply-btn {
  margin-left: auto;
  background: #7B5DB5;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 6px 14px;
  font-size: 12px;
  font-weight: 700;
  cursor: pointer;
}
```

---

#### DownstreamChecklist

**Purpose:** Pre-flight list generated as part of the draft, not after publish. Includes post-publish manual steps specific to this change. P01 and P04's downstream coordination requirement — built into the draft, not an afterthought.

**Schema:**
```typescript
interface DownstreamChecklistProps {
  items: {
    label: string
    status: 'checked' | 'warning' | 'manual'
    note?: string                    // additional context for warning/manual items
    reviewer?: string                // "Reviewer: Jamie R." for approval items
    link?: { label: string; href: string }
  }[]
}
```

**CSS spec:**
```css
.downstream-checklist {
  background: white;
  border: 1px solid rgba(0,0,0,0.12);
  border-radius: 10px;
  overflow: hidden;
  margin: 8px 0;
}

.checklist-header {
  padding: 9px 14px;
  background: #faf8f5;
  border-bottom: 1px solid rgba(0,0,0,0.08);
  font-size: 12px;
  font-weight: 700;
  color: #666666;
  display: flex;
  align-items: center;
  gap: 6px;
}

.checklist-item {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  padding: 8px 14px;
  border-bottom: 1px solid rgba(0,0,0,0.06);
  font-size: 12px;
  color: #252525;
}

.checklist-item:last-child { border-bottom: none; }

.checklist-checkbox {
  width: 15px;
  height: 15px;
  border-radius: 3px;
  flex-shrink: 0;
  margin-top: 2px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 9px;
  font-weight: 900;
  color: white;
}

.checklist-checkbox.checked { background: #006400; border: 1.5px solid #006400; }
.checklist-checkbox.warning { background: #c54300; border: 1.5px solid #c54300; }
.checklist-checkbox.manual  { background: transparent; border: 1.5px solid #C0C0C0; }

.checklist-note {
  font-size: 11px;
  color: #7A7A7A;
  margin-top: 2px;
  line-height: 1.4;
}

.checklist-reviewer-tag {
  display: inline-flex;
  align-items: center;
  gap: 3px;
  font-size: 9px;
  font-weight: 700;
  background: #FEF6D9;
  color: #c54300;
  border: 1px solid #fce9c0;
  padding: 1px 6px;
  border-radius: 9999px;
  margin-top: 3px;
}
```

---

#### ReversibilityIndicator

**Purpose:** Shows what can be rolled back, for how long, and what the rollback restores. Present on every DraftWorkspace and ConfirmationFrame. Rendered prominently — not buried. Load-bearing trust signal; cannot be removed for space reasons.

**Schema:**
```typescript
interface ReversibilityIndicatorProps {
  level: 'high' | 'medium' | 'low' | 'none'
  label: string
  // Examples:
  // high:   "Reversible — undo anytime in Menu Manager"
  // medium: "Reversible for 48h — then requires manual rollback"
  // low:    "Partially reversible — contact support to undo pricing history"
  // none:   "Not reversible — this action cannot be undone"
  rollback_token?: string
}
```

**CSS spec:**
```css
.reversibility-indicator {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 11px;
  font-weight: 700;
  padding: 5px 12px;
  border-radius: 9999px;
  margin: 8px 0;
}

.reversibility-indicator.high   { background: #E8F7D4; color: #006400; border: 1px solid #bbf7d0; }
.reversibility-indicator.medium { background: #FEF6D9; color: #c54300; border: 1px solid #fce9c0; }
.reversibility-indicator.low    { background: #FFE6E9; color: #D40023; border: 1px solid #fcd9d2; }
.reversibility-indicator.none   { background: #FFE6E9; color: #D40023; border: 1px solid #fcd9d2; }
```

---

### Phase 4 — Autonomy (Execute and Report)

> Exception-based. Surface only after Phase 3 trust is established. Operator is not reviewing every action — only deviations.

---

#### GuardrailStatus

**Purpose:** Shows the operator's defined rules and where the agent is operating relative to them. Contextual — shown when the agent is about to act, not as a permanent dashboard.

**Schema:**
```typescript
interface GuardrailStatusProps {
  rules: {
    label: string                    // "Price changes"
    max: string                      // "≤15%"
    current: string                  // "This rec: 9.6%"
    status: 'within' | 'approaching' | 'at_limit'
  }[]
}
```

**CSS spec:**
```css
.guardrail-status {
  background: white;
  border-radius: 10px;
  border: 1.5px solid #bbf7d0;       /* success green border */
  padding: 12px 14px;
  margin: 8px 0;
}

.guardrail-header {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  font-weight: 700;
  color: #006400;
  margin-bottom: 10px;
}

.guardrail-rule {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 0;
  border-bottom: 1px solid #E8F7D4;
  font-size: 12px;
  color: #252525;
}

.guardrail-rule:last-child { border-bottom: none; }

.guardrail-value { font-weight: 700; }
.guardrail-value.within     { color: #006400; }
.guardrail-value.approaching { color: #c54300; }
.guardrail-value.at_limit   { color: #D40023; }
```

---

#### DailySummary

**Purpose:** Morning briefing card. Actions taken (with audit links), recommendations pending, guardrail boundaries approaching. Designed to be reviewed in under 60 seconds — a brief, not a report.

**Schema:**
```typescript
interface DailySummaryProps {
  date: string
  items: {
    type: 'action_taken' | 'recommendation_pending' | 'guardrail_approaching' | 'data_update'
    label: string
    detail?: string
    audit_id?: string                // link to AuditEntry
    priority: 'high' | 'normal' | 'low'
  }[]
}
```

**CSS spec:**
```css
.daily-summary {
  background: white;
  border-radius: 10px;
  border: 1px solid rgba(0,0,0,0.12);
  overflow: hidden;
  margin: 8px 0;
}

.daily-summary-header {
  padding: 10px 14px;
  background: #faf8f5;
  border-bottom: 1px solid rgba(0,0,0,0.08);
  font-size: 12px;
  font-weight: 700;
  color: #666666;
  display: flex;
  align-items: center;
  gap: 6px;
}

.summary-item {
  display: flex;
  gap: 10px;
  padding: 8px 14px;
  border-bottom: 1px solid rgba(0,0,0,0.06);
  align-items: flex-start;
}

.summary-item:last-child { border-bottom: none; }

.summary-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  flex-shrink: 0;
  margin-top: 5px;
}

.summary-dot.action_taken            { background: #006400; }     /* green */
.summary-dot.recommendation_pending  { background: #FF4C00; }     /* orange */
.summary-dot.guardrail_approaching   { background: #c54300; }     /* warning */
.summary-dot.data_update             { background: #2B4FB9; }     /* blue */

.summary-text {
  font-size: 12px;
  color: #252525;
  line-height: 1.45;
}

.summary-detail {
  font-size: 11px;
  color: #7A7A7A;
  margin-top: 2px;
}

.summary-audit-link {
  font-size: 11px;
  color: #2B4FB9;
  font-weight: 600;
  text-decoration: none;
  margin-top: 2px;
  display: inline-block;
}
```

---

#### AuditEntry

**Purpose:** Every agent action creates one. Contains timestamp, action taken, reasoning summary, data sources, expected outcome, rollback token + expiry. Reachable from DailySummary — not buried in settings.

**Schema:**
```typescript
interface AuditEntryProps {
  entries: {
    timestamp: string
    actor_type: 'agent' | 'human'
    actor_name: string
    action: string
    detail?: string
  }[]
  rollback_token?: string
  rollback_expires_at?: string
}
```

**CSS spec:**
```css
.audit-log {
  background: #faf8f5;
  border: 1px solid rgba(0,0,0,0.12);
  border-radius: 10px;
  overflow: hidden;
  margin: 8px 0;
}

.audit-header {
  padding: 9px 14px;
  background: white;
  border-bottom: 1px solid rgba(0,0,0,0.08);
  font-size: 12px;
  font-weight: 700;
  color: #666666;
  display: flex;
  align-items: center;
  gap: 6px;
}

.audit-entry {
  padding: 8px 14px;
  border-bottom: 1px solid rgba(0,0,0,0.06);
  display: flex;
  gap: 10px;
  align-items: flex-start;
}

.audit-entry:last-child { border-bottom: none; }

.audit-timestamp {
  font-size: 10px;
  color: #C0C0C0;
  font-family: Menlo, Monaco, Consolas, monospace;
  white-space: nowrap;
  margin-top: 2px;
  flex-shrink: 0;
}

.audit-text {
  font-size: 12px;
  color: #666666;
  line-height: 1.45;
  flex: 1;
}

/* Actor pills */
.actor-pill {
  display: inline-flex;
  align-items: center;
  gap: 3px;
  font-size: 9px;
  font-weight: 700;
  padding: 1px 7px;
  border-radius: 9999px;
  margin-right: 4px;
}

.actor-pill.agent { background: #F5F3FF; color: #7B5DB5; border: 1px solid #E0DCEF; }
.actor-pill.human { background: #E3F0FB; color: #2B4FB9; border: 1px solid #bfdbfe; }
```

---

#### GuardrailStatus (Reassurance Banner variant)

Used at the bottom of any agent action surface to answer the implicit question: "what did the AI do to my account?"

```html
<div class="guardrail-reassurance">
  <span class="guardrail-shield">🛡️</span>
  <p><strong>IQ never writes without approval.</strong> All changes above required
     a human to confirm. Agent access is read-only until you explicitly publish.</p>
</div>
```

```css
.guardrail-reassurance {
  background: white;
  border: 1.5px solid #bbf7d0;
  border-radius: 10px;
  padding: 12px 14px;
  display: flex;
  align-items: flex-start;
  gap: 10px;
  margin: 8px 0;
  font-size: 12px;
  color: #252525;
  line-height: 1.55;
}

.guardrail-reassurance strong { color: #006400; font-weight: 700; }
```

---

## Part 5 — Composition Decision Tree

When a skill completes and is ready to render, walk this tree to determine the component set. The agent follows these rules deterministically — no aesthetic choices.

```
1. Did data_health_check pass?
   ├─ NO  → render DataHealthCard only. STOP.
   └─ YES → continue to step 2.

2. What is skill.mode?
   ├─ 'propose'  → start with RecommendationCard
   ├─ 'draft'    → start with DraftWorkspace
   └─ 'execute'  → start with ConfirmationFrame

3. Does output contain affected_entities?
   ├─ YES → inject CascadeMap before confirm affordance. REQUIRED.
   └─ NO  → skip.

4. Does output contain a projection (basis + value)?
   ├─ YES → render AnchoredProjection. Refuse to render raw unanchored number.
   └─ NO  → skip.

5. Does output contain alternatives?
   ├─ YES → inject AlternativesPanel into detail tier.
   └─ NO  → skip.

6. Does the draft require reviewer_role?
   ├─ YES → inject ReviewerAssignment + ScopeSelector into DraftWorkspace.
   └─ NO  → skip.

7. Does operator have non-immediate cadence?
   ├─ YES → generate UrgencyBadge from recommendation_timing object.
   └─ NO  → use default urgency language.

8. Is rollback_token present?
   ├─ YES → render ReversibilityIndicator prominently.
   └─ NO  → render read-only indicator ("this action cannot be undone") if mode = execute.
```

---

## Part 6 — Composition Rules

**Rule 1 — Phase gates the component set.**
A Phase 1 operator only sees Intelligence Components. DraftWorkspace doesn't exist for them — not because it's hidden, but because the skill runs in `propose` mode and won't create a draft object until trust is established. The component set is unlocked by trust phase, not feature flags.

**Rule 2 — Skill output fields determine required components.**
If `affected_entities` is returned, CascadeMap is required. If a projection is returned without `basis`, AnchoredProjection must refuse to render an unanchored number and surface a degraded state instead. The component enforces the output contract.

**Rule 3 — Complexity determines component depth.**
2 affected entities → CascadeMap renders as a compact list. 10+ entities or arbitrage warnings → CascadeMap renders as an expanded tree with warnings surfaced at top. Component adapts to input complexity, not a fixed template.

**Rule 4 — Operator behavior refines defaults.**
If the operator has expanded the detail tier of the last 8 RecommendationCards, default to detail-open. If they've dismissed UrgencyBadges without reading them, de-escalate urgency framing for this operator. Defaults are operator-specific.

**Rule 5 — Trust signals are never optional.**
ProvenanceLabel, AnchoredProjection basis, and ReversibilityIndicator cannot be removed for space or aesthetic reasons. They are load-bearing. If the layout can't fit them, the layout is wrong.

---

## Part 7 — Anti-Patterns to Retire

| Anti-pattern | Problem | Fix |
|---|---|---|
| Single-step action skills | Takes input → directly modifies state with no draft/review step | Decompose into propose → draft → execute |
| Unattributed numbers | Returns dollar/% figure without basis and provenance | Always include `basis` and `provenance` in output; use AnchoredProjection |
| Hardcoded urgency copy | Static "act now" / "effective tomorrow" strings | Replace with `recommendation_timing` object; UrgencyBadge generates the label |
| Initiator-as-approver | Assumes the person who ran the skill can approve its output | Add `reviewer_role` to draft; route to role, not user |
| Flat output structure | Returns undifferentiated text or single result object | Restructure to summary/detail two-tier schema |
| Silent cascade | Change skill proposes modifying a priced entity without mapping modifier impact | Run `cascade_mapper` before surfacing any recommendation |

---

## Part 8 — Behavioral Mode Reference

Every action skill must declare and respect a behavioral mode. No skill defaults to `execute`.

| Mode | What it does | State created | Can auto-advance to next? |
|---|---|---|---|
| `propose` | Returns recommendation with full reasoning | None | No — operator must click "Create draft" |
| `draft` | Creates a persistent change package | Draft record in `draft_manager` | No — requires reviewer approval |
| `execute` | Applies the change | Permanent change | N/A (terminal state) |

Skills that currently go directly to action must be decomposed. These are not the same skill with a confirmation dialog — they are structurally separate phases with different state ownership.
