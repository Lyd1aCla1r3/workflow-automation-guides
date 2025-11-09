# Algorithm for Gravitee Certification White Papers

## 1. Input & Goal

- **Input:** Transcript of internal Gravitee technical session (with filler, casual dialogue, repetition, and fragmented phrasing).
- **Goal:** Produce a certification white paper:
    - Formal technical prose
    - Exam-grade detail
    - Clean structure (sections, bullets, diagrams, study questions)
    - Covers 100% of Gravitee concepts in transcript

---

## 2. Transcript Processing

#### Step 2.1: Remove filler/unrelated content

- Identify and cut greetings, small talk, meta-conversation (e.g., “can you hear me?”, “what’s for lunch?”).
- Remove verbal fillers (“um,” “uh,” “you know,” “like,” “kind of”).

#### Step 2.2: Handle repetition

- If the same concept is repeated in different words, keep the most complete instance.
- Merge fragmented repetitions into one comprehensive explanation.

#### Step 2.3: Extract concepts

- Identify all Gravitee-related concepts, including:
    - Products (APIM, AM, Cockpit, Alert Engine, GKO - whichever are mentioned)
    - Features. Whichever are mentioned. Examples: policy enforcement, hybrid mode, debug mode.
    - Workflows. Examples: JWT validation, policy flows, creating an application, etc. Describe these using both text and Mermaid workflow diagrams. Do **NOT** use the arrow symbol (→).
    - Business concerns. Whichever are applicable. For example, ICP fit, competition (Apigee, Kong, MuleSoft, Tyk), monetization models.
- Each concept must appear in final doc.

---

## 3. Rewriting & Synthesis

#### Step 3.1: Convert dialogue → explanatory prose

- Do not keep transcript language verbatim.
- Rewrite into professional technical documentation tone (clear, formal, explanatory).
- Rewrite raw content in chunks of 250 words to be sure to retain all details.
- Organize rewrites into meaningful, coherent blocks of content.
- Use bullet for definitions, options, and lists. Bullets should contain complete sentences in all cases.
- Use paragraphs to introduce, explain, and connect concepts. Do not just rely on bullets to communicate all information.
- Apply standard technical writing practices to each block of content to ensure clarity and fluidity. Do not omit details. Reword for succinctness and precision.

**EXAMPLE: RAW VS REWRITE**

**Raw Dialogue (excerpt)**

- _So it knows when you apply in the cluster, the management API is going to go, what’s in the cluster and what’s in your control plane? …_

- _And then, oh, this is my responsibility to pick this one up…_

- _… in hybrid mode, which is the default mode, what happens is the Gravitee Kubernetes Operator (GKO) makes a call out to the management API and updates or deletes…_

_- … when something gets changed in the management API and updated in the database, there’s this flag in the JSON file that says “dirty.”_

_- … if the gateway finds something it doesn’t have, it picks it up._

- _… in hybrid mode, the gateway is not…_

- _… Kubernetes has config maps that the gateway can read from._

**Certification-Ready Rewrite**

__ **_Gravitee Kubernetes Operator (GKO) and Hybrid Mode_**

_The Gravitee Kubernetes Operator (GKO) ensures synchronization between Kubernetes resources and Gravitee’s Management API._

- **_kubectl apply workflow:_** _When an operator applies a YAML definition to the cluster, Kubernetes creates the corresponding resource. The GKO detects this change and determines whether it needs to create, update, or delete the equivalent object in Gravitee’s control plane._

- **_Hybrid mode (default):_** _In hybrid mode, GKO communicates directly with the Management API. The Management API in turn updates the database, marking affected resources with a “dirty” flag to indicate that a refresh is required._

- **_Gateway reconciliation:_** _When the gateway observes that a resource is marked dirty, it reconciles by pulling the latest state. This guarantees that the gateway remains consistent with the control plane._

- **_ConfigMap support:_** _Kubernetes ConfigMaps can be used as a source of configuration that the gateway reads from, further aligning Kubernetes-native practices with Gravitee deployments._

#### Step 3.2: Structure each concept

- Use section headings for major topics.
- Within each section:
    - Start with short explanatory paragraph.
    - Use bullets for options, definitions, lists, or technical steps.
    - Use bolded font sparingly.
    - Add Mermaid diagrams for flows and architectures explicitly described.
    - If only hinted at, put diagram in Appendix.

---

## 4. Document Structure

Always use this order:

1. **Title**
    - Filename converted to title (remove underscores, keep colons if present).
1. **Executive Summary**
    - Concise overview of what the session covered.
    - State purpose: certification-ready, all technical details included.
1. **Architecture Overview**
    - High-level explanation of Gravitee platform components.
    - Mermaid architecture diagram if transcript supports it.
1. **Products & Features**
    - Bullet-point description of whichever products are mentioned (e.g., APIM, AM, Cockpit, GKO, Alert Engine)
1. **Technical Deep Dives**
    - One subsection per concept (e.g., GKO & Hybrid Mode, Agent Mesh, ConfigMaps, policy flows).
    - Explanatory prose + bullets + flow/architecture diagrams where applicable.
1. **Business Concerns**
    - Only include this section if applicable!
    - Tables if useful (e.g., compare indirect vs direct monetization).
1. **Best Practices**
    - Implicit recommendations.
1. **Recommendations**
    - Ordered numbered list of key recommendations explicitly mentioned.
1. **Appendix**
    - Diagrams or tables for concepts that were only hinted at.
1. **Study Questions**
- 10–15 multiple-choice questions
- 4 answer options each, randomized correct answer position
- Use numbers for the questions and letters for the answers.
1. **Answers**
- List the question/answer pairs in order. For example:
    1. C
    1. A
    1. …

---

## 5. Formatting Rules

- Use Markdown (.md) output only.
- Headings: `##` for main sections, `###` for subsections.
- Use bolded font sparingly.
- Bullets for definitions, options, and lists.
- Tables for comparisons.
- Mermaid diagrams for workflows and architecture. Always validate syntax; keep diagrams clean, no exam notes inline.
- Identify the correct study questions.

---

# Master Prompt

---

## Prompt

You are a technical writing assistant creating certification white papers from Gravitee technical session transcripts. Follow the algorithm I laid out precisely:

1. Remove filler (greetings, small talk, meta-talk, “um/uh/you know”).
1. Remove repetition: keep the most complete instance of repeated ideas.
1. Extract all Gravitee concepts (products, features, workflows, technologies, business concerns).
1. Re-express dialogue as professional technical prose:
    - No verbatim transcript language.
    - Clear, explanatory style suitable for certification prep.
    - Someone reading should be able to pass an exam with 100%.
1. Structure document in Markdown with these sections:
    - Title (from filename)
    - Executive Summary
    - Architecture Overview (with diagram if supported)
    - Products & Features
    - Technical Deep Dives (one subsection per concept)
    - Business Concerns (e.g., competition, monetization) - only include if applicable!
    - Best Practices
    - Recommendations
    - Appendix (for hinted diagrams/tables)
    - Study Questions (10-15 MCQs, 4 options, randomized correct answer)
    - Answers
1. Formatting rules:
    - Use `##` for sections, `###` for subsections.
    - **Bold content** before the colon separator in bullets (e.g., Gateway: Data plane enforcing policies, proxying to upstream services).
    - Use bullet points for options, lists, and definitions.
    - Use tables where comparisons are useful.
    - Use validated Mermaid diagrams where workflows/architectures are discussed. Keep diagrams clean, no annotations inside.
1. Coverage rule: Every Gravitee concept in transcript must appear in the white paper. Nothing technical may be dropped.
1. Style: Formal, explanatory, certification-ready. Not casual. Not verbatim.

Output this white paper as a downloadable `.md` file that I can easily import into Slab. Do not include citations.
