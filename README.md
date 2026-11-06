# System Design Portal — Intern Handbook

Welcome to the System Design Portal content-development project.

This repository contains the working structure and starter material for building a practical system-design learning platform. Your task is not simply to collect definitions. You are expected to research how systems and their components work, explain the reasoning behind architectural decisions, document trade-offs, and identify how the concepts could be explored through an interactive playground or simulation.

Use this README as the project-wide guide. The individual case-study and component files are the detailed working documents for each assigned item.

---

## 1. Project Objectives

The portal is intended to help learners understand, inspect, design, and evaluate software systems.

The content should help a learner:

- Understand a system through a realistic problem and its requirements.
- Move from a simple or naive design towards a more capable architecture.
- Understand the role and behaviour of individual components.
- Follow requests and data through a system.
- Explore bottlenecks, failure modes, scaling strategies, and trade-offs.
- Experiment with configuration choices and observe their effects.
- Practise explaining and defending architecture decisions.

The catalogue supplied with this project contains **250 case studies across 13 categories**. The knowledge library contains **50 major knowledge domains**. These catalogues define the project's source structure; do not independently rename, merge, or reorder their entries without approval.

## 2. Your Responsibilities

For each assigned item, produce research that is:

1. **Technically accurate** — terminology, behaviour, and explanations must be correct.
2. **Understandable** — explain concepts clearly without removing important technical detail.
3. **Evidence-based** — use credible sources and record where claims came from.
4. **Practical** — include realistic examples, flows, configurations, and failure scenarios.
5. **Useful for learning** — explain why a design works, where it fails, and what changes under different conditions.
6. **Simulation-aware** — identify meaningful inputs, outputs, variables, and observable behaviour where simulation is relevant.

Do not treat the assignment as a copy-and-paste research exercise. Read sources, compare explanations, understand the mechanism, and write the result in your own words.

## 3. Workstreams

The project has two connected content workstreams.

### A. Case Studies

A case study explains how to design or reason about a complete system. It begins with a problem, derives requirements, develops an architecture, and examines how the design behaves under load and failure.

The case-study catalogue provides the assigned title, category, and ordering. Use the case-study template in the relevant case-study folder.

### B. Component Library

A component entry explains one reusable building block of a system, such as a load balancer, API gateway, cache, database, message broker, or other architectural component.

A component entry should explain what the component does, how it works, how it interacts with other components, what can go wrong, and which configuration choices matter.

### How the Workstreams Connect

Case studies and components must use stable IDs to refer to one another. For example, a case study should reference the component IDs it uses rather than creating a new, slightly different name for the same component. Component entries should list relevant case studies where that component appears.

Keep the master index and relationship file consistent with the content folders. Do not create duplicate IDs.

## 4. Repository Structure

Use the existing folder structure. A typical item should follow this pattern:

```text
system-design-content/
├── case-studies/
│   └── CS-001-url-shortener/
│       ├── research.md
│       ├── architecture.png
│       ├── data-flow.md
│       ├── simulation-spec.md
│       └── references.md
├── component-library/
│   └── CMP-001-load-balancer/
│       ├── specification.md
│       ├── simulation-spec.md
│       ├── visual-spec.md
│       └── references.md
└── master-index/
    ├── case-study-index.csv
    ├── component-index.csv
    └── relationships.csv
```

The exact files required for an assignment may vary. Follow the assignment tracker and ask the coordinator before changing the folder or file naming convention.

### Naming and IDs

- Case studies use IDs such as `CS-001`.
- Components use IDs such as `CMP-001`.
- Keep IDs stable even if a title is refined.
- Use lowercase, hyphen-separated folder names.
- Do not reuse an existing ID for a different item.
- Update the relevant CSV index when an item is added or its approved metadata changes.

## 5. Case-Study Documentation

Use `research.md` as the main case-study document. The supplied case-study structure has **18 sections**. Preserve the section names and order:

| No. | Section | What to Document |
|---:|---|---|
| 01 | Story | Introduce the problem, users, context, and motivation. |
| 02 | Requirements | Separate functional requirements from non-functional requirements. |
| 03 | Scale | State the workload assumptions and explain the expected scale. |
| 04 | Naive Architecture | Show a simple initial design that could satisfy the basic requirements. |
| 05 | Problems | Explain the limitations, bottlenecks, and risks in that design. |
| 06 | Architecture Evolution | Show how and why the design changes as requirements and scale increase. |
| 07 | Components | Identify the components and reference their component-library IDs where available. |
| 08 | Data Flow | Explain the important request, response, and background-processing flows. |
| 09 | Failure Scenarios | Describe likely failures, their effects, and possible handling or recovery. |
| 10 | Scaling | Explain relevant horizontal/vertical scaling and workload-specific strategies. |
| 11 | Trade-offs | Compare alternatives and explain the consequences of design choices. |
| 12 | Cost | Provide a transparent cost model or estimate, with assumptions and pricing sources. |
| 13 | Security | Cover relevant threats, trust boundaries, and protective measures. |
| 14 | Observability | Identify useful logs, metrics, traces, alerts, and service-level indicators. |
| 15 | Simulation | Specify what a learner should be able to change and observe. |
| 16 | Design Challenge | Provide a task that asks the learner to make or improve a design decision. |
| 17 | AI Interview | Add questions that test understanding and the ability to justify decisions. |
| 18 | Architecture Score | Define or apply a clear evaluation rubric; explain the score rather than presenting an unexplained number. |

### Requirements and Scale Assumptions

Do not present invented workload figures as real-world facts. If exact figures are not available, label them as assumptions and explain why they are reasonable for the exercise.

For example, a workload model might state assumed daily requests, peak requests per second, read/write ratio, average payload size, or retention period. Explain which design decisions those assumptions influence.

### Architecture Evolution

Do not jump directly from a simple diagram to a complex architecture. Explain the reason for each meaningful change. A learner should be able to follow the chain:

**Requirement or pressure → limitation in the current design → proposed change → benefit and trade-off.**

### Company-Specific Systems

When a case study is based on a known product or company, do not claim that your proposed architecture is the company's actual internal production architecture unless a credible public source explicitly documents it.

Clearly distinguish:

- **Publicly documented fact:** supported by a source.
- **Common industry pattern:** a generally used design approach, not necessarily confirmed for the named company.
- **Proposed design:** your reasoned architecture for the learning exercise.
- **Assumption or estimate:** a value introduced to make the example or calculation possible.

This distinction is especially important for private infrastructure, traffic figures, internal technology choices, and cost estimates.

## 6. Component-Library Documentation

Use `specification.md` as the main component document. Cover the relevant topics from the supplied template, including:

- Component identity, category, and concise definition.
- The problem it solves and when it is useful.
- Internal working principles and important concepts.
- Component types or implementation approaches, where applicable.
- Algorithms, routing policies, or configuration options, where applicable.
- Inputs, outputs, and important state.
- Dependencies and interactions with other components.
- Common use cases and practical examples.
- Advantages, limitations, and design trade-offs.
- Failure modes, recovery behaviour, and scaling considerations.
- Security considerations.
- Monitoring and operational signals.
- Playground or simulation requirements.
- Credible references and further reading.

Avoid descriptions that only say what a component is. Explain what happens when a request reaches it, what decisions it makes, what it sends onward, and what happens when dependencies are unavailable.

### Component Boundaries

Be clear about what belongs to the component and what belongs to neighbouring components. For example, explain whether a feature is performed by the load balancer itself, by an application server, or by a separate health-checking or service-discovery system. Do not attribute every surrounding system capability to the component being documented.

## 7. Research Methodology and Source Quality

Use a combination of primary and reputable secondary sources.

Preferred sources include:

1. Official product and vendor documentation.
2. Standards and technical specifications (for example, RFCs).
3. Engineering blogs or architecture articles published by the organisations involved.
4. Well-regarded technical books, papers, and conference talks.
5. Reputable educational resources for supplementary explanations.

### Research Process

1. Read the assigned topic and identify the questions the document must answer.
2. Find at least one authoritative source for the core behaviour.
3. Cross-check important claims against another credible source where possible.
4. Record the exact page or document title and URL in `references.md`.
5. Write a synthesis in your own words.
6. Mark assumptions, estimates, and proposed designs explicitly.
7. Revisit references before submission to ensure they still support the claims.

A search-result snippet is not sufficient evidence for a technical claim. Open and read the source itself.

### Source and Copyright Rules

- Do not copy large sections of documentation, articles, diagrams, or books.
- Do not remove attribution from diagrams or images.
- Prefer original diagrams created for this project.
- Use short quotations only when necessary, with attribution.
- Record the source and access date for important external material.
- Do not include confidential, private, or personal information.

## 8. Architecture Diagrams and Data Flows

Diagrams should communicate system behaviour, not merely decorate the document.

Every architecture diagram should:

- Have a clear title and readable labels.
- Show the relevant actors, components, and external dependencies.
- Use arrows consistently and make direction clear.
- Distinguish synchronous requests from asynchronous messages where relevant.
- Avoid unexplained abbreviations.
- Match the written explanation and data-flow document.
- Identify trust boundaries or failure boundaries when relevant.

For important flows, explain the steps in order. State which component initiates the action, what data is passed, what response is expected, and where errors can occur.

Use a separate sequence or flow diagram when a single architecture diagram becomes too crowded. Do not hide important behaviour behind a generic arrow labelled “process”.

## 9. Simulation and Playground Research

A simulation specification describes behaviour that can be implemented and tested. It is not a request for a static illustration.

For each proposed simulation, define:

### Inputs

What can the learner change? Examples include request rate, number of servers, cache capacity, timeout, retry count, routing policy, or failure probability. Include valid ranges and defaults where possible.

### Model and Rules

Explain the simplified model the simulator should implement. State the assumptions and formulas. If the simulation is conceptual rather than a production-accurate model, say so.

### Outputs

What should the learner observe? Examples include latency, throughput, queue length, error rate, server utilisation, cache hit ratio, dropped requests, or estimated cost.

### Interactions

Describe what happens when the learner changes a setting, starts or pauses a run, introduces a failure, or compares configurations.

### Experiments

Provide a few repeatable experiments. Each should state the starting configuration, the change to make, and the behaviour the learner should observe.

### Validation

Define expected outcomes or acceptance checks. For example, if traffic is increased while server capacity remains fixed, explain what the model should show once demand exceeds capacity.

Do not invent precise performance predictions without a model and assumptions. Do not label a visual animation as a working simulation unless the underlying behaviour is actually modelled.

## 10. Documentation Standards

All submissions must follow these standards:

- Use Markdown headings consistently.
- Use concise, technically precise language.
- Define an acronym on first use.
- Explain specialised terms when the intended learner may not know them.
- Use tables for structured comparisons, configuration options, and input/output definitions.
- Use code blocks for commands, configuration, and code.
- Keep diagrams and written explanations consistent.
- Include units in measurements (ms, seconds, requests/second, GB, and so on).
- State assumptions beside the calculations or claims they affect.
- Avoid unsupported superlatives such as “best”, “fastest”, or “most scalable”.
- Avoid filler text and repeated explanations.
- Check spelling, grammar, links, and Markdown rendering before submission.

A good explanation answers not only **what** a component does, but also **why**, **how**, **when**, and **what happens if it fails**.

## 11. Quality Review and Scoring

Submissions are reviewed for technical correctness, completeness, reasoning, practical value, and evidence quality.

Use this project rubric as the review baseline:

| Criterion | Weight |
|---|---:|
| Technical accuracy | 25% |
| Completeness | 20% |
| Architecture and data-flow understanding | 20% |
| Practical examples | 15% |
| Source quality | 10% |
| Simulation or visualization detail | 10% |
| **Total** | **100%** |

A high score does not compensate for a critical technical error. Reviewers may return work for correction even when most sections are complete.

### Suggested Workflow Statuses

Follow the assignment tracker. The standard progression is:

1. **Assigned** — topic and owner are recorded.
2. **Researching** — sources are being collected and evaluated.
3. **Documentation** — the research is being written and diagrams/specifications are being prepared.
4. **Submitted for Review** — the deliverables are ready for review.
5. **Technical Review** — a reviewer is checking accuracy and completeness.
6. **Approved** — the work has passed review.

Do not mark an item Approved yourself unless the coordinator has explicitly assigned you that review authority. Record reviewer feedback and resolve requested changes before resubmission.

## 12. Submission Deliverables

Check the assignment tracker for the exact deliverables for your item. A typical case-study submission includes:

- Completed `research.md`.
- Architecture diagram(s).
- `data-flow.md`.
- `simulation-spec.md`, if required.
- `references.md`.
- Updated case-study index and relationship entries, if assigned.

A typical component submission includes:

- Completed `specification.md`.
- `simulation-spec.md`, if required.
- `visual-spec.md`, if required.
- Component diagram(s), if required.
- `references.md`.
- Updated component index and relationship entries, if assigned.

Keep all files in the correct item folder. Submit the folder or repository link using the method specified by the coordinator. Do not submit only a screenshot or a link to an incomplete draft.

## 13. Final Submission Checklist

Before changing the status to **Submitted for Review**, confirm:

- [ ] I used the assigned title, category, and ID.
- [ ] I followed the relevant template and preserved its required sections.
- [ ] Important technical claims are supported by credible references.
- [ ] Facts, common patterns, proposals, and assumptions are clearly distinguished.
- [ ] Work is written in my own words.
- [ ] Diagrams are readable and agree with the written explanation.
- [ ] Data flows identify the important steps and interactions.
- [ ] Failure modes and trade-offs are explained where relevant.
- [ ] Simulation inputs, model, outputs, and experiments are specific enough to implement.
- [ ] Links and references work.
- [ ] The master index and relationships are updated if assigned.
- [ ] Files are saved in the correct folder with the expected names.
- [ ] I have checked the document for incomplete sections and obvious errors.

## 14. What Not to Do

- Do not invent internal architecture details for real companies.
- Do not present assumptions or estimates as verified facts.
- Do not copy source material and submit it as original writing.
- Do not use diagrams with unexplained boxes, arrows, or abbreviations.
- Do not describe a component only with a generic definition.
- Do not list technologies without explaining why they are used.
- Do not claim a simulation works when only a static UI or animation exists.
- Do not add unsupported performance numbers or cost figures.
- Do not change shared IDs, catalogue ordering, templates, or folder conventions without approval.
- Do not mark work approved without reviewer confirmation.

## 15. Questions and Blockers

If the assigned topic is unclear, a source is inaccessible, two sources appear to conflict, or the template does not fit the topic, record the issue and ask the coordinator before making a structural change.

When asking for help, include:

- The assigned item ID and title.
- The section or requirement causing the blocker.
- What you have already checked.
- The sources or alternatives you found.
- The specific decision or clarification you need.

The goal is to produce content that another developer, educator, or learner can trust and use. Prioritise correctness, clear reasoning, and reproducible explanations over volume.