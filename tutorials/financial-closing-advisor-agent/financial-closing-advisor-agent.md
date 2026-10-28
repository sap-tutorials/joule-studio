---
author_name: Samir Hamichi
author_profile: https://github.com/shamichi-repo
keywords: tutorial
auto_validation: true
time: 20
tags: [software-product>joule studio, joule, joule work, tutorial>beginner, tutorial>license]
primary_tag: software-product>joule studio
parser: v2
---

# Guide Financial Closing Activities with an Intelligent Closing Advisor Agent

## Prerequisites

- Access to **Joule Studio** 
- Access to an **SAP S/4HANA Cloud** system with:
  - **Financial Closing Cockpit** configured with a closing task list for the relevant period
  - Closing tasks defined with sequence numbers, dependencies, responsible owners, and statuses
  - Relevant financial posting transactions accessible (depreciation, accruals, GR/IR clearing, revaluation, cost allocations)
- Familiarity with period-end or year-end financial closing processes in SAP S/4HANA


## You will learn

- How to identify a financial closing workflow that benefits from intelligent status monitoring and sequenced recommendation
- How to write an effective intent statement for an agent 
- How to review and validate the assets created, including the generated **Product Requirements Document (PRD)**
- How Joule Studio generates a solution using **dedicated MCP servers** for the closing task list and underlying financial data
- How to deploy an agent that continuously monitors closing cycle progress and recommends the next closing activities to action, in the correct sequence, with clear reasoning


## Intro

>**IMPORTANT**
>
>**Welcome to the Agent lab**
>
>You are working with a pre-release version of the Joule Studio. This gives you an early look at our upcoming capabilities. Please keep the following in mind:
>
> - **Features are subject to change:** The UI, terminology, and functionality you see may differ from the final product.
> - **Educational use only:** This environment is designed for learning and experimentation, not for production use.
> - **Potential instability:** As a preview version, you may encounter occasional instability or unexpected behavior.

Every financial close is a race against time. Dozens of interdependent tasks - depreciation runs, foreign currency revaluations, GR/IR clearings, intercompany reconciliations, accruals, cost allocations, and more - must be completed in a defined sequence before the books can be locked. Miss one step, execute two tasks out of order, or fail to notice a blocked task until late in the cycle, and the entire close slips.

In this tutorial, you will build a **Financial Closing Advisor Agent** using Joule Studio. The agent connects to the SAP Financial Closing Cockpit to read the status of every task in the current closing cycle, evaluates the dependency graph and priority of the full task list, identifies what is completed, what is ready to start, and what is blocked - then recommends to the financial controller exactly which closing activities to action next, in the correct sequence, with the reasoning made explicit.

You will move through all phases of the Joule Studio's intent-based development process: from a plain-language intent statement to a deployed Python agent integrated with SAP S/4HANA via dedicated MCP servers.

> **About the scenario:** The closing tasks and sequences referenced in this tutorial are illustrative. The agent pattern applies to any organisation running a structured financial close in SAP S/4HANA with tasks managed in the Financial Closing Cockpit.

> **What makes financial closing a strong automation candidate?** The closing cycle has a well-defined trigger (period-end), a known and structured task list, deterministic dependency rules between tasks, and a clear success criterion (all tasks completed, period locked). Those characteristics make it an ideal target for a recommendation agent that can evaluate state and advise action, removing the need for a controller to mentally traverse the entire dependency graph under time pressure.


---

### Understand the Business Challenge

**Current pain points**

The manual coordination of closing cycle activities creates the following well-understood operational challenges:

- **No single status view** - Closing task status is distributed across multiple SAP Fiori apps, transactions, and sometimes external spreadsheets. Controllers must actively check each task individually to build a picture of overall progress.
- **Dependency tracking by memory** - Experienced controllers carry the dependency graph in their heads. When a task is blocked, the knock-on effects on downstream tasks must be mentally calculated- an error-prone process under time pressure.
- **Reactive bottleneck discovery** - Blocked or at-risk tasks are typically identified only when a downstream task fails to start, rather than when the blocking condition first appears. By then, the window for remediation is narrower.
- **No sequenced prioritisation** - When multiple tasks are actionable simultaneously, controllers must decide which to action first based on experience. There is no systematic evaluation of which sequence minimises close risk.
- **Fragmented communication** - When tasks are blocked by other teams (accounts payable, asset accounting, intercompany counterparts), the controller must initiate manual follow-up across email, chat, and phone - with no consolidated escalation trail.
- **Progress invisible to management** - Close cycle progress is not visible to Finance leadership without a manual status update from the controller, making real-time oversight difficult at the most critical point in the fiscal calendar.

**Three categories of closing task state**

At any point during the closing cycle, every task in the closing task list falls into one of three state categories.

| State category | Description | Example |
|----------------|-------------|---------|
| **Actionable** | Tasks that are ready to be executed - all predecessor tasks are complete and no blocking conditions exist. These are the agent's primary recommendations. | Foreign currency revaluation is ready because the GL period is open and all prior adjustments are posted. |
| **Blocked** | Tasks that cannot start because one or more predecessor tasks are incomplete, in error, or pending external input. These must be resolved before the cycle can progress. | Asset depreciation run is blocked because the asset master data change cutoff has not been confirmed by the asset accounting team. |
| **At-risk** | Tasks that are in progress or scheduled but are approaching their due time without being completed, or whose predecessor chain puts them at risk of missing the closing deadline. | Intercompany reconciliation is in progress but three company codes have unmatched items outstanding with less than 4 hours to the close deadline. |

**Typical closing task sequence**

A standard financial closing cycle includes tasks across multiple SAP modules that must be executed in a defined order. 

| Phase | Example tasks |
|-------|--------------|
| **Pre-close preparation** | Confirm posting period cutoff, validate open purchase orders and goods receipts, confirm asset master data freeze |
| **Operational close** | Post accruals and deferrals, run GR/IR account clearing, process recurring journal entries |
| **Asset and valuation** | Run asset depreciation, execute foreign currency revaluation, post loan and interest accruals |
| **Intercompany** | Reconcile intercompany balances, post elimination entries, confirm intercompany netting |
| **Allocations and distributions** | Run cost center allocations, execute profit center distributions, settle internal orders |
| **Reporting preparation** | Validate balance sheet accounts, run financial statement versions, confirm management reporting data |
| **Period lock** | Lock posting period for operational users, release period for reporting, archive closing documentation |

> The agent you build in this tutorial provides a live, consolidated view of the entire closing task list, evaluates the dependency graph automatically, flags blocked and at-risk tasks proactively, and recommends the next actions to take, in sequence and with reasoning, so the controller can focus on execution rather than coordination.


### Open Joule Studio and Define the Agent Intent

The **Intent** phase is the starting point of every agent in Joule Studio. You describe what you want the agent to do in plain business language. No technical specification required at this stage.



1. Open **Joule Work** and navigate to **Joule Studio** by selecting **<>** in the left navigation panel.

    ![Create solution](bis1.1.png)

2. Select **Create** to open the Create Agent dialog.

3. Complete the three fields in the dialog:

    - **Solution**: Select **New Solution** to create this agent as a standalone solution.
    - **Name**: Enter the following agent name:
        ```
        Financial Closing Advisor
        ```
    - **Intent Statement**: Enter the following statement in plain business language:
        ```
        Please create an agent. The agent should support financial controllers during the financial
        closing cycle by monitoring the status, priority, and sequence of all closing tasks and workflows. 
        It retrieves the current status of each closing activity from the SAP Financial Closing Cockpit, 
        identifies completed, in-progress, blocked, and pending tasks, evaluates the dependency chain and 
        priority of each task, and recommends to the controller the next financial closing activities to 
        be actioned - in the correct sequence and with clear reasoning. Where tasks are blocked by errors, 
        missing data, or incomplete upstream activities, the agent identifies the root cause and suggests 
        corrective actions. The agent supports configuration for closing calendar, task ownership, escalation 
        thresholds, and deadline proximity alerts.
        ```

    ![Create Agent](create_agent.png)

4. Select **Quick Create**. This adds *Fast Track* to the intent statement, which skips the clarifying questions and directly moves towards solution building.

5. Select **Create** to proceed.

> **Effective Intent Statement:**
> an effective intent statement clearly explains the agent's purpose, the data it uses, the decisions it helps make, and the constraints or rules it must follow. The example above addresses each of these four areas. If the way Joule Studio interprets your intent differs from what you intended, revisit this statement and refine it before moving forward.

> **Tip: Why sequencing and reasoning belong in the intent statement**
> The key differentiator of this agent is not that it reads task status - that is straightforward data retrieval. What makes it valuable is that it evaluates the dependency chain and recommends the *correct sequence* of next actions with *explicit reasoning*. Including both elements in the intent statement ensures Joule Studio generates a dependency evaluation engine alongside the data retrieval layer, rather than producing a simple status dashboard.



5. Confirm the business goals, and other questions, if any, to the best of your knowledge.

    ![Business goals and metrics](metrics.png)

    ![Business goals confirmation](confirm.png)

This request for confirmation comes even in fast track mode.



### Review the Intent

After submitting the intent, Joule Studio generates a file **"intent.md"** that reflects its interpretation of the solution for you. There is also a summary in the chat. You would validate each section carefully before proceeding to the Requirements phase if you had not chosen the fast track mode. Otherwise, you may iterate updating the intent by asking the assistant through the chat to do it, reflecting the changes in the solution.

1. Under **Solution Progress**, choose **Intent Summary**.

    ![Intent](intent.png)

    If you do not see the **Intent** in the middle section, make sure you are on **Develop**.

2. Verify that this matches your intent. If the summary is missing an element or describes a different scope, you can refine the intent statement and re-submit.    

3. Scroll down to the **Fit Gap Analysis** and see what Joule has identified.

    ![Fit-Gap Analysis](fit-gap.png)

4. Then you can scroll down to the **Recommended Solution**. It describes a Python-based AI agent that uses the Agent-to-Agent (A2A) and what it should be able to do.

5. At the bottom of the solution card, review the **Intent fit** score. A score of 90% or higher confirms a very high degree of alignment between the proposed solution and your original intent statement. If the score is lower, consider refining your intent statement.



### Product Requirements Document

In the **Requirements** phase, Joule Studio generates a full **Product Requirements Document (PRD)**. Review it carefully - it defines the dependency engine logic, automation boundaries, LLM scope, and operational guardrails for the agent.

1. Choose **Requirements** under **Solution Progress**. This opens the low-code view of the PRD.

    ![Requirements](requirements.png)


2. You can also access the requirements in the **File tree**. This gives the code view of the same document.

    ![Requirements file](requirements-file-tree.png)

3. Review the product requirement document and confirm that it accurately reflects your requirements.

#### Key sections and why they matter

- **Product Purpose & Value Proposition**

Defines the business problem, target users, expected benefits, and strategic objectives. It explains why the solution should exist and the value it is expected to deliver.

- **Business Metrics**

Defines the measurable success criteria for the solution. It establishes the KPIs that will be used to evaluate business impact and adoption.

- **Requirements**

Captures the functional capabilities the solution must provide. User stories, acceptance criteria, and priorities ensure the development team understands what needs to be delivered.

- **Solution Architecture**

Describes the high-level technical design, including the agent, integrations, tools, APIs, and system components required to implement the solution.

- **Agent Extensibility & Instrumentation**

Explains how the solution can evolve over time and how operational events will be logged and monitored to support maintenance, observability, and future enhancements.

- **Automation & Agent Behaviour**

Defines how the agent operates, what actions it can perform autonomously, where human approval is required, which data sources it uses, and the guardrails that govern its behavior.

- **Milestones**

Defines the key business outcomes and execution checkpoints the agent must achieve. Milestones provide traceability, validation criteria, and operational visibility into the agent's performance.



### Inspect the Generated Specification

In the **Specification** phase, Joule Studio translates the PRD into a generated file tree - the complete technical blueprint for the agent. No artifacts are written yet.

1. Choose **Specifications** under **Solution Progress**.

    ![Specification](specification.png)

> **Architecture note:** The Financial Closing Advisor Agent uses **two dedicated MCP servers** with different roles. The first as a primary data source - it provides the task list, statuses, sequences, and dependencies - and is also write-enabled for task status updates upon controller confirmation. The second read-only is used for root cause analysis.

2. Select any file in the tree to inspect its contents in the right-hand panel. 

    ![Specification](specification-file-tree.png)

    The entire specification is transparent and reviewable before any solution implementation is triggered.


### Generate the Solution

During the generation of the Solution, Joule Studio executes the specification and generates the complete, runnable solution - the dependency graph engine, the root cause analyser, the two MCP server integrations, the recommendation narration, and the task confirmation flow - with no manual development required.

1. Ask Joule to build the solution by entering **execute specification/specification.md** in the chat. It will also understand more human readable commands like **Build the solution**.

> If Joule seems to be inactive at some point during this phase, you could ask something like **What is the status?** Joule would then tell you, and you would then command it to continue.

2. Wait for the solution generation to complete.

    ![Solution](what_was_built.png)

Joule Studio performs the following during this phase:

- Scaffolds the full Python agent with the dependency graph engine, root cause analyser, and LLM narration prompts from the `assets/` tree.
- Wires up the MCP servers
- Configures the **SAP Generative AI Hub** connection for LLM-based recommendation narration and conversational guidance.
- Implements the deadline proximity scoring logic and the escalation alert mechanism.
- Applies the four operational guardrails, including the predecessor validation check and the stale status protection.
- Instruments all agent actions with audit logging.

3. Select the agent under **Solution Artifacts** to open its definition. 

    ![Solution](solution-agent.png)

In the low-code view you can see:

- Tools and Skills, with the MCP Servers that allow the agent to use services, and the skills that have already been created
- Evaluation where you can add and generate evaluation scenarios

4. To see the generated Python code, select **agent.py** under the **File tree**. 

    ![Agent code](agent-python.png)

Once the solution is built, you can proceed to the **Testing Overview** to see the automated test results.


### Validate the Agent with Automated Tests

The Testing phase is triggered automatically by Joule once the solution build is complete. It runs an automated validation suite derived from the specification - you do not need to initiate it manually. Wait for Joule to confirm that testing has finished before proceeding.

![Testing Overview](what-was-built-tests.png)

Verify that all tests pass before proceeding to deployment.

> If any test fails, Joule will usually try to fix it automatically. If it gets stuck, you might converse with Joule to help fix the problem.

1. Open **Testing Overview**.

    ![Testing Overview](testing-overview-new.png)


2. Select the agent under **Preview**.

    ![Solution Preview](solution-preview-new.png)

    You can do some further testing here before deployment. Since the solution is not yet deployed, mock data will be used and not everything is testable. You can try something like "How can I test this agent?" or "What are you responsible for?" to start the testing process.

### Deploy the Agent to Development Landscape

With all tests passed and the validation score confirmed, the solution is ready for deployment.

1. Choose **Deploy**. Then confirm with **Deploy** again.

    ![Deploy](deploy_with_system.png)


Joule Studio packages the agent and both MCP servers and deploys them to the **SAP managed service** on BAIP. Once deployment completes, `deploy_result.json` in the `specification/` folder is populated with the live endpoint URL, runtime ID, and deployment timestamp.

Your agent is now operational and available to financial controllers.

> If you are working in a team or want to version-control the generated code, Joule Studio supports GitHub sync. You can find this option under Actions in the top bar.

> For governance reasons, deployment to production is not done from within Joule Studio.

**How the agent works in production**

Once deployed, agents are automatically discoverable through Joule. When users ask questions that match the agent’s area of responsibility, Joule routes the request to the agent. Users don’t need to know that a custom agent exists or how to find it.

**Monitoring the Agent**

Once deployed, switch to the **Manage** tab in Joule Studio to access governance and monitoring. Here you can view runtime status, active deployments, usage metrics, and audit logs.




### Summary

You have completed the end-to-end design and deployment of a Financial Closing Advisor Agent using SAP Joule Studio. In doing so, you have:

- Identified the three closing task state categories (actionable, blocked, at-risk), the typical closing task sequence across preparation, operational close, valuation, intercompany, allocations, reporting, and period lock phases, and the six manual pain points - fragmented status view, dependency tracking by memory, reactive bottleneck discovery, unstructured prioritisation, fragmented escalation, and invisible progress - that the agent addresses
- Written an intent statement that explicitly captures both the status monitoring objective and the sequenced recommendation with reasoning - ensuring Joule Studio generates a dependency graph engine rather than a simple status reader
- Reviewed and validated an Idea Board with five measurable goals spanning task retrieval, dependency evaluation, bottleneck identification, recommendation generation, and task execution support
- Evaluated a generated PRD defining three product objectives, a hybrid automation level, explicit engine vs. LLM boundaries with the key principle that the LLM narrates but never independently assesses task state, and four operational guardrails including predecessor validation completeness and stale data protection
- Inspected a generated file tree with two MCP servers - a read/write Closing Cockpit server scoped to controller-confirmed write operations and a read-only Financial Data server scoped to root cause analysis- with the configuration parameters that govern dependency evaluation, deadline thresholds, and escalation behaviour
- Reviewed a comprehensive test suite covering dependency traversal for linear and parallel chains, blocked and at-risk classification, root cause analysis, deadline escalation, confirmation gate enforcement, narration quality, and stale data detection
- Deployed the agent to the SAP managed service and understood its full production interaction sequence - from task list retrieval and dependency graph traversal through root cause analysis, prioritised recommendation, conversational follow-up, task confirmation, and continuous refresh

The agent transforms the financial close from a manually coordinated, memory-dependent, reactively managed process into a **continuously monitored, intelligently sequenced, and fully auditable workflow** - giving controllers a live action plan at every moment of the close cycle and giving Finance leadership the visibility they need without requiring manual status updates.
