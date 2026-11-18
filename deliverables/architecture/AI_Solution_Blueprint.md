# Vendor Intelligence & Automated Triage (VIAT) System
## AI Solution Blueprint
**Prepared For:** ApexLogistics US  
**Prepared By:** OmniNode Systems  
**Document Version:** 1.0 - Proof of Concept Phase  

---

## 1. Executive Summary

ApexLogistics US is currently experiencing operational bottlenecks in its vendor support division due to an unscalable array of custom glue code used to manage vendor communications. This legacy infrastructure lacks observability, suffers from silent failures, and cannot intelligently parse nuanced freight disputes, resulting in brand-damaging errors and SLA breaches. 

OmniNode Systems has been engaged to replace this brittle architecture with the **Vendor Intelligence & Automated Triage (VIAT) System**. VIAT is an n8n-based, resilient agentic automation framework designed to autonomously categorize, evaluate, and route vendor inquiries. This blueprint defines the underlying AI architecture, containerized deployment strategy, and security measures required to safely process freight disputes, ensuring that high-stakes issues are routed to a Human-in-the-Loop (HITL) for executive oversight.

---

## 2. Foundational AI Concepts

To achieve intelligent triage, the VIAT system leverages Large Language Models (LLMs) over other generative architectures. Understanding the mechanics of these models is critical for enterprise deployment.

### Two-Pass Training Context
The selected LLM is Google **Gemini 2.0 Flash** (`models/gemini-2.0-flash`), a Transformer-based model accessed through a standard LLM API. Like other production LLMs, it is built with a two-pass training methodology:
1. **Pre-training (pass 1):** Gemini 2.0 Flash is trained on a very large, multilingual corpus of text, code and documents to predict the next token. This pass gives it a broad, high-dimensional representation of language syntax, semantics and reasoning patterns, which is why it already understands freight vocabulary such as BOL, detention, lumper or TONU without any ApexLogistics-specific training.
2. **Post-training / fine-tuning (pass 2):** The pre-trained model is then refined with supervised instruction tuning and Reinforcement Learning from Human Feedback (RLHF). This pass aligns Gemini 2.0 Flash to follow instructions, respect system messages and return structured output, which is exactly what VIAT relies on: obeying the negative prompting rules and returning a single JSON triage decision. VIAT does not re-train the model; it steers this aligned model with prompt engineering (Section 5) and, in Phase 2, retrieval (RAG).

### Semantic Latent Space vs. GANs and VAEs
*   **Semantic Latent Space in LLMs:** LLMs map concepts into a continuous, high-dimensional semantic latent space where proximity equates to contextual similarity (e.g., "damaged pallet" is vectorially close to "freight claim"). This makes them exceptionally suited for Natural Language Understanding (NLU) tasks like parsing messy vendor emails.
*   **GANs (Generative Adversarial Networks):** Rely on a two-network system (generator and discriminator) playing a zero-sum game. While excellent for generating photorealistic images or synthetic tabular data, they struggle with discrete, sequential logic required for coherent text generation.
*   **VAEs (Variational Autoencoders):** Work by compressing input data into a probabilistic latent distribution and reconstructing it. VAEs are strong in anomaly detection and continuous data generation but lack the complex attention mechanisms (Transformers) necessary to maintain long-term context in dense vendor correspondence.

**Conclusion:** A Transformer-based LLM such as Gemini 2.0 Flash is the best-suited architecture for the text-heavy, reasoning-dependent requirements of the VIAT system.

---

## 3. Tripartite Architecture Mapping

The VIAT n8n PoC is designed using a modern tripartite agentic architecture, bridging the gap between vendor inputs and system actions.

1. **Perception (Sensory Input):** 
   *   **Implementation:** n8n Webhook Trigger.
   *   **Function:** Ingests incoming vendor tickets, payloads, and CSV data dumps. This replaces the legacy glue code's polling mechanisms, providing real-time, event-driven perception of vendor issues.
2. **Cognition/Brain (Processing & Decision Making):** 
   *   **Implementation:** n8n LLM Node (`VIAT Triage Executor (LLM)` — Basic LLM Chain backed by the Google Gemini Chat Model, `gemini-2.0-flash`, temperature 0).
   *   **Function:** Acts as the central "executor." It applies prompt templates to project the unstructured vendor text into its latent space, extracting key entities (Tracking IDs, dispute types) and classifying the severity of the ticket.
3. **Action (Execution & Feedback Loop):**
   *   **Implementation:** n8n Conditional Routing (If Nodes), Email Nodes, and HITL Wait Nodes.
   *   **Function:** Closes the perception-action loop. All non-high-value tickets trigger an automated email response, after passing the Brand Safety Guardrail. High-value disputes are routed to the HITL flow for executive review.

---

## 4. Compute, Storage & Containerization Strategy

To ensure VIAT does not succumb to the same fragility as the legacy infrastructure, it will be deployed using a robust, containerized microservices architecture.

### Deployment Topology
*   **Orchestration:** Docker Compose for the PoC (`infrastructure/docker-compose.yml`); Docker Swarm or Kubernetes (K8s) for production. The Compose file defines five services: `postgres`, `redis`, `n8n` (main: editor, webhook intake, HITL resume URLs), `n8n-worker` (execution) and `ngrok` (tunnel, optional profile). All n8n containers are pinned to image `n8nio/n8n:1.90.2`.
*   **Compute Constraints (n8n Workers):** Minimum `2 vCPU`, `4GB RAM` per instance to handle high-concurrency JSON parsing and API execution without OOM (Out of Memory) errors. In Compose, `deploy.resources.reservations` (`cpus: "2"`, `memory: 4g`) guarantees this minimum on both `n8n` and `n8n-worker`, and matching `deploy.resources.limits` caps each instance at the same size so one busy worker cannot starve the host; PostgreSQL is capped at `1 vCPU` / `1GB RAM`. Workers start only after PostgreSQL and Redis report healthy.
*   **Primary Storage:** **PostgreSQL** (`postgres:16-alpine`, persistent volume `postgres_data`) is mandated as the primary database for n8n to ensure ACID compliance, persistent execution logs, and credential storage. SQLite is strictly prohibited for production. The same database holds the `viat_dead_letter_queue` table (Section 6). Failed executions are always saved (`EXECUTIONS_DATA_SAVE_ON_ERROR=all`) and execution history is pruned after 14 days (`EXECUTIONS_DATA_MAX_AGE=336` hours) to bound storage growth.
*   **Scaling Strategy (Queue Mode):** To handle seasonal spikes in freight disputes (e.g., Q4 holiday surges), n8n will be deployed in **Queue Mode** (`EXECUTIONS_MODE=queue`). **Redis** (`redis:7-alpine`) will act as the message broker, distributing execution workloads across multiple containerized n8n worker nodes. The main instance only accepts webhooks and enqueues jobs, so capacity is added horizontally without downtime: `docker compose up -d --scale n8n-worker=4`. All instances share one `N8N_ENCRYPTION_KEY` so workers can decrypt credentials.
*   **Local Development & Testing:** For the initial PoC, ngrok is configured to expose the local webhook securely to the internet, allowing ingestion of the sample vendor CSV dataset. It runs as the `ngrok` service under the `tunnel` profile:
    ```bash
    NGROK_AUTHTOKEN=<token> NGROK_DOMAIN=<name>.ngrok-free.app \
    WEBHOOK_URL=https://<name>.ngrok-free.app/ \
    docker compose --profile tunnel up -d
    ```
    `NGROK_DOMAIN` is required for the `tunnel` profile (the container exits with an error if it is missing) and must match `WEBHOOK_URL`, which makes n8n generate public webhook and HITL resume URLs.
*   **Workflow Import & Activation:** Import both workflows with the n8n CLI so their IDs are kept and the primary workflow's `errorWorkflow` setting stays linked to the DLQ:
    ```bash
    # run from infrastructure/
    docker compose cp ../n8n_workflows n8n:/tmp/n8n_workflows
    docker compose exec n8n n8n import:workflow --separate --input=/tmp/n8n_workflows
    ```
    If the workflows are imported through the editor instead ("Import from File"), n8n assigns new IDs: open the primary workflow → **Settings → Error Workflow** and select "VIAT - Dead Letter Queue (Error Handler)", then save. Next, create the credentials (Gemini, SMTP, Postgres, and the `VIAT Webhook Header Auth` Header Auth credential with name `X-VIAT-Token` and a long random value) and **activate** the primary workflow. The production URL `/webhook/vendor-ticket` only responds while the workflow is active; for a one-off editor test, click "Listen for test event" and use `/webhook-test/vendor-ticket`.
*   **Webhook Security:** Because the tunnel exposes the webhook publicly, the Webhook node requires Header Auth: requests without the correct `X-VIAT-Token` header are rejected with `403` before any workflow logic runs. Production deployments must also replace every default secret in Compose (`POSTGRES_PASSWORD`, `N8N_ENCRYPTION_KEY`) via a `.env` file that is never committed.
*   **Sample Data Test:** Each row of `vendor_tickets_sample-csv.csv` is posted as one JSON ticket, replacing the legacy glue code:
    ```bash
    VIAT_TOKEN=<token> python3 -c "import csv,json,os,sys,urllib.request as u; [u.urlopen(u.Request(sys.argv[1], json.dumps(r).encode(), {'Content-Type':'application/json','X-VIAT-Token':os.environ['VIAT_TOKEN']})) for r in csv.DictReader(open('vendor_tickets_sample-csv.csv'))]" https://<name>.ngrok-free.app/webhook/vendor-ticket
    ```

---

## 5. Prompt Engineering Strategy

The LLM node relies on advanced prompt engineering to ensure accurate and safe triage. The following JSON configurations (source of truth: `prompts/triage_prompts.json`) are injected into the n8n LLM Node: Template 1 is the node's System message and Template 2 is its User prompt, where `{{ $json.* }}` are n8n expressions filled from the normalized ticket. Both templates require the model to output a single structured JSON object (schema in `prompts/triage_prompts.json` → `output_schema`).

### Prompt Template 1: System Definition & Negative Prompting
This prompt enforces strict boundaries, specifically utilizing **negative prompting** to prevent the model from hallucinating contractual terms. It explicitly prohibits any mention of US dollar amounts or contract timelines (SLA terms, payment terms, due dates, deadlines) in every output field.

```json
{
  "role": "system",
  "content": "You are the VIAT (Vendor Intelligence & Automated Triage) classifier for ApexLogistics US. You read one vendor freight ticket and return a triage decision as a single JSON object. OUTPUT FORMAT: Respond with ONLY one valid JSON object. Do not wrap it in markdown code fences and do not add any text before or after it. NEGATIVE PROMPTING - YOU MUST NOT: (1) mention, repeat, estimate, calculate or invent any US dollar amount, price, rate, tariff, fee, penalty, payout or other monetary figure in any output field; (2) mention, promise or invent any contract timeline, SLA term, payment term, due date, deadline or turnaround time in any output field; (3) state or imply ApexLogistics pricing, SLA commitments or contractual obligations; (4) invent facts, IDs or names that are not present in the ticket. When the ticket concerns money or contract terms, refer to it only in neutral words such as 'the disputed charge' or 'the payment terms', and say that the responsible ApexLogistics team will review it."
}
```

### Prompt Template 2: User Inquiry & Self-Asking Framework
This prompt utilizes a **self-asking** chain to force the model to decompose complex, multi-layered vendor complaints before arriving at a final categorization. The model asks and answers five follow-up questions (core issue, reference IDs, formal dispute, high-value signals, decision). A ticket is classified as **'High-Value Dispute'** when it is a formal dispute AND at least one high-value signal applies: the vendor's `Priority_Level` is High, the vendor explicitly calls it high-value or asks for escalation or executive review, the stated impact exceeds 5,000 USD, or legal action or ending the relationship is threatened. That class drives the HITL route, so every High-priority dispute in the sample CSV (e.g. VEN-1002, VEN-1011, VEN-1023) reaches an executive before any reply is sent.

```json
{
  "role": "user",
  "content": "Triage the vendor ticket below.\n\nTicket ID: {{ $json.ticket_id }}\nVendor: {{ $json.vendor_name }}\nSubject: {{ $json.subject }}\nVendor priority: {{ $json.priority_level }}\nMessage: {{ $json.message_body }}\n\nUse self-asking. Ask yourself each follow-up question below, answer it from the ticket only, then decide.\nQ1: What is the core issue being reported (for example billing or payment discrepancy, damaged or short freight, detention or accessorial charge, scheduling or operational problem, document request, general inquiry)?\nQ2: Which load numbers, invoice numbers, BOL IDs, check numbers or other reference IDs appear in the message?\nQ3: Is the vendor formally disputing a charge, deduction, claim or payment?\nQ4: Does ANY of these apply: (a) the vendor priority above is 'High'; (b) the vendor explicitly calls the issue high-value, or asks for it to be escalated or reviewed by an executive or VP; (c) the stated financial impact is above 5,000 USD; (d) the message threatens legal action or ending the business relationship? Answer only yes or no and name the letters that apply, without repeating any amount.\nQ5: Decision: if Q3 is yes AND Q4 is yes, the classification is 'High-Value Dispute'. Otherwise choose the best fit from 'Standard Dispute', 'Billing Inquiry', 'Operational Issue', 'Documentation Request', 'General Inquiry'.\n\nReturn ONLY a JSON object with exactly these keys:\n\"ticket_id\": string, the ticket ID above;\n\"self_ask\": array of 5 objects, each with \"question\" and \"answer\" strings, for Q1 to Q5;\n\"classification\": one of 'High-Value Dispute', 'Standard Dispute', 'Billing Inquiry', 'Operational Issue', 'Documentation Request', 'General Inquiry';\n\"severity_level\": one of 'low', 'medium', 'high', 'critical';\n\"extracted_ids\": array of strings;\n\"draft_response\": a short, polite reply to the vendor that acknowledges the issue and states it has been routed to the correct team.\nThe negative prompting rules from the system message apply to every key, including self_ask answers and draft_response: no US dollar amounts and no contract timelines."
}
```

---

## 6. Agentic Workflow Diagram (n8n PoC Overview)

The solution relies on two distinct n8n workflows to ensure both automation and system resiliency.

### Workflow A: Primary Triage & HITL (The Executor)
File: `n8n_workflows/ticket_triage_workflow.json` — "VIAT - Vendor Ticket Triage (Primary)". Its workflow setting `errorWorkflow` points to Workflow B (kept by CLI import; re-select it after an editor import, see Section 4).
1. **Webhook Trigger (`Vendor Ticket Webhook`):** `POST /webhook/vendor-ticket`, protected by Header Auth (`X-VIAT-Token`), receives one vendor ticket (one CSV row as JSON: `Ticket_ID`, `Date`, `Vendor_Name`, `Subject`, `Message_Body`, `Priority_Level`).
2. **Data Prep Node (`Normalize Ticket`):** Validates required fields and normalizes the payload. A malformed ticket throws an error so it reaches the DLQ instead of failing silently.
3. **LLM Node (`VIAT Triage Executor (LLM)` + `Google Gemini Chat Model`):** Executes the JSON prompt templates against the payload to classify the issue.
4. **Output Validation (`Parse Classification JSON`):** Parses the model's JSON and rejects non-JSON output or unknown classifications (routed to the DLQ).
5. **Brand Safety Guardrail (`Brand Safety Guardrail`):** Deterministic check on `draft_response` before either vendor-email path. It blocks any reply containing a US dollar amount or monetary figure (`$`, `USD`, `dollars`, amounts such as `1,250` or `450.00`), a contract timeline (`net 30`, `within 5 days`, `48 hours`) or an SLA/deadline/guarantee term. A blocked ticket throws an error, so it is isolated in the DLQ and no email is sent.
6. **If Node (`Is High-Value Dispute?`, Routing Logic):**
   *   *Path 1 (all other classes):* Routes to standard automated email response (`Send Automated Vendor Response`).
   *   *Path 2 (High-Value Dispute):* Routes to the **Human-in-the-Loop (HITL) Wait Node**.
7. **HITL Mock Notification (`HITL Executive Alert (Mock Notification)`):** Email to an ApexLogistics executive with the ticket, the proposed reply, and secure resume links (`{{ $execution.resumeUrl }}?decision=approve` / `?decision=reject`).
8. **HITL Wait Node (`HITL Wait - Executive Approval`):** Pauses workflow execution until the executive opens one of the links.
9. **If Node (`Executive Approved?`):** Only on `decision=approve` does the perception-action loop resume and `Send Approved Vendor Response` email the vendor. A rejection ends the execution without contacting the vendor.

### Workflow B: Dead Letter Queue (DLQ) & Observability
File: `n8n_workflows/dlq_error_workflow.json` — "VIAT - Dead Letter Queue (Error Handler)". To prevent the silent failures inherent in the old glue code, a secondary workflow acts as a safety net.
1. **Error Trigger Node (`Error Trigger`):** Fires for any failed execution in Workflow A (e.g., LLM API timeout or auth error, malformed payload, invalid model output).
2. **Data Extraction (`Extract Failure Details`):** Captures the workflow, execution ID and URL, the node that failed, the error message and the execution mode.
3. **Log & Isolate (`Log to Dead Letter Queue (Postgres)`):** Inserts a row into the PostgreSQL table `viat_dead_letter_queue` (created on first use) with `status = 'pending_review'`, including the full saved execution data (`raw_execution_data`), which contains the original vendor payload for replay. This ensures every failed execution is captured for review and replay instead of failing silently.

---

## 7. Responsible AI Risk Review & Guardrails

Deploying autonomous systems in enterprise supply chains carries inherent risks. The VIAT system incorporates the following guardrails:

*   **Risk:** Hallucination of Rates/SLAs.
    *   **Mitigation:** Enforced Negative Prompting at the system level (as detailed in Section 5): the model must not mention any US dollar amount or contract timeline in any output field, and must refer to money or terms only in neutral words.
*   **Risk:** Brand-damaging or malformed automated replies.
    *   **Mitigation:** Temperature 0 for consistent output; mandatory structured JSON output; the `Parse Classification JSON` node rejects any output that is not valid JSON or has an unknown classification. Vendor emails only use the model's short `draft_response`, which is bound by the negative prompting rules and then independently checked by the deterministic `Brand Safety Guardrail` node (Section 6, step 5). The guardrail does not trust the prompt: any draft mentioning a dollar amount, monetary figure, contract timeline or SLA/deadline term is blocked, no reply is sent, and the ticket is isolated in the DLQ for human review.
*   **Risk:** Unauthorized or spoofed tickets through the public tunnel.
    *   **Mitigation:** Header Auth on the Webhook node (`X-VIAT-Token`) rejects unauthenticated requests; secrets are supplied through an uncommitted `.env` file and shared only through n8n's encrypted credential store (`N8N_ENCRYPTION_KEY`).
*   **Risk:** Autonomous Action on High-Stakes Legal/Financial Disputes.
    *   **Mitigation:** The HITL Wait Node guarantees that any ticket classified as 'High-Value Dispute' requires explicit human authorization before an external action is taken; a rejection sends nothing to the vendor.
*   **Risk:** Silent System Failures & Lost Tickets.
    *   **Mitigation:** Implementation of the Error Trigger and Dead Letter Queue (DLQ) workflow, so every failed execution is logged to the DLQ; if the DLQ insert itself fails, the failure remains visible in n8n's saved error executions (`EXECUTIONS_DATA_SAVE_ON_ERROR=all`).

### Future Phase: Retrieval-Augmented Generation (RAG)
For Phase 2, we will integrate a **Retrieval-Augmented Generation (RAG)** architecture. Instead of relying purely on the LLM's latent space for general freight knowledge, the n8n workflow will query a vector database (e.g., Pinecone or Qdrant) containing ApexLogistics US's official Vendor SLA handbooks and standard operating procedures. The retrieved text will be injected into the prompt context, firmly grounding the AI's categorization logic in proprietary company policies and further reducing hallucination risks.