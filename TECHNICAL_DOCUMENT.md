# Technical Documentation

## 1. Introduction

This document provides a comprehensive technical overview of an automated browser interaction system. The system is designed to record user actions in a web browser, transform these recordings into structured workflows, and then execute these workflows, leveraging Artificial Intelligence (AI) to enhance both the creation and execution processes.

The purpose of this technical documentation is to detail:
*   The core interactions with AI (Large Language Models - LLMs) and the strategies behind them.
*   The prompt engineering techniques employed to guide LLM behavior for workflow generation and execution assistance.
*   The implementation details of the workflow lifecycle, from recording to execution.
*   The critical API endpoints and data type definitions that enable communication between system components.

Architecturally, the system comprises three main components:

1.  **Browser Extension (`extension/`)**: Captures user interactions within the browser (clicks, typing, navigation, scrolls, etc.), including DOM information and optional screenshots. It sends this recorded data to the backend for processing.
2.  **Backend (`workflows/`)**:
    *   Processes the raw recordings from the extension. A key feature is the `BuilderService`, which uses LLMs to analyze recordings and user goals, automatically generating structured and often parameterized workflow definitions (`WorkflowDefinitionSchema`).
    *   Stores these workflow definitions persistently (e.g., as JSON files).
    *   Executes the stored workflows. The `Workflow` service manages this, capable of running predefined deterministic browser actions and delegating more complex or dynamic tasks to AI-driven agents.
3.  **Web UI (`ui/`)**: A React-based single-page application that allows users to:
    *   Manage and list available workflows.
    *   Visualize workflow structures as interactive graphs.
    *   Edit workflow metadata (like name, description, input parameters) and individual step configurations.
    *   Trigger the execution of workflows and monitor their progress and logs.

A central theme of this system is the integration of AI. LLMs are not just an add-on but a core part of the architecture, used to elevate simple recordings into more robust, flexible, and intelligent automation scripts. This includes generating workflows that can adapt to dynamic content via agentic steps and providing mechanisms for AI-assisted error recovery during execution.

## 2. AI Interaction

Artificial Intelligence, primarily through Large Language Models (LLMs), is deeply integrated into several key areas of the system to enable automation, dynamic interaction, and intelligent data processing.

### Workflow Generation (`BuilderService`)

The `BuilderService` (in `workflows/workflow_use/builder/service.py`) is a core AI-driven component responsible for creating executable workflows from raw browser recordings.
*   **Interpretation of Recordings**: The LLM, guided by `WORKFLOW_BUILDER_PROMPT_TEMPLATE`, interprets a sequence of recorded browser events (clicks, inputs, navigations) and an optional high-level user goal.
*   **Deterministic vs. Agentic Steps**: A crucial role of the LLM is to decide when a recorded action can be a reliable deterministic step (e.g., clicking a static button) versus when it requires an "agentic" step (`"type": "agent"`). Agentic steps are chosen for tasks involving dynamic content, user choices from variable lists, or interactions with elements whose selectors might change frequently.
*   **Input Schema Definition**: The LLM analyzes the workflow and user goal to define an `input_schema`. This schema specifies the dynamic inputs the workflow will accept (e.g., a search term, a username). This makes workflows reusable and adaptable.
*   **Visual Context (Screenshots)**: If enabled, the `BuilderService` can pass screenshots associated with recorded steps to a vision-capable LLM. This visual context helps the LLM better understand the UI elements and the user's actions, leading to more accurate workflow generation.

### Workflow Execution (`Workflow` service & `Agent`)

AI plays a significant role during the execution of workflows, particularly for handling dynamic situations and extracting data. This is primarily managed by the `Workflow` service (`workflows/workflow_use/workflow/service.py`) and the `Agent` class (from the `browser_use` library).

*   **Agentic Steps**: When a workflow encounters a step of `type: "agent"`, the `Workflow` service delegates control to an `Agent`.
    *   The `Agent` uses an LLM to understand the `task` defined in the step (e.g., "Select the cheapest flight from the results").
    *   The LLM then iteratively chooses actions (like clicking, typing, scrolling, implemented via the `WorkflowController`) to interact with the live webpage in the browser, aiming to accomplish the given `task`.
*   **Error Fallback**: If a deterministic step (e.g., `click`, `input`) fails during execution, the `Workflow` service can invoke an LLM using the `WORKFLOW_FALLBACK_PROMPT_TEMPLATE`.
    *   The LLM is given context about the failed step, the overall workflow, and the error.
    *   It then attempts to achieve the original step's objective using a different approach, effectively providing AI-driven error recovery.
*   **Structured Data Extraction**:
    *   **`extract_page_content` Step Type**: Workflows can include steps specifically designed to extract information from a webpage. This step type likely uses an LLM (potentially with `STRUCTURED_OUTPUT_PROMPT`) to parse the page's content and retrieve the desired data based on a `goal`.
    *   **End-of-Workflow Output Conversion**: The `Workflow.run()` method can take an `output_model` (a Pydantic model). If provided, all `extracted_content` accumulated during the workflow run is passed to an LLM (using `STRUCTURED_OUTPUT_PROMPT`) to parse and fit this data into the specified structured `output_model`.

### LangChain Tool Integration

The system allows workflows themselves to be treated as tools within the LangChain framework, enabling more complex AI agent setups.
*   **`Workflow.as_tool()`**: This method in the `Workflow` service converts an entire workflow into a LangChain `StructuredTool`. The tool's input schema is dynamically generated from the workflow's `input_schema`.
*   **`Workflow.run_as_tool()`**: This method allows invoking the workflow using natural language. It wraps the workflow tool with an agent executor (like `create_tool_calling_agent`) which can understand a natural language prompt, extract the necessary inputs for the workflow tool, execute it, and return the result.

### User-Triggered AI Processes

The user interface facilitates interaction with these AI-driven processes:
*   **Workflow Execution**: Users can initiate the execution of workflows (which may have been generated by an LLM and/or contain AI-driven agentic steps) through the UI (`PlayButton`).
*   **Dynamic Input Provision**: The UI presents users with input fields based on the `input_schema` (which was potentially defined by an LLM during workflow building). This allows users to provide dynamic data to these AI-augmented processes.
*   **Observation and Management**: Users can observe the logs of these executions and manage them (e.g., cancel) via the `LogViewer`.

## 3. Prompt Engineering

Prompt engineering is crucial for guiding Large Language Models (LLMs) to perform specific tasks accurately and generate desired outputs. This system utilizes several focused prompts for different AI-driven functionalities. These are primarily defined in `workflows/workflow_use/builder/prompts.py` and `workflows/workflow_use/workflow/prompts.py`.

### `WORKFLOW_BUILDER_PROMPT_TEMPLATE`

*   **Purpose**: This prompt is designed to instruct an LLM to convert a JSON recording of browser events into a structured, executable JSON workflow definition that the system's runtime can directly consume.
*   **LLM Inputs**:
    *   The primary input is the sequence of recorded browser events, provided as individual JSON objects in subsequent messages. Screenshots can accompany these events if `use_screenshots` is enabled in `BuilderService`.
    *   A `user_goal` (a high-level description of the workflow's purpose) is also provided to the LLM.
    *   A list of `actions` available to the workflow, along with their parameters.
*   **Key Output Structure (`WorkflowDefinitionSchema`)**: The LLM is instructed to generate a JSON object with the following top-level keys:
    *   `workflow_analysis`: An initial analysis of the original recording, its purpose, and potential variables needed.
    *   `name`: A descriptive name for the workflow.
    *   `description`: A human-readable description.
    *   `input_schema`: An array defining the dynamic inputs the workflow will accept. This must follow a subset of JSON-Schema draft-7 semantics (e.g., `[{"name": "query", "type": "string", "required": true}]`). The prompt emphasizes creating relevant inputs based on the user goal or event parameters, rather than defaulting to an empty schema.
    *   `steps`: An ordered array of step objects to be executed.
    *   `version`: A version string for the workflow.
*   **Rules for `input_schema` Generation**:
    *   The LLM should aim to include at least one input unless the workflow is entirely static.
    *   Inputs should be based on the user's goal, parameters from recorded events (like search terms or form data), or values that could be reused.
    *   An empty `input_schema` requires justification in the `workflow_analysis`.
*   **Rules for `steps` Array**:
    *   Each step object MUST include a `type` field.
    *   **Agentic Steps (`"type": "agent"`)**:
        *   Used for tasks requiring interaction with dynamic content or user choices (e.g., selecting from changing search results, picking a date from a calendar).
        *   Must include a `task` string describing the user's goal for that step (e.g., "Select the product named {{product_name}}").
        *   Should include a `description` explaining why an agent is needed.
        *   Optionally, `max_steps` can limit agent exploration.
        *   The prompt guides the LLM to replace deterministic steps with agentic ones when dealing with frequently changing lists, time-sensitive elements, or content evaluation based on user input.
        *   Complex tasks should be broken into multiple, specific agentic steps.
    *   **`extract_page_content` Steps**: This type is preferred over agentic steps for simple data extraction from a page.
    *   **Deterministic Steps**: For other actions, the LLM should generally retain the original recorder event structure. The `type` must match an available action name, and other keys are parameters.
    *   Each step should have a short `description` of its objective.
    *   The LLM is advised to consider if navigating directly to a URL can bypass initial click steps (e.g., for a search).
*   **Placeholder Syntax**: When referencing workflow inputs within step parameters or agent tasks, the syntax `{{input_name}}` must be used (e.g., `"value": "{{search_term}}"`). Placeholders must be quoted to be valid JSON strings.
*   **Selector Replication**: The LLM should replicate all selectors provided in the original event data for each action.

### `WORKFLOW_FALLBACK_PROMPT_TEMPLATE`

*   **Purpose**: This prompt is used when a deterministic step in a workflow fails during execution. It instructs an LLM to take over and attempt to complete only that specific failed step using AI-driven browser interaction.
*   **Context Provided to LLM**:
    *   `step_index` and `total_steps`: Information about the failed step's position.
    *   `workflow_details`: An overview of the entire workflow (currently commented out in the template: `"{workflow_details}\n\n"`).
    *   `action_type`: The type of the action that failed (e.g., "click", "input").
    *   `fail_details`: Specifics about the failure, including the step's original parameters and the error message.
    *   `failed_value`: The intended target or value for the failed step (e.g., URL for navigation, text for input).
    *   `step_description`: The original description of the failed step's purpose.
*   **Key Instructions**:
    *   **Focus**: The LLM's primary task is to complete *only* the objective of the single failed step.
    *   **Retry Strategy**: Do not retry the exact same action that failed. Instead, choose different suitable actions to achieve the same goal.
    *   **Minimal Action**: Perform only the minimum actions necessary for the current step. Do not proceed to subsequent steps or perform extraneous actions.
    *   **Completion**: Once the objective of the current failed step is reached, call the "Done" action.

### `STRUCTURED_OUTPUT_PROMPT`

*   **Purpose**: This is a general-purpose prompt for extracting structured information from provided text content into a predefined Pydantic schema.
*   **Usage Context**:
    *   It's likely used by the `extract_page_content` step type within a workflow, where the goal is to pull specific data from a webpage.
    *   It's also used by the `_convert_results_to_output_model` method in the `Workflow` service (`workflows/workflow_use/workflow/service.py`). This method takes all `extracted_content` from various step results and uses this prompt to parse that combined text into a user-specified Pydantic `output_model` at the end of a workflow run.
*   **Key Instructions**:
    *   The LLM is positioned as a "data extraction expert."
    *   It must analyze the provided content and extract information according to a schema (which would be implicitly provided by LangChain's structured output mechanisms based on the target Pydantic model).
    *   Only extract information explicitly present in the content.
    *   Follow the schema precisely.

## 4. Workflow Implementation

This section outlines the lifecycle of a workflow, from its initial recording by the user to its execution by the backend, and how the UI facilitates these processes.

### Conceptual Workflow Lifecycle

1.  **Recording (Browser Extension)**:
    *   User actions (clicks, typing, navigation, scrolls) are captured by `extension/src/entrypoints/content.ts`.
    *   `content.ts` utilizes `rrweb` for capturing DOM events (like scrolls, though it also captures full snapshots and incremental DOM changes which are filtered) and custom event listeners for more specific interactions (click, input, key press, select change).
    *   It generates `XPath` and enhanced `CSSSelector` for targeted elements.
    *   These raw events, including screenshots for custom events (captured in `background.ts` via `chrome.tabs.captureVisibleTab`), are sent to `extension/src/entrypoints/background.ts`.
    *   `background.ts` stores these events in-memory (`sessionLogs`). It converts `StoredEvent` objects (which include `messageType`, `timestamp`, `tabId`, and event-specific payloads like `xpath`, `cssSelector`, `value`, `key`, `screenshot`) into a sequence of `Step` objects (defined in `extension/src/lib/workflow-types.ts`).
    *   This sequence of `Step` objects is assembled into a `Workflow` object (also from `extension/src/lib/workflow-types.ts`), which includes `name`, `description`, `version`, `input_schema` (initially empty), and the `steps`.
    *   The `Workflow` object is then sent to the backend's `/event` endpoint as the payload of an `HttpWorkflowUpdateEvent` upon changes. `HttpRecordingStartedEvent` and `HttpRecordingStoppedEvent` are also sent to this endpoint.

2.  **Building (Backend `BuilderService`)**:
    *   The backend's `/event` endpoint (likely handled by `workflows/backend/api.py` or a similar FastAPI route, though not explicitly detailed in provided files) receives the `HttpWorkflowUpdateEvent` from the extension. This event contains the recorded `Workflow` data.
    *   This data is then likely passed to the `BuilderService` (defined in `workflows/workflow_use/builder/service.py`).
    *   The `BuilderService.build_workflow` method takes this input workflow (which is a `WorkflowDefinitionSchema` but the extension sends a simpler `Workflow` type, so an implicit conversion or mapping must occur) and a `user_goal`.
    *   It uses an LLM (e.g., a vision-capable model if `use_screenshots` is true) with a detailed prompt template (`WORKFLOW_BUILDER_PROMPT_TEMPLATE` from `workflows/workflow_use/builder/prompts.py`).
    *   The prompt instructs the LLM to analyze the recorded steps, infer intent, define an `input_schema` for dynamic values, and decide between deterministic and agentic steps. It provides the LLM with a list of available actions and their schemas.
    *   The LLM's output is expected to be a JSON conforming to `WorkflowDefinitionSchema` (from `workflows/workflow_use/schema/views.py`). This schema includes `workflow_analysis`, refined `steps` (potentially converting some to `agent` type or adding `extract_page_content`), and the `input_schema`.
    *   The `BuilderService` can parse the LLM's string output (potentially from a markdown block) into the `WorkflowDefinitionSchema`.

3.  **Storage**:
    *   The generated `WorkflowDefinitionSchema` instances are stored as JSON files in a directory managed by `WorkflowService` (e.g., `workflows/examples/` or a configured temporary/data directory). The `WorkflowService` in `workflows/backend/service.py` lists, retrieves, and updates these files.

4.  **Execution (Backend `Workflow` service)**:
    *   When a workflow execution is requested (e.g., via `POST /api/workflows/execute`), the `WorkflowService` in `workflows/backend/service.py` loads the corresponding `WorkflowDefinitionSchema` JSON file.
    *   An instance of the `Workflow` class (from `workflows/workflow_use/workflow/service.py`) is created with this schema.
    *   The `Workflow.run()` method iterates through the `steps`:
        *   Placeholders in step parameters (e.g., `{{input_name}}`) are resolved using the current `context` (populated from workflow inputs).
        *   **Deterministic Steps** (e.g., `ClickStep`, `NavigationStep`): Executed by the `WorkflowController` which interacts with the `Browser` object (using Playwright). The `controller` uses the step's `type` and parameters. It waits for elements based on CSS selectors or XPath.
        *   **Agentic Steps** (`AgentTaskWorkflowStep`): An `Agent` (from `browser_use` library) is invoked with the specified `task` and `max_steps`. The agent uses an LLM and browser tools to accomplish the task.
        *   **Error Fallback**: If a deterministic step fails and `fallback_to_agent` is true, the `Workflow` service can construct a new agent task using `WORKFLOW_FALLBACK_PROMPT_TEMPLATE` (from `workflows/workflow_use/workflow/prompts.py`) to try and recover or complete the step's objective.
        *   **Output Handling**: The output of a step (e.g., extracted data) can be stored in the `context` if the step's `output` field is set, making it available for subsequent steps.
        *   **Structured Output**: If an `output_model` is provided to `Workflow.run()`, the collected results (especially `extracted_content` from `ActionResult`) are passed to an LLM with `STRUCTURED_OUTPUT_PROMPT` to fit them into the desired Pydantic model.

### User Interface (`ui/`) Components and Interaction

The UI, built with React and Vite, provides tools for visualizing, managing, and interacting with workflows.

*   **Visualization (`WorkflowLayout`, `jsonToFlow`)**:
    *   Workflows selected from the `Sidebar` are loaded via the backend API.
    *   `ui/src/utils/json-to-flow.ts` (`jsonToFlow` function) converts the workflow JSON (conforming to `WorkflowDefinitionSchema` from backend, parsed as `Workflow` type in UI via Zod `workflowSchema`) into a format suitable for `ReactFlow`.
    *   `ui/src/components/workflow-layout.tsx` uses `ReactFlow` to render the nodes (steps) and edges (connections) of the workflow. Node positions can be saved locally.
*   **Management (`Sidebar`, `WorkflowItem`)**:
    *   `ui/src/components/sidebar.tsx` lists available workflows fetched from `GET /api/workflows`.
    *   `ui/src/components/workflow-item.tsx` displays individual workflow names and versions. It allows selecting a workflow to view its details and graph.
*   **Metadata Editing (`Sidebar`, `WorkflowItem`)**:
    *   When a workflow is selected, its metadata (`name`, `description`, `version`, `input_schema`, `workflow_analysis`) is displayed in the `Sidebar` within the `WorkflowItem` component.
    *   An "Edit" button allows modification of these fields.
    *   Saving sends the updated metadata to `POST /api/workflows/update-metadata`.
*   **Step Editing (`NodeConfigMenu`)**:
    *   Clicking a node in the `ReactFlow` graph opens `ui/src/components/node-config-menu.tsx`.
    *   This menu displays the details of the selected step (`StepData`).
    *   It allows editing fields like `description`, `type`, `url` (for navigation), `cssSelector`, `xpath`, `value` (for input), etc.
    *   Changes are saved by sending a request to `POST /api/workflows/update` with the `workflowFilename`, `nodeId` (step index), and the modified `stepData`.
*   **Execution (from UI - `PlayButton`, `LogViewer`)**:
    *   The `PlayButton` component (in `ui/src/components/play-button.tsx`) handles the initiation of workflow execution.
        *   When clicked, it opens a modal (`showModal`) to collect input parameters.
        *   Input fields are dynamically generated based on the `workflowMetadata.input_schema` of the selected workflow, supporting types like string, number, and boolean. Required fields are marked.
        *   Upon submission, it calls the `POST /api/workflows/execute` backend endpoint with the workflow name and the collected `inputs`.
    *   If execution starts successfully, a `taskId` and initial `logPosition` are received from the backend.
    *   The `LogViewer` component (in `ui/src/components/log-viewer.tsx`) is then displayed (`showLogViewer`).
        *   It polls the `GET /api/workflows/logs/{task_id}` endpoint periodically (every 2 seconds) using the `taskId` and current `logPosition` to fetch new log entries.
        *   It displays the incoming logs and updates the workflow's `status` (e.g., "running", "completed", "failed", "cancelled").
        *   If the workflow is "running", a "Cancel" button is available. Clicking it calls `POST /api/workflows/tasks/{task_id}/cancel`.
        *   Logs can be downloaded as a text file.
        *   The viewer closes when the workflow finishes or the user manually closes it.

## 5. API & Type Definitions

This section details the various API endpoints, data schemas, and type definitions used across the system, enabling communication and data consistency between the browser extension, backend, and UI.

### Backend API Endpoints

The backend exposes several RESTful endpoints for managing and executing workflows. These are defined in `workflows/backend/routers.py` and their request/response models are in `workflows/backend/views.py`.

*   **`GET /api/workflows`**:
    *   Description: Lists all available workflow definition files.
    *   Response Model (`WorkflowListResponse`):
        *   `workflows`: `List[str]` - A list of workflow names (filenames).
*   **`GET /api/workflows/{name}`**:
    *   Description: Retrieves the JSON content of a specific workflow definition.
    *   Path Parameter: `name` (str) - The name of the workflow file.
    *   Response Model: `str` (raw JSON content of the workflow file).
*   **`POST /api/workflows/update`**:
    *   Description: Updates a specific node (step) within a workflow definition file.
    *   Request Model (`WorkflowUpdateRequest`):
        *   `filename`: `str` - The name of the workflow file to update.
        *   `nodeId`: `int` - The ID (index) of the step to update.
        *   `stepData`: `Dict[str, Any]` - The new data for the step.
    *   Response Model (`WorkflowResponse`):
        *   `success`: `bool` - Indicates if the update was successful.
        *   `error`: `Optional[str]` - Error message if the update failed.
*   **`POST /api/workflows/update-metadata`**:
    *   Description: Updates the metadata (name, description, version, input_schema, workflow_analysis) of a workflow.
    *   Request Model (`WorkflowMetadataUpdateRequest`):
        *   `name`: `str` - The current name of the workflow file.
        *   `metadata`: `Dict[str, Any]` - The new metadata for the workflow.
    *   Response Model (`WorkflowResponse`):
        *   `success`: `bool`
        *   `error`: `Optional[str]`
*   **`POST /api/workflows/execute`**:
    *   Description: Initiates the execution of a specified workflow.
    *   Request Model (`WorkflowExecuteRequest`):
        *   `name`: `str` - The name of the workflow to execute.
        *   `inputs`: `Dict[str, Any]` - Input values for the workflow.
    *   Response Model (`WorkflowExecuteResponse`):
        *   `success`: `bool` - Indicates if execution started successfully.
        *   `task_id`: `str` - A unique ID for the execution task.
        *   `workflow`: `str` - The name of the executed workflow.
        *   `log_position`: `int` - Initial position in the log file.
        *   `message`: `str` - Confirmation message.
*   **`GET /api/workflows/logs/{task_id}`**:
    *   Description: Retrieves logs for a specific workflow execution task.
    *   Path Parameter: `task_id` (str).
    *   Query Parameter: `position` (int, optional) - Starting position to read logs from.
    *   Response Model (`WorkflowLogsResponse`):
        *   `logs`: `List[str]` - List of log lines.
        *   `position`: `int` - New log position after reading.
        *   `log_position`: `int` (duplicate of `position`).
        *   `status`: `str` - Current status of the task.
        *   `result`: `Optional[List[Dict[str, Any]]]` - Final result if completed.
        *   `error`: `Optional[str]` - Error message if task failed.
*   **`GET /api/workflows/tasks/{task_id}/status`**:
    *   Description: Gets the status of a specific workflow execution task.
    *   Path Parameter: `task_id` (str).
    *   Response Model (`WorkflowStatusResponse`):
        *   `task_id`: `str`
        *   `status`: `str`
        *   `workflow`: `str` - Name of the workflow.
        *   `result`: `Optional[List[Dict[str, Any]]]`
        *   `error`: `Optional[str]`
*   **`POST /api/workflows/tasks/{task_id}/cancel`**:
    *   Description: Cancels an ongoing workflow execution task.
    *   Path Parameter: `task_id` (str).
    *   Response Model (`WorkflowCancelResponse`):
        *   `success`: `bool`
        *   `message`: `str`

### Core Workflow Schemas

Defined in `workflows/workflow_use/schema/views.py`, these Pydantic models form the backbone of workflow definitions.

*   **`WorkflowDefinitionSchema`**: The root model for a workflow file.
    *   `name`: `str` - The workflow's name.
    *   `description`: `str` - Human-readable description.
    *   `version`: `str` - Version identifier.
    *   `input_schema`: `List[WorkflowInputSchemaDefinition]` - Defines expected inputs.
    *   `steps`: `List[WorkflowStep]` - Ordered list of steps to execute.
    *   `workflow_analysis`: `Optional[str]` - LLM-generated analysis of the recorded workflow.
*   **`WorkflowStep`**: A Union of all possible step types.
    *   **Deterministic Steps**: Inherit from `TimestampedWorkflowStep` (includes `timestamp`, `tabId`).
        *   `NavigationStep`: (`type: "navigation"`) - Navigates to a URL.
            *   `url`: `str`
        *   `ClickStep`: (`type: "click"`) - Clicks an element.
            *   `cssSelector`: `str`
            *   `xpath`, `elementTag`, `elementText`: `Optional[str]`
        *   `InputStep`: (`type: "input"`) - Inputs text into an element.
            *   `cssSelector`: `str`
            *   `value`: `str`
            *   `xpath`, `elementTag`: `Optional[str]`
        *   `SelectChangeStep`: (`type: "select_change"`) - Changes the value of a select element.
            *   `cssSelector`: `str`
            *   `selectedText`: `str`
            *   `xpath`, `elementTag`: `Optional[str]`
        *   `KeyPressStep`: (`type: "key_press"`) - Simulates a key press.
            *   `cssSelector`: `str`
            *   `key`: `str`
            *   `xpath`, `elementTag`: `Optional[str]`
        *   `ScrollStep`: (`type: "scroll"`) - Scrolls the page.
            *   `scrollX`: `int`
            *   `scrollY`: `int`
        *   `PageExtractionStep`: (`type: "extract_page_content"`) - Extracts content from the page.
            *   `goal`: `str`
    *   **Agentic Step**:
        *   `AgentTaskWorkflowStep`: (`type: "agent"`) - Delegates a task to an AI agent.
            *   `task`: `str` - The objective for the agent.
            *   `max_steps`: `Optional[int]` - Max iterations for the agent.
    *   Common fields for all steps (from `BaseWorkflowStep`):
        *   `description`: `Optional[str]`
        *   `output`: `Optional[str]` (Context key to store step output)
*   **`WorkflowInputSchemaDefinition`**: Defines an input parameter for a workflow.
    *   `name`: `str` - Name of the property (used as key in input schema).
    *   `type`: `Literal['string', 'number', 'bool']`
    *   `required`: `Optional[bool]`

### Extension-to-Backend Communication

The browser extension communicates with the backend primarily to send recorded events and workflow updates. This is managed in `extension/src/entrypoints/background.ts` using types from `extension/src/lib/message-bus-types.ts`.

*   **Endpoint**: `http://127.0.0.1:7331/event`
    *   The `background.ts` script sends `POST` requests to this local server endpoint.
*   **`HttpEvent` Types**: This is a union of possible event structures sent to the backend.
    *   `HttpWorkflowUpdateEvent`:
        *   `type`: `"WORKFLOW_UPDATE"`
        *   `timestamp`: `number`
        *   `payload`: `Workflow` (defined in `extension/src/lib/workflow-types.ts`) - Contains the current state of the recorded workflow including all steps.
            *   The `Workflow` type in `workflow-types.ts` has `steps`, `name`, `description`, `version`, and `input_schema`.
            *   `Step` types within this (`NavigationStep`, `ClickStep`, `InputStep`, `KeyPressStep`, `ScrollStep`) are simpler representations compared to the backend's `WorkflowDefinitionSchema`, primarily containing raw event data like `url`, `xpath`, `cssSelector`, `value`, `key`, `scrollX`, `scrollY`, `screenshot` (optional base64 string).
    *   `HttpRecordingStartedEvent`:
        *   `type`: `"RECORDING_STARTED"`
        *   `timestamp`: `number`
        *   `payload`: `{ message: string }`
    *   `HttpRecordingStoppedEvent`:
        *   `type`: `"RECORDING_STOPPED"`
        *   `timestamp`: `number`
        *   `payload`: `{ message: string }`

### Frontend API Client & Types

The UI interacts with the backend API using a generated client and Zod-based type definitions.

*   **API Client Setup** (from `ui/src/lib/api/index.ts`):
    *   Uses `openapi-fetch` for making API requests and `openapi-react-query` for integrating with TanStack Query.
    *   Base URL: `http://localhost:8000` (configurable).
    *   Types for API paths and models are generated by `openapi-typescript` into `ui/src/lib/api/apigen.ts` (not directly analyzed but its presence is noted).
*   **Frontend Workflow Types** (primarily from `ui/src/types/workflow-layout.types.ts`):
    *   **`workflowSchema` (Zod)**: Defines the structure of a workflow for the UI. This schema is largely compatible with the backend's `WorkflowDefinitionSchema` but is defined in Zod for frontend validation.
        *   `workflow_analysis`: `z.string()`
        *   `name`: `z.string()`
        *   `description`: `z.string()`
        *   `version`: `z.string()`
        *   `steps`: `z.array(stepSchema)`
            *   `stepSchema`: Zod object for individual steps ( `description`, `output`, `timestamp`, `tabId`, `type`, and optional fields like `url`, `cssSelector`, etc.). Types are simpler enums e.g. `type: z.enum(['navigation', 'click', 'select_change', 'input'])`.
        *   `input_schema`: `z.array(inputFieldSchema)`
            *   `inputFieldSchema`: Zod object for input fields (`name`, `type`, `required`).
    *   **`Workflow` (TypeScript)**: Inferred type from `workflowSchema`.
    *   **`WorkflowMetadata`**: Interface for essential workflow metadata used in listings and editing.
        *   `name`, `description`, `version`: `string`
        *   `input_schema`: `any[]`
        *   `workflow_analysis`: `Optional[string]`
    *   **`StepData`** (from `ui/src/types/node-config-menu.types.ts` but mirrors `WorkflowStep` from `workflow-layout.types.ts`): Used for node configuration, representing the data within a single step.
        *   Includes fields like `description`, `type`, and other optional parameters specific to step types (`url`, `cssSelector`, `value`, `output`, etc.).

## 6. Conclusion

This system provides a powerful and flexible platform for automating browser-based tasks by seamlessly blending user-recorded actions with sophisticated AI-driven augmentation. The synergy between direct event capture and subsequent AI processing allows for the creation of robust, adaptable, and intelligent workflows that can handle dynamic web environments more effectively than traditional record-playback systems.

The core lifecycle of a workflow within this system can be summarized as follows:

*   **Capture**: The browser extension meticulously records user interactions (clicks, inputs, navigation, scrolls), enriching this data with contextual DOM information (selectors, element details) and visual evidence (screenshots). This forms the raw material for workflow creation.
*   **Build**: In the backend, the `BuilderService` employs Large Language Models, guided by carefully engineered prompts (`WORKFLOW_BUILDER_PROMPT_TEMPLATE`), to transform these raw recordings. This AI-driven process analyzes the user's actions and goals to generate a structured `WorkflowDefinitionSchema`. Key AI contributions here include the automatic identification of dynamic input parameters (defining the `input_schema`) and the strategic decision-making between simple deterministic steps and more complex, AI-powered agentic steps for interacting with variable content.
*   **Execute**: The `Workflow` service in the backend is responsible for running these structured workflows. It supports precise execution of deterministic actions (e.g., "click button X", "navigate to URL Y") via the `WorkflowController`. For steps requiring adaptability or complex decision-making (agentic steps), it delegates control to an AI `Agent` that interacts with the browser in real-time. Furthermore, AI is utilized for error recovery (`WORKFLOW_FALLBACK_PROMPT_TEMPLATE`) if deterministic steps fail, and for structured data extraction from web pages (`STRUCTURED_OUTPUT_PROMPT`).
*   **Manage**: A comprehensive Web UI allows users to visualize workflows as interactive graphs, inspect and edit workflow metadata and individual step configurations, and initiate and monitor workflow executions through a user-friendly interface, complete with input parameter entry and log viewing.

By combining the strengths of direct user recording with the analytical and adaptive capabilities of AI, the system is designed to facilitate the creation and execution of sophisticated browser automation tasks. This approach aims for automations that are not only easier to create but also more resilient to changes in web page structure and content, paving the way for more effective and intelligent task automation.
