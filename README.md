Absolutely — below is a complete, GitHub-ready README tailored to the project you shared, including architecture, workflow, setup, CSV schema, compliance guardrails, execution flow, sample output, and troubleshooting.

 # AI Revenue Recovery Engine

 A **multi-agent, compliance-aware revenue recovery workflow** built with **LangGraph, LangChain, OpenAI, and Google Colab**.

 The system processes overdue payment records from a CSV file, diagnoses payment failures, generates personalized Hinglish WhatsApp outreach messages, simulates recovery execution, and applies deterministic compliance guardrails before allowing additional customer outreach.

 The workflow is designed around a simple principle:

 > **AI can recommend and execute bounded recovery actions, but deterministic guardrails always control when automation must stop or escalate to a human.**

---

 ## Table of Contents

 - Overview
- Key Features
- Architecture
- Workflow
- Agent Responsibilities
- Compliance Guardrails
- State Model
- Project Structure
- Requirements
- Installation
- OpenAI API Key Setup
- Input CSV Format
- Running the Project
- Execution Flow Example
- Output JSON
- Batch Analytics
- Sample Output
- Recovery Simulation Logic
- Audit Trail
- Important Implementation Notes
- Troubleshooting
- Security Considerations
- Limitations
- Future Enhancements
- Disclaimer
- License

---

 ## Overview

 The **AI Revenue Recovery Engine** is an asynchronous, graph-based workflow for handling overdue payment recovery scenarios.

 Instead of allowing an LLM to independently decide whether and how often a customer should be contacted, the project separates responsibilities into specialized components:

 1. **Diagnosis Agent** verifies the payment failure context.
2. **Strategist Agent** uses an LLM to generate a personalized Hinglish recovery message.
3. **Execution Agent** simulates the recovery action and determines whether payment was recovered.
4. **Compliance Guardrail** deterministically evaluates whether the workflow can continue.
5. **Human Escalation** handles accounts that exceed predefined automation boundaries.
6. **Compliance Stop** permanently stops automated outreach after the configured attempt threshold.
7. **Batch Processor** processes multiple customer records and generates aggregate recovery metrics.

 The resulting workflow provides both:

 - **Customer-level results**
- **Batch-level recovery analytics**

---

 # Key Features

 ## Multi-Agent Architecture

 The project separates recovery responsibilities across multiple workflow nodes rather than putting all business logic into a single LLM call.

 ## LangGraph State Machine

 The entire recovery lifecycle is represented as a directed state graph.

 This makes the workflow:

 - Explicit
- Observable
- Deterministic at routing boundaries
- Easier to extend
- Easier to audit

 ## AI-Powered Hinglish Outreach

 The Strategist Agent uses an OpenAI chat model to generate natural Hinglish customer communication.

 The prompt instructs the model to:

 - Remain polite
- Avoid aggressive collection language
- Avoid threats
- Explain the payment failure reason
- Return structured JSON

 ## Deterministic Compliance Layer

 The LLM does **not** decide whether another outreach attempt is allowed.

 Instead, a deterministic router evaluates:

 - Debt age
- Amount due
- Number of outreach attempts
- Current recovery status

 This provides a bounded automation layer around the AI component.

 ## Human Escalation

 Accounts that exceed configured debt-age or amount thresholds are routed to a human collections workflow.

 ## Harassment Prevention

 The workflow stops automated outreach once the maximum configured number of attempts has been reached.

 ## Audit Trail

 Every major action is recorded in the customer's `audit_trail`.

 This makes it possible to understand:

 - Which agent acted
- What it did
- What message was generated
- Whether an attempt succeeded
- Why automation stopped
- Why a customer was escalated

 ## Batch-Level Analytics

 The pipeline calculates:

 - Total amount at risk
- Total recovered
- Cash remaining at risk
- Recovery yield rate
- Compliance halts
- Human interventions

---

 # Architecture

 The high-level architecture is:

```
                    ┌─────────────────────┐
                    │      CSV Input      │
                    │ Customer Accounts   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Diagnosis Agent    │
                    │ Verify failure      │
                    │ context             │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Strategist Agent    │
                    │ OpenAI LLM           │
                    │ Hinglish outreach   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Execution Agent    │
                    │ Recovery simulation │
                    └──────────┬──────────┘
                               │
                               ▼
                  ┌──────────────────────────┐
                  │ Compliance Guardrail     │
                  │ Deterministic Router     │
                  └────────────┬─────────────┘
                               │
             ┌─────────────────┼─────────────────┐
             │                 │                 │
             ▼                 ▼                 ▼
       ┌───────────┐    ┌──────────────┐   ┌──────────────┐
       │ Recovered │    │ Human        │   │ Compliance   │
       │           │    │ Escalation   │   │ Stop         │
       └───────────┘    └──────────────┘   └──────────────┘
             │                 │                 │
             └─────────────────┴─────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Batch Analytics     │
                    │ + Audit JSON        │
                    └─────────────────────┘
```

---

 # Workflow

 The workflow is implemented using a LangGraph `StateGraph`.

 The graph starts with:

```
DiagnosisAgent
```

 and proceeds to:

```
StrategistAgent
        ↓
ExecutionAgent
        ↓
ComplianceGuardrail
```

 The guardrail can then route the workflow to one of four outcomes:

```
RECOVERED
    ↓
MarkRecovered
    ↓
END
```

 or:

```
ESCALATED_HUMAN
    ↓
HumanEscalation
    ↓
END
```

 or:

```
STOPPED_COMPLIANCE
    ↓
ComplianceStop
    ↓
END
```

 or back to:

```
StrategistAgent
    ↓
ExecutionAgent
```

 for another bounded attempt.

---

 # Agent Responsibilities

 ## 1\. DiagnosisAgent

 The Diagnosis Agent verifies the basic payment failure context.

 Currently, this node performs a lightweight deterministic diagnosis and adds an audit record.

 Example audit entry:

```
{
    "node": "DiagnosisAgent",
    "action": "Verified issue 'user_dropped_at_otp' for channel 'upi'."
}
```

 ### Current responsibility

 - Record the payment failure reason
- Record the payment channel
- Add the diagnosis to the audit trail

 ### Future responsibility

 This agent could be extended to:

 - Classify failure reasons
- Detect invalid payment states
- Identify retry eligibility
- Determine whether additional customer information is required
- Normalize gateway error codes

---

 # 2\. StrategistAgent

 The Strategist Agent generates the customer-facing recovery message.

 It uses:

```
ChatOpenAI(
    model="gpt-4o",
    temperature=0.3
)
```

 The prompt instructs the model to produce a natural Hinglish WhatsApp message.

 The generated response is expected to have the structure:

```
{
    "generated_outreach_message": "..."
}
```

 The message is then stored in:

```
state["current_strategy"]
```

 ### Example

 For an OTP abandonment scenario, the fallback message may look like:

```
Hi Rahul, aapka ₹2,500 ka payment checkout OTP page par drop ho gaya tha.
Aap is safe link par click karke direct secure check-out complete kar sakte hain.
```

 The actual LLM-generated message may differ.

---

 # 3\. ExecutionAgent

 The Execution Agent represents the recovery execution layer.

 In the current project, this is a **deterministic simulation** rather than a real payment collection system.

 The agent:

 - Increments the outreach count
- Determines whether the simulated recovery succeeds
- Updates the customer status
- Adds the result to the audit trail

 A successful attempt changes the state to:

```
RECOVERED
```

 Otherwise, the workflow continues to the compliance guardrail.

---

 # 4\. Compliance Guardrail

 The compliance router is one of the most important parts of the architecture.

 It is deliberately deterministic and does not rely on an LLM.

```
def compliance_guardrail_router(state):
```

 The router evaluates the current state after each execution attempt.

---

 # Compliance Guardrails

 The current implementation has three primary rules.

 ## Rule 1 — Stop if Recovered

 If:

```
state["status"] == "RECOVERED"
```

 the workflow routes to:

```
MarkRecovered
```

 and terminates.

 No additional outreach is performed.

---

 ## Rule 2 — Human Escalation for Large or Aged Accounts

 The workflow escalates if either condition is true:

```
state["days_overdue"] > 60
```

 OR:

```
state["amount_due"] >= 50000
```

 The account is routed to:

```
HumanEscalation
```

 and receives the final status:

```
ESCALATED_HUMAN
```

 This prevents the automated recovery engine from continuing indefinitely on accounts outside the defined automation boundary.

---

 ## Rule 3 — Maximum Outreach Attempts

 If:

```
state["outreach_count"] >= 3
```

 the workflow routes to:

```
ComplianceStop
```

 and sets:

```
STOPPED_COMPLIANCE
```

 This creates a bounded maximum outreach policy.

---

 # State Model

 Every customer is represented using the `AgentState` TypedDict.

```
class AgentState(TypedDict):
    customer_id: str
    customer_name: str
    amount_due: float
    days_overdue: int
    failure_reason: str
    payment_channel: str
    outreach_count: int
    historical_context: str
    current_strategy: str
    status: str
    audit_trail: List[Dict[str, Any]]
```

 ## State Fields

 | Field | Description |
| --- | --- |
| `customer_id` | Unique customer/account identifier |
| `customer_name` | Customer name |
| `amount_due` | Outstanding amount |
| `days_overdue` | Number of days payment is overdue |
| `failure_reason` | Payment failure reason |
| `payment_channel` | Payment method/channel |
| `outreach_count` | Number of recovery attempts |
| `historical_context` | Relevant historical information |
| `current_strategy` | Latest generated recovery message |
| `status` | Current workflow state |
| `audit_trail` | Historical workflow actions |

---

 # Status Values

 The workflow uses the following primary status values:

```
ACTIVE
RECOVERED
STOPPED_COMPLIANCE
ESCALATED_HUMAN
```

 ### ACTIVE

 The account is still being processed.

 ### RECOVERED

 The simulated recovery succeeded.

 ### STOPPED\_COMPLIANCE

 The automated workflow reached its configured outreach boundary.

 ### ESCALATED\_HUMAN

 The account exceeded the configured automation boundary and requires human intervention.

---

 # Project Structure

 A recommended project structure is:

```
ai-revenue-recovery-engine/
│
├── README.md
├── recovery_engine.py
├── input/
│   └── customers.csv
│
├── output/
│   └── recovery_results.json
│
├── requirements.txt
│
└── .gitignore
```

 For Google Colab, the complete workflow can also be maintained inside a single notebook:

```
revenue_recovery_engine.ipynb
```

---

 # Requirements

 The project requires Python 3.9+.

 Primary dependencies include:

```
langgraph
langchain
langchain-core
langchain-openai
typing-extensions
```

 Google Colab additionally provides:

```
google.colab.userdata
```

 for accessing Secrets Manager values.

---

 # Installation

 Install the required packages in Google Colab:

```
pip install -U langgraph langchain langchain-core langchain-openai typing-extensions
```

 If using a local Python environment:

```
python -m pip install -U langgraph langchain langchain-core langchain-openai typing-extensions
```

---

 # OpenAI API Key Setup

 The project expects an OpenAI API key to be stored securely rather than hard-coded into the notebook.

 The current implementation retrieves the key from Google Colab Secrets:

```
from google.colab import userdata

os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY")
```

 ## Google Colab Setup

 1. Open the notebook in Google Colab.
2. Open the **Secrets** panel.
3. Add a secret named:

```
OPENAI_API_KEY
```

 4. Paste your OpenAI API key as the secret value.
5. Make the secret available to the notebook.
6. Run the initialization cell.

 The application should display:

```
🔑 OpenAI API Key verified securely via Colab Secrets.
```

 If the key is unavailable, the project falls back to predefined code-level message generation.

---

 # Input CSV Format

 The batch processor expects a CSV containing the following columns:

```
customer_id
customer_name
amount_due
days_overdue
failure_reason
payment_channel
outreach_count
historical_context
```

 ## Example

```
customer_id,customer_name,amount_due,days_overdue,failure_reason,payment_channel,outreach_count,historical_context
CUST001,Rahul,2500,5,user_dropped_at_otp,upi,0,First failed payment
CUST002,Priya,12000,15,bank_mandate_failed,bank_mandate,1,Previous retry unsuccessful
CUST003,Amit,75000,20,insufficient_balance,upi,0,High value account
CUST004,Neha,3200,70,card_payment_failed,card,1,Long overdue account
```

---

 # Input Field Details

 ## `customer_id`

 A unique identifier for the customer.

 Example:

```
CUST001
```

 ## `customer_name`

 Name used for personalized communication.

 Example:

```
Rahul
```

 ## `amount_due`

 Outstanding amount.

 Example:

```
2500
```

 The value is converted into a Python `float`.

 ## `days_overdue`

 Number of days since payment became overdue.

 Example:

```
15
```

 The value is converted into an integer.

 ## `failure_reason`

 The reason associated with the failed payment.

 Examples:

```
user_dropped_at_otp
bank_mandate_failed
insufficient_balance
card_payment_failed
```

 ## `payment_channel`

 Payment channel involved in the failure.

 Examples:

```
upi
card
bank_mandate
netbanking
```

 ## `outreach_count`

 Number of previous outreach attempts.

 Example:

```
1
```

 ## `historical_context`

 Free-text historical information provided to the Strategist Agent.

 Example:

```
Customer previously attempted payment twice.
```

---

 # Running the Project

 After loading the input CSV, specify the input and output files:

```
OUTPUT_FILE = "customers.csv"
OUTPUT_JSON = "recovery_results.json"
```

 Reset the batch metrics:

```
reset_batch_analytics()
```

 Then execute:

```
results = await process_recovery_batch(
    OUTPUT_FILE,
    OUTPUT_JSON
)
```

 The system will:

 1. Read the CSV.
2. Create an `AgentState` for each customer.
3. Add the customer's amount to total at-risk value.
4. Execute the LangGraph workflow.
5. Record the final status.
6. Update batch analytics.
7. Save customer-level results.
8. Write the final JSON output.
9. Print a dashboard to the console.

---

 # Execution Flow Example

 Consider:

```
Customer: Rahul
Amount Due: ₹2,500
Days Overdue: 5
Failure Reason: user_dropped_at_otp
Outreach Count: 0
```

 The graph executes approximately as follows:

```
START
  │
  ▼
DiagnosisAgent
  │
  ▼
StrategistAgent
  │
  │ Generates Hinglish message
  ▼
ExecutionAgent
  │
  ├── Success
  │      │
  │      ▼
  │   RECOVERED
  │      │
  │      ▼
  │  MarkRecovered
  │      │
  │      ▼
  │     END
  │
  └── Failure
         │
         ▼
  ComplianceGuardrail
         │
         ├── More automation allowed
         │       │
         │       ▼
         │   StrategistAgent
         │
         ├── Amount/debt age threshold
         │       │
         │       ▼
         │   HumanEscalation
         │
         └── Attempt threshold
                 │
                 ▼
          ComplianceStop
```

---

 # Output JSON

 The pipeline produces:

```
recovery_results.json
```

 The output contains two primary sections:

```
{
    "generated_at": "...",
    "batch_metrics": {},
    "customer_results": []
}
```

---

 # Batch Analytics

 The following metrics are generated.

 ## Total At Risk

 The sum of all customer balances processed during the batch.

```
total_at_risk
```

 ## Total Recovered

 The sum of balances associated with customers whose final status is:

```
RECOVERED
```

```
total_recovered
```

 ## Cash Left At Risk

 Calculated as:

```
total_at_risk - total_recovered
```

 ## Recovery Yield Rate

 Calculated as:

```
(total_recovered / total_at_risk) * 100
```

 provided that `total_at_risk` is greater than zero.

 ## Compliance Halts

 Number of customers whose workflow ended with:

```
STOPPED_COMPLIANCE
```

 ## Human Interventions

 Number of customers whose workflow ended with:

```
ESCALATED_HUMAN
```

---

 # Sample Output

 A simplified output may look like:

```
{
    "generated_at": "2026-09-05T22:30:00",
    "batch_metrics": {
        "total_at_risk": 82700.0,
        "total_recovered": 2500.0,
        "cash_left_at_risk": 80200.0,
        "recovery_yield_rate": 3.025,
        "compliance_halts": 1,
        "human_interventions": 2
    },
    "customer_results": [
        {
            "customer_id": "CUST001",
            "customer_name": "Rahul",
            "amount_due": 2500.0,
            "days_overdue": 5,
            "failure_reason": "user_dropped_at_otp",
            "payment_channel": "upi",
            "outreach_count": 1,
            "historical_context": "First failed payment",
            "current_strategy": "Hi Rahul, ...",
            "status": "RECOVERED",
            "audit_trail": [
                {
                    "node": "DiagnosisAgent",
                    "action": "Verified issue 'user_dropped_at_otp' for channel 'upi'."
                },
                {
                    "node": "StrategistAgent",
                    "action": "LLM Generated Hinglish Alert: \"Hi Rahul, ...\""
                },
                {
                    "node": "ExecutionAgent",
                    "action": "SUCCESS! Payment clear via active recovery engine workflow allocation."
                }
            ]
        }
    ]
}
```

---

 # Audit Trail

 Every customer contains an `audit_trail`.

 Example:

```
[
    {
        "node": "DiagnosisAgent",
        "action": "Verified issue 'user_dropped_at_otp' for channel 'upi'."
    },
    {
        "node": "StrategistAgent",
        "action": "LLM Generated Hinglish Alert: \"Hi Rahul, ...\""
    },
    {
        "node": "ExecutionAgent",
        "action": "SUCCESS! Payment clear via active recovery engine workflow allocation."
    }
]
```

 For a compliance stop, the trail may contain:

```
{
    "node": "ComplianceGuardrail",
    "action": "SAFETY STOP: Bounded execution threshold reached (Max 3 attempts). Silencing automation to prevent harassment."
}
```

 For human escalation:

```
{
    "node": "ComplianceGuardrail",
    "action": "CRITICAL: Transferred to Human Collections Panel. Debt age/amount breached automation boundaries."
}
```

 This audit structure can later be used for:

 - Operational monitoring
- Compliance review
- Debugging
- Agent evaluation
- Customer dispute investigation
- Analytics

---

 # Recovery Simulation Logic

 The current Execution Agent does **not** initiate an actual payment transaction.

 Instead, it uses a deterministic simulation based on the numeric portion of the customer ID:

```
id_numeric = "".join(filter(str.isdigit, state["customer_id"]))
seed_factor = int(id_numeric) if id_numeric else 0

is_successful = (seed_factor % 10) < 4
```

 Therefore, customer IDs ending in:

```
0
1
2
3
```

 will satisfy the simulated success condition.

 For example:

```
CUST001 → success
CUST002 → success
CUST003 → success
CUST004 → failure
```

 This mechanism provides repeatable behavior during demos and testing.

 It should **not** be used as a real-world payment prediction mechanism.

---

 # Important Implementation Notes

 ## 1\. The Current Execution Agent Is a Simulation

 The project currently does not connect to a real payment gateway or payment status API.

 The following statement:

```
SUCCESS! Payment clear via active recovery engine workflow allocation.
```

 represents a simulated result.

 A production system should verify recovery using an authoritative payment transaction/status source.

---

 ## 2\. The Compliance Layer Is Deterministic

 This is intentional.

 The LLM can generate communication, but it does not control the compliance boundary.

 The router decides whether automation may continue.

 This separation is important because probabilistic LLM output should not be responsible for enforcing hard operational limits.

---

 ## 3\. The LLM Output Is Parsed as JSON

 The Strategist Agent expects:

```
{
    "generated_outreach_message": "..."
}
```

 If parsing or API invocation fails, the code falls back to predefined messages.

---

 ## 4\. Batch Analytics Are Global

 The project uses:

```
BATCH_ANALYTICS = {...}
```

 as a global dictionary.

 Therefore, callers should invoke:

```
reset_batch_analytics()
```

 before starting a new independent batch.

 Otherwise, metrics from previous runs can carry over.

 A production implementation should preferably make analytics batch-scoped rather than global.

---

 # Troubleshooting

 ## `OPENAI_API_KEY` Not Found

 You may see:

```
⚠️ Warning: 'OPENAI_API_KEY' not found in Secrets.
```

 Check that:

 - The Colab secret is named exactly `OPENAI_API_KEY`.
- The secret is available to the notebook.
- The notebook has permission to access the secret.

 The application contains fallback messaging, so execution may continue without the API.

---

 ## JSON Parsing Error

 If the model returns malformed JSON, the Strategist Agent falls back to a predefined message.

 For production use, consider:

 - Structured output support
- Schema validation
- Retry logic
- More explicit model output constraints

---

 ## Input CSV File Not Found

 If you receive:

```
FileNotFoundError
```

 verify:

```
OUTPUT_FILE = "customers.csv"
```

 matches the actual file location.

 For Colab, you can upload the CSV using the Colab file interface or mount Google Drive.

---

 ## Incorrect Batch Metrics

 Make sure to call:

```
reset_batch_analytics()
```

 before every independent batch.

---

 # Security Considerations

 ## Never Hard-Code API Keys

 Do not commit:

```
OPENAI_API_KEY = "sk-..."
```

 to source control.

 Use a secrets manager or environment variable.

 ## Protect Customer Data

 Customer records may contain sensitive financial and personally identifiable information.

 Production deployments should implement:

 - Encryption at rest
- Encryption in transit
- Access controls
- Data minimization
- Secure logging
- Retention policies
- Role-based access
- Audit controls

 ## Avoid Sensitive Data in Logs

 The current audit trail stores generated customer outreach messages.

 In a production environment, consider whether storing complete customer-facing messages in logs is necessary.

 Sensitive information should be redacted where appropriate.

---

 # Limitations

 This project is currently a **prototype/demo architecture**, not a production collections system.

 Important limitations include:

 - Payment execution is simulated.
- Recovery success is determined using a deterministic customer-ID rule.
- Compliance thresholds are illustrative configuration values.
- No real payment gateway verification is performed.
- No actual WhatsApp delivery integration exists.
- No customer response handling exists.
- No authentication/authorization layer is included.
- No persistent database is used.
- Batch analytics are maintained in process memory.
- No retry/backoff strategy for external API failures is implemented.
- LLM-generated messages require additional production validation.
- Regulatory requirements vary by jurisdiction and use case.

---

 # Future Enhancements

 ## Real Payment Verification

 Replace the simulated Execution Agent with a payment-status integration.

 For example:

```
ExecutionAgent
      ↓
Payment Gateway
      ↓
Transaction Status
      ↓
Recovery Result
```

 The execution result should be based on authoritative transaction data rather than a simulated success rule.

---

 ## Real WhatsApp Integration

 The Strategist Agent could generate the message while a separate execution service handles delivery.

```
StrategistAgent
      ↓
Message Validation
      ↓
WhatsApp Provider
      ↓
Customer
```

 The system should also track:

 - Sent
- Delivered
- Read
- Failed
- Customer replied

---

 ## Customer Response Agent

 A future conversational agent could classify customer responses:

```
Customer Response
        │
        ├── Wants payment link
        ├── Payment already completed
        ├── Financial difficulty
        ├── Dispute
        ├── Wrong account
        └── Human assistance requested
```

 The response classifier should still operate inside deterministic business and compliance boundaries.

---

 ## Better Failure Classification

 The Diagnosis Agent could normalize gateway failures into categories:

```
OTP_ABANDONMENT
INSUFFICIENT_FUNDS
BANK_DECLINE
CARD_DECLINE
MANDATE_FAILURE
NETWORK_FAILURE
CUSTOMER_CANCELLED
UNKNOWN
```

 This would allow the Strategist Agent to produce more targeted communication.

---

 ## Persistent Database

 Instead of relying entirely on CSV and JSON, production deployments could use a database for:

 - Customer records
- Recovery attempts
- Message history
- Payment events
- Compliance decisions
- Agent traces
- Metrics

---

 ## Human-in-the-Loop Dashboard

 A dashboard could display:

```
┌────────────────────────────────────────────┐
│       REVENUE RECOVERY DASHBOARD           │
├────────────────────────────────────────────┤
│ Total At Risk        ₹XX,XXX               │
│ Total Recovered     ₹XX,XXX               │
│ Recovery Rate        XX.XX%                │
│ Human Escalations    XX                    │
│ Compliance Stops     XX                    │
├────────────────────────────────────────────┤
│ Accounts Requiring Human Attention         │
│                                            │
│ CUST001   ₹75,000   65 days   ESCALATED   │
│ CUST002   ₹55,000   12 days   ESCALATED   │
└────────────────────────────────────────────┘
```

---

 # Recommended Production Architecture

 A production version could evolve into:

```
                    ┌───────────────────┐
                    │ Customer / Event  │
                    │ Data Sources      │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ Diagnosis Agent   │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ Policy Engine     │
                    │ + Guardrails      │
                    └─────────┬─────────┘
                              │
                       Automation Allowed?
                              │
                    ┌─────────┴─────────┐
                    │                   │
                   YES                  NO
                    │                   │
                    ▼                   ▼
          ┌──────────────────┐   ┌───────────────┐
          │ Strategist Agent │   │ Human Review  │
          └────────┬─────────┘   └───────────────┘
                   │
                   ▼
          ┌──────────────────┐
          │ Message Validator│
          └────────┬─────────┘
                   │
                   ▼
          ┌──────────────────┐
          │ Communication    │
          │ Provider         │
          └────────┬─────────┘
                   │
                   ▼
             Customer
                   │
                   ▼
          Payment / Response
                   │
                   ▼
          ┌──────────────────┐
          │ Outcome Verifier │
          └────────┬─────────┘
                   │
                   ▼
          ┌──────────────────┐
          │ Audit + Analytics│
          └──────────────────┘
```

 The key architectural principle remains:

```
LLM = Recommendation / Generation
Policy Engine = Enforcement
Payment System = Source of Truth
Human = Escalation Boundary
Audit Layer = Accountability
```

---

 # Design Principles

 This project follows several important principles.

 ## Bounded Autonomy

 The AI is not given unlimited control over customer interactions.

 Automation operates within explicit boundaries.

 ## Deterministic Safety

 Hard business and compliance rules are implemented as deterministic code rather than relying on model behavior.

 ## Separation of Concerns

 Diagnosis, strategy, execution, routing, and escalation are separate workflow components.

 ## Observability

 Every important workflow action is recorded.

 ## Graceful Degradation

 If the LLM is unavailable, the system has predefined fallback messaging.

 ## Repeatability

 The simulated execution logic is deterministic, making demonstrations and testing reproducible.

---

 # Testing Recommendations

 Before connecting the workflow to real customer systems, test at least the following scenarios.

 ### Normal Successful Recovery

```
Amount: ₹2,500
Days overdue: 5
Outreach count: 0
```

 Expected:

```
RECOVERED
```

 when the deterministic simulation succeeds.

 ### Large Amount

```
Amount: ₹50,000+
```

 Expected:

```
ESCALATED_HUMAN
```

 ### Long Overdue

```
Days overdue: 61+
```

 Expected:

```
ESCALATED_HUMAN
```

 ### Maximum Outreach

```
Outreach count: 3+
```

 Expected:

```
STOPPED_COMPLIANCE
```

 ### LLM Failure

 Disable or invalidate the API configuration.

 Expected behavior:

```
Fallback message generated
```

 rather than complete workflow failure.

---

 # Example Metrics Dashboard

 After processing a batch, the application prints:

```
==================== BATCH METRICS REPORT CARD ====================

🟢 Total Recovered Cash  : ₹XX,XXX.00
🔴 Total Cash Left At Risk: ₹XX,XXX.00
📈 Recovery Yield Rate    : XX.XX%
🛑 Compliance Halts       : X accounts
🧑 Human Interventions    : X accounts

===================================================================
```

 These metrics provide a simple executive-level view of the batch.

---

 # Why LangGraph?

 LangGraph is particularly useful for this use case because revenue recovery is not simply a linear LLM pipeline.

 The workflow contains:

 - State
- Conditional routing
- Loops
- Termination conditions
- Human escalation
- Compliance boundaries
- Auditability

 A conventional chain might look like:

```
Input → LLM → Output
```

 while this system requires:

```
Input
  ↓
Diagnosis
  ↓
Strategy
  ↓
Execution
  ↓
Policy Decision
  ├── Retry
  ├── Recover
  ├── Escalate
  └── Stop
```

 A graph-based architecture maps naturally to these requirements.

---

 # Technology Stack

 | Technology | Purpose |
| --- | --- |
| Python | Core implementation |
| LangGraph | Stateful workflow orchestration |
| LangChain | LLM integration |
| OpenAI | AI-generated customer communication |
| Google Colab | Development/runtime environment |
| CSV | Input data source |
| JSON | Output/audit format |
| asyncio | Asynchronous workflow execution |
| TypedDict | State schema |

---

 # Quick Start

 The shortest path to run the project is:

```
pip install -U langgraph langchain langchain-core langchain-openai typing-extensions
```

 Configure the Colab secret:

```
OPENAI_API_KEY
```

 Prepare:

```
customers.csv
```

 Then run:

```
reset_batch_analytics()

results = await process_recovery_batch(
    "customers.csv",
    "recovery_results.json"
)
```

 The resulting file will be:

```
recovery_results.json
```

---

 # Disclaimer

 This repository is intended for **educational, prototyping, and workflow-engineering purposes**.

 The compliance thresholds and recovery policies in this example are illustrative and should **not** be interpreted as legal, regulatory, financial, or collections guidance.

 Before deploying a revenue recovery system against real customers, the workflow should be reviewed by the appropriate legal, compliance, security, risk, and payments teams.

 In particular, production implementations should independently validate applicable requirements concerning customer communications, payment recovery, consent, frequency of contact, data protection, financial services regulation, and human escalation.

---

 # License

 Add the license appropriate for your project.

 For example:

```
MIT License
```

 If this project is intended to be open source, create a `LICENSE` file containing the complete license text.

---

 # Author

 **AI Revenue Recovery Engine**

 Built as a demonstration of:

 - Multi-agent orchestration
- Stateful AI workflows
- LLM-powered customer communication
- Deterministic compliance guardrails
- Human-in-the-loop escalation
- Batch recovery analytics
- Auditable AI execution

---

 ## Project Summary

 At its core, this project demonstrates a **bounded AI agent architecture for revenue recovery**:

```
             ┌─────────────────────────┐
             │      Customer Data      │
             └────────────┬────────────┘
                          ↓
             ┌─────────────────────────┐
             │    Diagnosis Agent      │
             └────────────┬────────────┘
                          ↓
             ┌─────────────────────────┐
             │   Strategist Agent      │
             │      (OpenAI LLM)       │
             └────────────┬────────────┘
                          ↓
             ┌─────────────────────────┐
             │    Execution Agent      │
             └────────────┬────────────┘
                          ↓
             ┌─────────────────────────┐
             │  Compliance Guardrail   │
             └────────────┬────────────┘
                          ↓
           ┌──────────────┼──────────────┐
           ↓              ↓              ↓
       RECOVERED       ESCALATED       STOPPED
                       HUMAN         COMPLIANCE
           └──────────────┬──────────────┘
                          ↓
             ┌─────────────────────────┐
             │ Audit + Batch Analytics │
             └─────────────────────────┘
```

 The most important design choice is that **the LLM generates strategy, while deterministic policy code controls the boundaries of automation**. This makes the architecture substantially easier to reason about, test, audit, and extend.

 One code issue to fix before running your notebook: the final call uses `OUTPUT_FILE`, but the snippet never defines it. Define it first, for example `OUTPUT_FILE = "customers.csv"`. Also, if you intend the README to describe a production-ready system, I’d label the current payment execution and compliance thresholds as **demo/simulation logic**, as the README above does.
