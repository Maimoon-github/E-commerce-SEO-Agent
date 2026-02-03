Below is a **directly implementable** mapping of the design into:

1. a **fully specified LangGraph State schema** (fields \+ types \+ ownership)  
2. a **CrewAI Flow layout** (agents \+ tasks \+ execution order \+ routing rules)  
3. **exact node names**, explicit **routing/transition rules**, and **ChangeSet JSON contracts** per node (inputs/outputs/state mutations)

This follows the “Observe → Diagnose → Recommend → Approve → Execute → Validate → Learn” loop you defined.

---

# **1\) LangGraph State schema (fully specified)**

## **1.1 Schema: Python types (Pydantic models)**

**Determinism rules baked in**

* All lists that represent “queues” or “ranked outputs” must be **sorted** using the exact keys specified below.  
* IDs are deterministic: `sha256(run_id + stable_key_fields)` truncated to 16 chars.  
* No node may write outside its owned fields (ownership table below).

from \_\_future\_\_ import annotations

from pydantic import BaseModel, Field  
from typing import Literal, Optional, Dict, List, Any  
from datetime import datetime

\# \---------- Core primitives \----------

Market \= Literal\["US", "EU", "AU"\]  
OpportunityType \= Literal\["TECH", "PRODUCT", "CONTENT", "AUTHORITY"\]  
IssueType \= Literal\[  
    "INDEXATION", "CRAWL", "CWV", "SCHEMA", "DUPLICATION", "CANONICAL", "INTERNAL\_LINKING",  
    "PDP\_THIN", "CATEGORY\_THIN", "CANNIBALIZATION", "CTR\_DROP", "RANK\_DROP", "BACKLINK\_GAP"  
\]  
ChangeSetKind \= Literal\[  
    "TECH\_FIX", "PRODUCT\_PDP\_UPDATE", "CATEGORY\_UPDATE", "META\_UPDATE",  
    "INTERNAL\_LINKS\_UPDATE", "CONTENT\_BRIEF", "CONTENT\_REFRESH", "AUTHORITY\_OUTREACH\_PLAN"  
\]  
ApprovalStatus \= Literal\["NOT\_REQUIRED", "PENDING", "APPROVED", "REJECTED", "PARTIAL"\]  
ExecutionStatus \= Literal\["SKIPPED", "QUEUED", "APPLIED", "FAILED", "ROLLED\_BACK"\]

class RunMeta(BaseModel):  
    run\_id: str  
    source: Literal\["schedule", "webhook", "manual"\]  
    started\_at: datetime  
    ended\_at: Optional\[datetime\] \= None  
    environment: Literal\["dev", "staging", "prod"\]  
    brand: Literal\["Aurum Pickleball"\] \= "Aurum Pickleball"  
    domain: str \= "https://aurumpickleball.com"

class IntegrationsRef(BaseModel):  
    \# references ONLY, never secrets  
    gsc\_property\_url: str  
    ga4\_property\_id: str  
    serp\_provider: Literal\["serpapi", "dataforseo", "none"\]  
    cms\_provider: Literal\["shopify", "woocommerce", "headless", "none"\]  
    db\_provider: Literal\["postgres", "bigquery", "none"\]

class RiskConfig(BaseModel):  
    approval\_threshold: int \= Field(4, ge=0, le=5)          \# risk \>= 4 \-\> approval interrupt  
    auto\_execute\_max\_risk: int \= Field(2, ge=0, le=5)        \# risk \<= 2 may auto-apply  
    blocklist\_kinds: List\[ChangeSetKind\] \= Field(default\_factory=list)

class ScoringConfig(BaseModel):  
    \# Priority Score \= (Impact \* Confidence) / max(Effort, 1\)  
    \# Tie-break: higher impact, then higher confidence, then lower effort, then URL asc  
    min\_priority\_score: float \= 0.25  
    max\_items\_per\_run: int \= 50

class RoutingConfig(BaseModel):  
    \# strict routing by OpportunityType  
    allow\_types: List\[OpportunityType\] \= Field(default\_factory=lambda: \["TECH","PRODUCT","CONTENT","AUTHORITY"\])  
    market\_order: List\[Market\] \= Field(default\_factory=lambda: \["US","EU","AU"\])  \# deterministic

class AgentConfig(BaseModel):  
    \# references to models/providers by name (no keys here)  
    planner\_model: str  
    writer\_model: str  
    validator\_model: Optional\[str\] \= None

class Config(BaseModel):  
    integrations: IntegrationsRef  
    scoring: ScoringConfig  
    risk: RiskConfig  
    routing: RoutingConfig  
    agent: AgentConfig

\# \---------- Inputs snapshots \----------

class GSCSnapshot(BaseModel):  
    collected\_at: datetime  
    window\_days: int  
    pages: List\[Dict\[str, Any\]\]        \# normalized later  
    queries: List\[Dict\[str, Any\]\]      \# normalized later  
    index\_coverage: List\[Dict\[str, Any\]\]  
    sitemap\_status: List\[Dict\[str, Any\]\]

class GA4Snapshot(BaseModel):  
    collected\_at: datetime  
    window\_days: int  
    landing\_pages: List\[Dict\[str, Any\]\]   \# sessions, revenue, cvr, etc.  
    conversions: List\[Dict\[str, Any\]\]

class SERPSnapshot(BaseModel):  
    collected\_at: datetime  
    keywords: List\[Dict\[str, Any\]\]        \# keyword, top urls, features, etc.  
    competitors: List\[str\]

class CrawlSnapshot(BaseModel):  
    collected\_at: datetime  
    tool: Literal\["screamingfrog", "sitebulb", "custom"\]  
    pages: List\[Dict\[str, Any\]\]           \# status, title, canon, h1, schema presence, etc.  
    diffs: List\[Dict\[str, Any\]\]           \# changes vs previous crawl

class BacklinkSnapshot(BaseModel):  
    collected\_at: datetime  
    tool: Literal\["ahrefs", "semrush", "majestic", "custom", "none"\]  
    referring\_domains: List\[Dict\[str, Any\]\]  
    anchors: List\[Dict\[str, Any\]\]

class Inputs(BaseModel):  
    gsc: Optional\[GSCSnapshot\] \= None  
    ga4: Optional\[GA4Snapshot\] \= None  
    serp: Optional\[SERPSnapshot\] \= None  
    crawl: Optional\[CrawlSnapshot\] \= None  
    backlinks: Optional\[BacklinkSnapshot\] \= None

\# \---------- Findings & scoring \----------

class Issue(BaseModel):  
    issue\_id: str  
    issue\_type: IssueType  
    market: Market  
    url: Optional\[str\] \= None  
    template: Optional\[str\] \= None  
    severity: int \= Field(3, ge=1, le=5)  
    evidence: Dict\[str, Any\]  
    detected\_at: datetime  
    owner\_hint: OpportunityType         \# which specialist should handle

class Opportunity(BaseModel):  
    opportunity\_id: str  
    opportunity\_type: OpportunityType  
    market: Market  
    primary\_url: str  
    related\_urls: List\[str\] \= Field(default\_factory=list)  
    keyword\_cluster: List\[str\] \= Field(default\_factory=list)  
    issue\_refs: List\[str\] \= Field(default\_factory=list)     \# issue\_ids  
    evidence: Dict\[str, Any\]

    impact: int \= Field(0, ge=0, le=5)  
    confidence: int \= Field(0, ge=0, le=5)  
    effort: int \= Field(1, ge=1, le=5)  
    risk: int \= Field(0, ge=0, le=5)

    priority\_score: float \= 0.0  
    rationale: str

class Findings(BaseModel):  
    issues: List\[Issue\] \= Field(default\_factory=list)  
    opportunities: List\[Opportunity\] \= Field(default\_factory=list)

class Scores(BaseModel):  
    \# sorted list of opportunity\_ids by deterministic order:  
    \# priority\_score desc, impact desc, confidence desc, effort asc, primary\_url asc  
    ranked\_opportunity\_ids: List\[str\] \= Field(default\_factory=list)  
    score\_calculated\_at: Optional\[datetime\] \= None

\# \---------- Plan / ChangeSets \----------

OperationType \= Literal\[  
    "UPDATE\_META", "UPDATE\_H1", "UPDATE\_BODY\_COPY", "ADD\_INTERNAL\_LINKS",  
    "UPDATE\_CANONICAL", "UPDATE\_ROBOTS", "UPDATE\_SCHEMA\_JSONLD",  
    "CREATE\_CONTENT\_BRIEF", "REFRESH\_CONTENT\_BRIEF",  
    "CREATE\_OUTREACH\_LIST"  
\]

class Operation(BaseModel):  
    op\_id: str  
    op\_type: OperationType  
    target: Dict\[str, Any\]                 \# e.g. {"url": "..."} or {"template": "product"}  
    payload: Dict\[str, Any\]                \# actual diffs or brief outline  
    rollback: Dict\[str, Any\]               \# inverse changes or restore tokens  
    qa\_checks: List\[str\]                   \# deterministic checklist items  
    risk: int \= Field(0, ge=0, le=5)

class ChangeSet(BaseModel):  
    changeset\_id: str  
    kind: ChangeSetKind  
    opportunity\_id: str  
    market: Market

    approval\_required: bool  
    approval\_status: ApprovalStatus \= "PENDING"  
    approved\_by: Optional\[str\] \= None  
    approved\_at: Optional\[datetime\] \= None

    execution\_status: ExecutionStatus \= "QUEUED"  
    executed\_at: Optional\[datetime\] \= None

    target: Dict\[str, Any\]                  \# {"url": "..."} or {"template": "..."}  
    operations: List\[Operation\]

    predicted\_impact: Dict\[str, Any\]        \# {"ctr\_lift":0.1,"rank\_lift":2}  
    rollback\_plan: Dict\[str, Any\]  
    notes: str \= ""

class Ticket(BaseModel):  
    ticket\_id: str  
    system: Literal\["jira", "asana", "trello", "github", "none"\]  
    title: str  
    description: str  
    priority: Literal\["P0", "P1", "P2"\]  
    url\_refs: List\[str\]  
    created\_at: datetime

class Plan(BaseModel):  
    \# queue is deterministic: market order, then ranked\_opportunity\_ids order  
    work\_queue: List\[str\] \= Field(default\_factory=list)      \# opportunity\_ids remaining  
    in\_progress: Optional\[str\] \= None                        \# opportunity\_id currently handled  
    changesets: List\[ChangeSet\] \= Field(default\_factory=list)  
    tickets: List\[Ticket\] \= Field(default\_factory=list)  
    plan\_created\_at: Optional\[datetime\] \= None

\# \---------- Approvals / Execution / Validation \----------

class ApprovalDecision(BaseModel):  
    changeset\_id: str  
    decision: ApprovalStatus                 \# APPROVED / REJECTED / PARTIAL  
    reviewer: str  
    comment: Optional\[str\] \= None  
    decided\_at: datetime

class Approvals(BaseModel):  
    required: bool \= False  
    status: ApprovalStatus \= "NOT\_REQUIRED"  
    decisions: List\[ApprovalDecision\] \= Field(default\_factory=list)

class Execution(BaseModel):  
    applied\_changesets: List\[str\] \= Field(default\_factory=list)     \# changeset\_ids  
    failed\_changesets: List\[str\] \= Field(default\_factory=list)  
    skipped\_changesets: List\[str\] \= Field(default\_factory=list)  
    execution\_notes: List\[str\] \= Field(default\_factory=list)

class MetricPoint(BaseModel):  
    ts: datetime  
    metrics: Dict\[str, Any\]

class Validation(BaseModel):  
    pre\_metrics: Dict\[str, Any\] \= Field(default\_factory=dict)  
    post\_metrics: Dict\[str, Any\] \= Field(default\_factory=dict)  
    checks: List\[Dict\[str, Any\]\] \= Field(default\_factory=list)  
    validated\_at: Optional\[datetime\] \= None

class AuditEvent(BaseModel):  
    ts: datetime  
    node: str  
    event: Literal\["START", "END", "TOOL\_CALL", "WARN", "ERROR"\]  
    detail: Dict\[str, Any\]

class ErrorEvent(BaseModel):  
    ts: datetime  
    node: str  
    error\_type: str  
    message: str  
    context: Dict\[str, Any\]

\# \---------- Final State \----------

class AurumSEOState(BaseModel):  
    run: RunMeta  
    config: Config

    inputs: Inputs \= Field(default\_factory=Inputs)  
    findings: Findings \= Field(default\_factory=Findings)  
    scores: Scores \= Field(default\_factory=Scores)  
    plan: Plan \= Field(default\_factory=Plan)  
    approvals: Approvals \= Field(default\_factory=Approvals)  
    execution: Execution \= Field(default\_factory=Execution)  
    validation: Validation \= Field(default\_factory=Validation)

    audit\_log: List\[AuditEvent\] \= Field(default\_factory=list)  
    errors: List\[ErrorEvent\] \= Field(default\_factory=list)

---

## **1.2 Field ownership (who is allowed to write what)**

“Ownership” is enforceable by code review \+ runtime assertions in each node.

| State Path | Owner Node(s) (writers) | Readers |
| ----- | ----- | ----- |
| `inputs.gsc` | `collect_gsc` | all downstream |
| `inputs.ga4` | `collect_ga4` | all downstream |
| `inputs.serp` | `collect_serp_snapshot` | all downstream |
| `inputs.crawl` | `collect_crawl_diff` | all downstream |
| `findings.issues` | `classify_issues` | scoring \+ specialists |
| `findings.opportunities` | `score_opportunities` | router \+ specialists \+ QA |
| `scores.*` | `score_opportunities` | router |
| `plan.work_queue`, `plan.plan_created_at` | `score_opportunities` | router |
| `plan.in_progress` | `route_by_type_and_priority` | specialists |
| `plan.changesets` | `techseo_agent_node`, `productseo_agent_node`, `content_agent_node`, `authority_agent_node` | QA \+ execute |
| `approvals.*` | `qa_guardrail_node`, `approval_interrupt_node` | execute |
| `execution.*` | `execute_changeset_node` | validate |
| `plan.tickets` | `create_tickets_node` | persist |
| `validation.*` | `validate_metrics_node` | persist |
| `audit_log`, `errors` | every node (append-only) | observability |

---

# **2\) LangGraph nodes, routing, and transition rules (exact)**

## **2.1 Exact node names (no placeholders)**

**Observe**

1. `collect_gsc`  
2. `collect_ga4`  
3. `collect_serp_snapshot`  
4. `collect_crawl_diff`

**Diagnose**  
5\. `normalize_inputs`  
6\. `detect_anomalies`  
7\. `classify_issues`

**Recommend**  
8\. `score_opportunities`  
9\. `route_by_type_and_priority`  
10\. `techseo_agent_node`  
11\. `productseo_agent_node`  
12\. `content_agent_node`  
13\. `authority_agent_node`

**Approve**  
14\. `qa_guardrail_node`  
15\. `approval_interrupt_node`

**Execute**  
16\. `execute_changeset_node`  
17\. `create_tickets_node`

**Validate & Learn**  
18\. `validate_metrics_node`  
19\. `persist_run_node`

---

## **2.2 Transition graph (explicit adjacency \+ conditions)**

### **Direct edges**

START  
  \-\> collect\_gsc  
  \-\> collect\_ga4  
  \-\> collect\_serp\_snapshot  
  \-\> collect\_crawl\_diff  
  \-\> normalize\_inputs  
  \-\> detect\_anomalies  
  \-\> classify\_issues  
  \-\> score\_opportunities  
  \-\> route\_by\_type\_and\_priority

### **Conditional edges from `route_by_type_and_priority`**

Let `next_id = plan.work_queue[0]` if exists.

route\_by\_type\_and\_priority:  
  IF plan.work\_queue is empty \-\> qa\_guardrail\_node

  ELSE:  
    look up Opportunity(next\_id).opportunity\_type  
    IF "TECH"      \-\> techseo\_agent\_node  
    IF "PRODUCT"   \-\> productseo\_agent\_node  
    IF "CONTENT"   \-\> content\_agent\_node  
    IF "AUTHORITY" \-\> authority\_agent\_node

### **Loop edges back to router (deterministic batching)**

techseo\_agent\_node      \-\> route\_by\_type\_and\_priority  
productseo\_agent\_node   \-\> route\_by\_type\_and\_priority  
content\_agent\_node      \-\> route\_by\_type\_and\_priority  
authority\_agent\_node    \-\> route\_by\_type\_and\_priority

### **Conditional edges from `qa_guardrail_node`**

Define:

* `R = config.risk.approval_threshold`  
* `requires_approval = any(cs.approval_required for cs in plan.changesets)`  
* `has_high_risk = any(max(op.risk for op in cs.operations) >= R for cs in plan.changesets)`

qa\_guardrail\_node:  
  IF plan.changesets is empty \-\> create\_tickets\_node

  ELSE IF requires\_approval OR has\_high\_risk \-\> approval\_interrupt\_node

  ELSE \-\> execute\_changeset\_node

### **Conditional edges from `approval_interrupt_node`**

Assume `approval_interrupt_node` pauses until `approvals.decisions` is populated externally.

approval\_interrupt\_node:  
  IF approvals.status in \["APPROVED", "PARTIAL"\] \-\> execute\_changeset\_node  
  IF approvals.status \== "REJECTED"              \-\> create\_tickets\_node

### **Post-execution**

execute\_changeset\_node \-\> create\_tickets\_node \-\> validate\_metrics\_node \-\> persist\_run\_node \-\> END

---

# **3\) ChangeSet JSON contracts (schemas \+ per-node state mutations)**

## **3.1 Shared JSON contracts (canonical schemas)**

### **3.1.1 `Opportunity` JSON contract**

{  
  "opportunity\_id": "string(16)",  
  "opportunity\_type": "TECH|PRODUCT|CONTENT|AUTHORITY",  
  "market": "US|EU|AU",  
  "primary\_url": "string(url)",  
  "related\_urls": \["string(url)"\],  
  "keyword\_cluster": \["string"\],  
  "issue\_refs": \["string(16)"\],  
  "evidence": { "any": "json" },  
  "impact": 0,  
  "confidence": 0,  
  "effort": 1,  
  "risk": 0,  
  "priority\_score": 0.0,  
  "rationale": "string"  
}

### **3.1.2 `ChangeSet` JSON contract**

{  
  "changeset\_id": "string(16)",  
  "kind": "TECH\_FIX|PRODUCT\_PDP\_UPDATE|CATEGORY\_UPDATE|META\_UPDATE|INTERNAL\_LINKS\_UPDATE|CONTENT\_BRIEF|CONTENT\_REFRESH|AUTHORITY\_OUTREACH\_PLAN",  
  "opportunity\_id": "string(16)",  
  "market": "US|EU|AU",  
  "approval\_required": true,  
  "approval\_status": "PENDING|APPROVED|REJECTED|PARTIAL|NOT\_REQUIRED",  
  "target": { "url": "string(url)" },  
  "operations": \[  
    {  
      "op\_id": "string(16)",  
      "op\_type": "UPDATE\_META|UPDATE\_H1|UPDATE\_BODY\_COPY|ADD\_INTERNAL\_LINKS|UPDATE\_CANONICAL|UPDATE\_ROBOTS|UPDATE\_SCHEMA\_JSONLD|CREATE\_CONTENT\_BRIEF|REFRESH\_CONTENT\_BRIEF|CREATE\_OUTREACH\_LIST",  
      "target": { "url": "string(url)" },  
      "payload": { "any": "json" },  
      "rollback": { "any": "json" },  
      "qa\_checks": \["string"\],  
      "risk": 0  
    }  
  \],  
  "predicted\_impact": { "any": "json" },  
  "rollback\_plan": { "any": "json" },  
  "notes": "string"  
}

### **3.1.3 State patch contract (every node returns)**

Each node returns a **partial state patch** with only owned fields.

{  
  "audit\_log\_append": \[  
    { "ts": "iso8601", "node": "string", "event": "START|END|TOOL\_CALL|WARN|ERROR", "detail": {} }  
  \],  
  "errors\_append": \[  
    { "ts": "iso8601", "node": "string", "error\_type": "string", "message": "string", "context": {} }  
  \]  
}

---

## **3.2 Node-by-node contracts (inputs, outputs, mutations)**

For each node: **READS**, **WRITES**, and the **exact mutation**.

---

### **Node: `collect_gsc`**

**READS**

* `run.domain`, `config.integrations.gsc_property_url`

**WRITES**

* `inputs.gsc`

**OUTPUT (state patch)**

{  
  "inputs": {  
    "gsc": {  
      "collected\_at": "iso8601",  
      "window\_days": 28,  
      "pages": \[\],  
      "queries": \[\],  
      "index\_coverage": \[\],  
      "sitemap\_status": \[\]  
    }  
  }  
}

---

### **Node: `collect_ga4`**

**READS**

* `config.integrations.ga4_property_id`

**WRITES**

* `inputs.ga4`

**OUTPUT**

{  
  "inputs": {  
    "ga4": {  
      "collected\_at": "iso8601",  
      "window\_days": 28,  
      "landing\_pages": \[\],  
      "conversions": \[\]  
    }  
  }  
}

---

### **Node: `collect_serp_snapshot`**

**READS**

* `inputs.gsc.queries` (top queries), `config.integrations.serp_provider`

**WRITES**

* `inputs.serp`

**OUTPUT**

{  
  "inputs": {  
    "serp": {  
      "collected\_at": "iso8601",  
      "keywords": \[\],  
      "competitors": \[\]  
    }  
  }  
}

---

### **Node: `collect_crawl_diff`**

**READS**

* crawl source config (external), `run.domain`

**WRITES**

* `inputs.crawl`

**OUTPUT**

{  
  "inputs": {  
    "crawl": {  
      "collected\_at": "iso8601",  
      "tool": "screamingfrog",  
      "pages": \[\],  
      "diffs": \[\]  
    }  
  }  
}

---

### **Node: `normalize_inputs`**

**READS**

* `inputs.*`

**WRITES**

* *none of the main sections*; it must only **normalize in-place** by producing a safe canonical structure.  
* To keep ownership strict, we treat this node as generating a **normalized view embedded in evidence** later.  
* So it writes **only** audit info unless you explicitly add `inputs_normalized` to schema.

**OUTPUT**

{  
  "audit\_log\_append": \[  
    { "ts": "iso8601", "node": "normalize\_inputs", "event": "END", "detail": { "normalized": true } }  
  \]  
}

(If you want strict normalized storage, add `inputs_normalized` to state and assign ownership to this node.)

---

### **Node: `detect_anomalies`**

**READS**

* `inputs.gsc`, `inputs.ga4`

**WRITES**

* *none directly*; anomalies get embedded in issues evidence during `classify_issues`.

**OUTPUT**

{  
  "audit\_log\_append": \[  
    { "ts": "iso8601", "node": "detect\_anomalies", "event": "END", "detail": { "anomaly\_scan": "complete" } }  
  \]  
}

---

### **Node: `classify_issues`**

**READS**

* `inputs.*`

**WRITES**

* `findings.issues`

**OUTPUT**

{  
  "findings": {  
    "issues": \[  
      {  
        "issue\_id": "string(16)",  
        "issue\_type": "CTR\_DROP",  
        "market": "US",  
        "url": "https://aurumpickleball.com/collections/paddles",  
        "template": null,  
        "severity": 4,  
        "evidence": { "gsc\_ctr\_change": \-0.23, "impressions\_28d": 84210 },  
        "detected\_at": "iso8601",  
        "owner\_hint": "PRODUCT"  
      }  
    \]  
  }  
}

---

### **Node: `score_opportunities`**

**READS**

* `findings.issues`, `inputs.*`, `config.scoring`, `config.routing.market_order`

**WRITES**

* `findings.opportunities`  
* `scores.ranked_opportunity_ids`  
* `plan.work_queue`  
* `plan.plan_created_at`

**Deterministic ordering**  
Sort by:

1. `priority_score desc`  
2. `impact desc`  
3. `confidence desc`  
4. `effort asc`  
5. `primary_url asc`

Then build `plan.work_queue` by iterating `market_order` and preserving ranked order within each market.

**OUTPUT**

{  
  "findings": { "opportunities": \[ /\* Opportunity\[\] \*/ \] },  
  "scores": {  
    "ranked\_opportunity\_ids": \["op1","op2"\],  
    "score\_calculated\_at": "iso8601"  
  },  
  "plan": {  
    "work\_queue": \["op1","op2"\],  
    "plan\_created\_at": "iso8601"  
  }  
}

---

### **Node: `route_by_type_and_priority`**

**READS**

* `plan.work_queue`, `findings.opportunities`, `config.routing.allow_types`

**WRITES**

* `plan.in_progress`  
* `plan.work_queue` (pop front)

**RULE**

* Pop the next opportunity that matches allowed types.  
* Set `plan.in_progress = next_id`.

**OUTPUT**

{  
  "plan": {  
    "in\_progress": "op1",  
    "work\_queue": \["op2"\]  
  }  
}

---

## **Specialist nodes (generate ChangeSets)**

Each specialist node **must**:

* read `plan.in_progress`  
* fetch the corresponding `Opportunity`  
* append **one or more ChangeSets** to `plan.changesets`  
* clear `plan.in_progress = null` (so router can pick next)

### **Node: `techseo_agent_node`**

**WRITES**

* `plan.changesets` (append)  
* `plan.in_progress` (set null)

**OUTPUT**

{  
  "plan": {  
    "in\_progress": null,  
    "changesets": \[  
      {  
        "changeset\_id": "cs\_tech\_01",  
        "kind": "TECH\_FIX",  
        "opportunity\_id": "op1",  
        "market": "US",  
        "approval\_required": true,  
        "approval\_status": "PENDING",  
        "target": { "url": "https://aurumpickleball.com/sitemap.xml" },  
        "operations": \[  
          {  
            "op\_id": "op\_fix\_schema",  
            "op\_type": "UPDATE\_SCHEMA\_JSONLD",  
            "target": { "url": "https://aurumpickleball.com/products/example" },  
            "payload": { "jsonld\_patch": {} },  
            "rollback": { "restore\_jsonld": {} },  
            "qa\_checks": \["schema\_validates", "no\_price\_claim\_change"\],  
            "risk": 3  
          }  
        \],  
        "predicted\_impact": { "index\_eligibility": "improve" },  
        "rollback\_plan": { "strategy": "restore\_previous\_jsonld" },  
        "notes": "Template-level schema fix"  
      }  
    \]  
  }  
}

### **Node: `productseo_agent_node`**

Same pattern, but kinds are typically:

* `PRODUCT_PDP_UPDATE`, `META_UPDATE`, `INTERNAL_LINKS_UPDATE`

### **Node: `content_agent_node`**

Kinds:

* `CONTENT_BRIEF`, `CONTENT_REFRESH`

### **Node: `authority_agent_node`**

Kinds:

* `AUTHORITY_OUTREACH_PLAN` (no auto-link building)

---

### **Node: `qa_guardrail_node`**

**READS**

* `plan.changesets`, `config.risk`

**WRITES**

* `approvals.required`  
* `approvals.status`  
* per-ChangeSet: set `approval_required`, possibly `approval_status="NOT_REQUIRED"` for low risk

**RULES**

* For each ChangeSet:  
  * If `max(operation.risk) >= approval_threshold` → `approval_required=true`  
  * Else if `max(operation.risk) <= auto_execute_max_risk` → `approval_required=false` and `approval_status="NOT_REQUIRED"`  
* If any `approval_required=true` → `approvals.required=true` and `approvals.status="PENDING"`

**OUTPUT**

{  
  "approvals": { "required": true, "status": "PENDING", "decisions": \[\] },  
  "plan": {  
    "changesets": \[  
      { "changeset\_id": "cs1", "approval\_required": true, "approval\_status": "PENDING" }  
    \]  
  }  
}

---

### **Node: `approval_interrupt_node`**

**Purpose**

* Pause execution until `approvals.decisions` is present (human/system input).

**WRITES**

* `approvals.status`  
* Update each ChangeSet’s `approval_status` based on decisions.

**RULES**

* If all required ChangeSets approved → `approvals.status="APPROVED"`  
* If mix → `"PARTIAL"`  
* If all rejected → `"REJECTED"`

**OUTPUT**

{  
  "approvals": {  
    "status": "PARTIAL",  
    "decisions": \[  
      { "changeset\_id": "cs1", "decision": "APPROVED", "reviewer": "seo\_lead", "decided\_at": "iso8601" }  
    \]  
  },  
  "plan": {  
    "changesets": \[  
      { "changeset\_id": "cs1", "approval\_status": "APPROVED" }  
    \]  
  }  
}

---

### **Node: `execute_changeset_node`**

**READS**

* `plan.changesets`, `approvals.status`, `config.risk`

**WRITES**

* `execution.*`  
* per ChangeSet: `execution_status`, `executed_at`

**EXECUTION RULES**

* Execute only ChangeSets where:  
  * `approval_status in ["APPROVED","NOT_REQUIRED"]`  
  * kind not in `config.risk.blocklist_kinds`

All others are skipped and converted into tickets later.

**OUTPUT**

{  
  "execution": {  
    "applied\_changesets": \["cs1"\],  
    "failed\_changesets": \[\],  
    "skipped\_changesets": \["cs2"\],  
    "execution\_notes": \["Applied cs1 via CMS API"\]  
  },  
  "plan": {  
    "changesets": \[  
      { "changeset\_id": "cs1", "execution\_status": "APPLIED", "executed\_at": "iso8601" },  
      { "changeset\_id": "cs2", "execution\_status": "SKIPPED" }  
    \]  
  }  
}

---

### **Node: `create_tickets_node`**

**READS**

* `plan.changesets` (especially skipped/rejected), `execution.failed_changesets`

**WRITES**

* `plan.tickets` (append)

**RULES**

* Create tickets for:  
  * rejected ChangeSets  
  * skipped ChangeSets  
  * failed ChangeSets

**OUTPUT**

{  
  "plan": {  
    "tickets": \[  
      {  
        "ticket\_id": "t1",  
        "system": "jira",  
        "title": "Fix PDP schema validation errors",  
        "description": "See ChangeSet cs2. Apply JSON-LD patch \+ verify rich results.",  
        "priority": "P0",  
        "url\_refs": \["https://aurumpickleball.com/products/example"\],  
        "created\_at": "iso8601"  
      }  
    \]  
  }  
}

---

### **Node: `validate_metrics_node`**

**READS**

* `inputs.gsc`, `inputs.ga4`, `execution.applied_changesets`

**WRITES**

* `validation.pre_metrics`, `validation.post_metrics`, `validation.checks`, `validation.validated_at`

**RULES**

* Pre metrics are the input window metrics.  
* Post metrics are collected on next scheduled run (or if you have short-window checks, store those too).  
* Always produce explicit checks like:  
  * schema validates (pass/fail)  
  * pages still indexable  
  * no 404 introduced

**OUTPUT**

{  
  "validation": {  
    "pre\_metrics": { "org\_revenue\_28d": 12000 },  
    "post\_metrics": { "org\_revenue\_28d": 12500 },  
    "checks": \[{ "name": "schema\_validates", "passed": true }\],  
    "validated\_at": "iso8601"  
  }  
}

---

### **Node: `persist_run_node`**

**READS**

* entire state

**WRITES**

* none (side effect: DB write)

**OUTPUT**

{  
  "audit\_log\_append": \[  
    { "ts": "iso8601", "node": "persist\_run\_node", "event": "END", "detail": { "persisted": true } }  
  \]  
}

---

# **4\) CrewAI Flow layout (agents, tasks, execution order, routing)**

This Flow mirrors the LangGraph routing exactly, but expressed as **CrewAI agents \+ tasks** with explicit dependencies.

## **4.1 Agents (exact names \+ permissions)**

1. **AurumSEO\_ManagerAgent**  
   * Responsibilities: orchestration \+ routing decisions  
   * Tools: read-only (state store read, logging)  
   * **No write tools**  
2. **AurumSEO\_TechSEOAgent**  
   * Tools: GSC/GA read, crawl read, schema validator  
   * Output: `ChangeSet(kind=TECH_FIX)`  
3. **AurumSEO\_ProductSEOAgent**  
   * Tools: catalog read, CMS draft write (optional gated), internal link planner  
   * Output: PDP/category/meta/internal-links ChangeSets  
4. **AurumSEO\_ContentAgent**  
   * Tools: SERP snapshot read, brief generator, content refresh planner  
   * Output: content briefs / refresh ChangeSets  
5. **AurumSEO\_AuthorityAgent**  
   * Tools: backlink snapshot read  
   * Output: outreach plan ChangeSets (no auto outreach)  
6. **AurumSEO\_QAComplianceAgent**  
   * Tools: schema validator, brand policy rules, risk scoring validator  
   * Output: approval requirement \+ final QA pass/fail flags  
7. **AurumSEO\_ExecutionAgent**  
   * Tools: CMS write (approval gated), ticketing write  
   * Output: execution results \+ tickets

---

## **4.2 Tasks (exact task names \+ inputs/outputs)**

Each task receives the shared `AurumSEOState` as `context` and returns a **state patch**.

### **Task list in order**

1. `Task_CollectGSC`  
2. `Task_CollectGA4`  
3. `Task_CollectSERP`  
4. `Task_CollectCrawlDiff`  
5. `Task_ClassifyIssues`  
6. `Task_ScoreOpportunitiesAndBuildQueue`  
7. `Task_RouteNextOpportunity`  
8. Specialist task (one of):  
   * `Task_GenerateTechChangeSet`  
   * `Task_GenerateProductChangeSet`  
   * `Task_GenerateContentChangeSet`  
   * `Task_GenerateAuthorityChangeSet`  
9. Loop back to `Task_RouteNextOpportunity` until queue empty  
10. `Task_QAGuardrail`  
11. `Task_ApprovalGate`  
12. `Task_ExecuteApprovedChangeSets`  
13. `Task_CreateTickets`  
14. `Task_ValidateMetrics`  
15. `Task_PersistRun`

---

## **4.3 Flow routing rules (explicit)**

### **Flow control conditions**

* `queue_empty = len(plan.work_queue) == 0`  
* `next_type = Opportunity(plan.in_progress).opportunity_type`  
* `needs_approval = approvals.required is True and approvals.status == "PENDING"`

### **Deterministic specialist routing**

Task\_RouteNextOpportunity:  
  IF queue\_empty \-\> Task\_QAGuardrail  
  ELSE:  
    set plan.in\_progress \= pop(plan.work\_queue\[0\])  
    route by opportunity\_type:  
      TECH      \-\> Task\_GenerateTechChangeSet  
      PRODUCT   \-\> Task\_GenerateProductChangeSet  
      CONTENT   \-\> Task\_GenerateContentChangeSet  
      AUTHORITY \-\> Task\_GenerateAuthorityChangeSet  
  After specialist \-\> Task\_RouteNextOpportunity

### **Approval routing**

Task\_QAGuardrail:  
  IF plan.changesets is empty \-\> Task\_CreateTickets  
  ELSE IF approvals.required \-\> Task\_ApprovalGate  
  ELSE \-\> Task\_ExecuteApprovedChangeSets

Task\_ApprovalGate:  
  IF approvals.status in \["APPROVED","PARTIAL"\] \-\> Task\_ExecuteApprovedChangeSets  
  IF approvals.status \== "REJECTED"             \-\> Task\_CreateTickets

---

## **4.4 CrewAI Flow skeleton (engineer-ready outline)**

\# Pseudocode structure that mirrors the routing exactly (names are exact)

from crewai.flow import Flow, start, listen

class AurumSEO\_Flow(Flow):

    @start()  
    def Task\_CollectGSC(self, state): ...  
      
    @listen(Task\_CollectGSC)  
    def Task\_CollectGA4(self, state): ...

    @listen(Task\_CollectGA4)  
    def Task\_CollectSERP(self, state): ...

    @listen(Task\_CollectSERP)  
    def Task\_CollectCrawlDiff(self, state): ...

    @listen(Task\_CollectCrawlDiff)  
    def Task\_ClassifyIssues(self, state): ...

    @listen(Task\_ClassifyIssues)  
    def Task\_ScoreOpportunitiesAndBuildQueue(self, state): ...

    @listen(Task\_ScoreOpportunitiesAndBuildQueue)  
    def Task\_RouteNextOpportunity(self, state): ...  
    \# inside: deterministic routing \+ calls to specialist tasks

(You’ll implement the loop by calling the next task method based on the explicit conditions above.)

---

# **5\) Implementation notes engineers actually care about**

## **5.1 Strict determinism checklist**

* Always sort opportunities using the exact order defined in `score_opportunities`  
* Only mutate owned fields  
* Use stable IDs (`sha256`) to prevent duplicates across runs  
* Convert “non-executed” items into tickets **every run** (no silent drops)

## **5.2 No secrets in state**

The schema holds **integration references only**, not tokens/keys. Credentials remain in your runtime secret manager.

---

If you want, I can also generate:

* the **LangGraph `StateGraph` builder code** (`add_node`, `add_edge`, `add_conditional_edges`) using these exact node names, and  
* a **JSON Schema file** for `ChangeSet`, `Opportunity`, and node patches so CI can validate every run.

