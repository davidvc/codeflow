# Workflow Template System - High-Level Design

## Executive Summary

This document describes the architecture of a workflow template system designed to solve three critical problems with AI coding agents: unreliable multi-step execution, inability to restart from failure points, and context window degradation. The system enables defining reusable workflow templates that can be instantiated as persistent, stateful task DAGs, with each task executed in a fresh agent context.

## Problem Statement

AI coding agents face three interconnected challenges:

1. **Unreliable Multi-Step Execution**: Agents don't consistently follow complex multi-step instructions defined in markdown documents
2. **No Restart Capability**: When workflows fail partway through, agents cannot easily resume from the failure point
3. **Context Window Degradation**: Large context windows cause hallucinations and poor decision-making

## Solution Overview

A multi-layer system that separates concerns:

- **Template Layer**: Define reusable workflow patterns
- **Data Plane**: Persistent, stateful task storage with dependency graphs
- **Execution Plane**: Fresh-context agent execution per task
- **Orchestration Layer**: DAG-aware workflow engine

All layers use pluggable interfaces to support multiple implementations.

## System Architecture

### Hexagonal Architecture

The system follows hexagonal (ports and adapters) architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                         Domain Core                          │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  Workflow   │  │   Template   │  │   Execution      │   │
│  │  Templates  │  │ Instantiation│  │   Session        │   │
│  └─────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   ┌────▼────┐           ┌────▼────┐           ┌────▼────┐
   │  Port:  │           │  Port:  │           │  Port:  │
   │  Task   │           │  Agent  │           │  Event  │
   │ Backend │           │Executor │           │ Stream  │
   └────┬────┘           └────┬────┘           └────┬────┘
        │                     │                     │
   ┌────┴────────┐       ┌────┴─────────┐     ┌────┴─────┐
   │   Adapters  │       │   Adapters   │     │ Adapters │
   │             │       │              │     │          │
   │ • Beads     │       │ • Claude     │     │ • Stdout │
   │ • Linear    │       │   Code       │     │ • File   │
   │ • Jira      │       │ • Aider      │     │ • HTTP   │
   │ • Custom    │       │ • Cursor     │     │ • Custom │
   └─────────────┘       │ • Custom     │     └──────────┘
                         └──────────────┘
```

### Architectural Layers

**Domain Core**
- Business logic independent of external systems
- Workflow template definitions and validation
- Task instantiation and DAG construction
- Execution orchestration logic

**Ports (Interfaces)**
- Abstract contracts for external dependencies
- No implementation details
- Dependency inversion principle

**Adapters (Implementations)**
- Concrete implementations of ports
- Integration with external systems
- Technology-specific details

## Core Components

### 1. Workflow Templates

**Purpose**: Define reusable, parameterized workflow patterns

**Responsibilities**:
- Define task sequence and dependencies
- Specify per-task instructions and context
- Support parameterization for customization
- Validation of template structure

**Key Concepts**:
- **Template Definition**: YAML/JSON structure defining tasks, dependencies, parameters
- **Parameters**: Variables that can be bound during instantiation
- **Task Specification**: Instructions, acceptance criteria, context for each task
- **Dependency Declaration**: DAG structure using task IDs

**Example Template Structure**:
```
Template:
  - name
  - description
  - parameters
  - tasks:
      - id
      - title
      - instructions
      - acceptance_criteria
      - depends_on: [task_ids]
      - context_variables
```

### 2. Template Instantiation

**Purpose**: Convert templates into concrete task instances

**Responsibilities**:
- Bind template parameters to values
- Create concrete tasks in backend
- Establish dependency relationships
- Generate unique instance identifiers

**Process**:
1. Validate template structure
2. Bind parameters with user-provided values
3. Generate task instances with interpolated content
4. Create tasks in backend via Task Backend Port
5. Establish dependencies to form DAG
6. Return workflow instance identifier

### 3. Task Backend Port

**Purpose**: Abstract interface for task persistence and management

**Interface Operations**:
- `createTask(spec)`: Create new task
- `updateTask(id, updates)`: Update task state
- `getTask(id)`: Retrieve task details
- `findReadyTasks(workflowId)`: Find tasks with no open blockers
- `addDependency(childId, parentId)`: Create dependency
- `claimTask(id, agentId)`: Atomically claim task for execution
- `completeTask(id, result)`: Mark task complete with results
- `failTask(id, error)`: Mark task failed with error details

**Key Properties**:
- Persistent storage
- Dependency graph support
- Status tracking (pending, claimed, in_progress, completed, failed)
- Query capabilities for DAG traversal
- Atomic operations for concurrency safety

### 4. Backend Adapters

**Beads Adapter** (Primary Implementation):
- Maps port operations to beads CLI commands
- Uses `bd create`, `bd dep add`, `bd ready`, `bd update --claim`
- Leverages git-backed storage and Dolt's merge capabilities
- Supports multi-agent workflows via hash-based IDs

**Other Potential Adapters**:
- Linear/GitHub Issues adapter
- Jira adapter
- Custom database adapter
- In-memory adapter (for testing)

**Adapter Responsibilities**:
- Translate abstract operations to backend-specific commands
- Handle backend-specific error cases
- Map domain concepts to backend data model

### 5. Coding Agent Port

**Purpose**: Abstract interface for executing tasks with AI agents

**Interface Operations**:
- `createSession(taskContext)`: Initialize fresh agent session
- `executeTask(sessionId, task)`: Execute task in session
- `getSessionOutput(sessionId)`: Retrieve execution results
- `terminateSession(sessionId)`: Clean up session resources

**Key Properties**:
- Fresh context per task (no context pollution)
- Task context injection
- Result capture
- Error handling
- Session isolation

### 6. Agent Adapters

**Claude Code Adapter** (Primary Implementation):
- Spawn fresh claude CLI session per task
- Inject task instructions via prompt
- Capture output and artifacts
- Parse completion status

**Other Potential Adapters**:
- Aider adapter
- Cursor adapter
- Custom LLM integration
- Human-in-the-loop adapter (for testing)

**Adapter Responsibilities**:
- Session lifecycle management
- Context injection strategies
- Output parsing and result extraction
- Error detection and reporting

### 7. Execution Engine

**Purpose**: Orchestrate DAG-based workflow execution

**Responsibilities**:
- DAG traversal and scheduling
- Task execution coordination
- Error handling and retry logic
- Progress tracking and reporting
- Concurrency management

**Execution Algorithm**:
```
while workflow not complete:
  1. Query backend for ready tasks (no open blockers)
  2. Select next task (priority-based or user-defined)
  3. Claim task atomically
  4. Create fresh agent session
  5. Execute task via Agent Port
  6. Capture results
  7. Update task status in backend
  8. If task failed: retry or mark failed
  9. If task succeeded: mark complete
  10. Repeat
```

**Error Handling**:
- Configurable retry policies
- Failed task isolation (doesn't block unrelated tasks)
- Error reporting and logging
- Manual intervention hooks

**Concurrency**:
- Parallel execution of independent tasks
- Atomic task claiming prevents duplicate work
- Backend handles concurrent updates

## Workflow Execution Model

### Phase 1: Template Definition

**Actor**: Workflow Author

**Activities**:
- Define workflow template structure
- Specify tasks with instructions and acceptance criteria
- Declare dependencies to form DAG
- Define parameters for customization
- Validate template syntax

**Artifacts**:
- Template file (YAML/JSON)
- Template validation results

### Phase 2: Template Instantiation

**Actor**: User/System

**Activities**:
- Select workflow template
- Provide parameter values
- Trigger instantiation
- System creates tasks in backend
- System establishes dependencies

**Artifacts**:
- Workflow instance ID
- Concrete tasks in backend
- Dependency graph

### Phase 3: Workflow Execution

**Actor**: Execution Engine

**Activities**:
- Loop: find ready tasks
- Claim task
- Spawn fresh agent session
- Inject task context and instructions
- Execute task
- Capture results
- Update task status
- Handle errors
- Repeat until complete

**Artifacts**:
- Execution logs
- Task completion results
- Artifacts produced by agents (code, docs, etc.)
- Error reports

### Phase 4: Monitoring & Intervention

**Actor**: User/System

**Activities**:
- Query workflow status
- View task progress
- Manually intervene on failures
- Restart from failure point
- Cancel workflow if needed

**Artifacts**:
- Status reports
- Progress metrics
- Intervention history

## Design Principles

### 1. Hexagonal Architecture
- Core domain isolated from infrastructure concerns
- Dependencies point inward
- External systems accessed only through ports

### 2. Dependency Inversion
- High-level modules don't depend on low-level modules
- Both depend on abstractions (ports)
- Enables pluggability and testing

### 3. Single Responsibility
- Each component has one reason to change
- Clear separation of concerns
- Template definition ≠ instantiation ≠ execution

### 4. Open/Closed Principle
- Open for extension via new adapters
- Closed for modification of core domain
- Add new backends/agents without changing core

### 5. Interface Segregation
- Focused, minimal port interfaces
- Clients don't depend on unused operations
- Task Backend Port vs Agent Port vs Event Port

### 6. Explicit over Implicit
- Clear data flow
- Explicit dependency declarations
- No hidden coupling

## Key Design Decisions

### Decision 1: Pluggable Backends via Ports

**Rationale**: Different teams use different task management systems. Ports enable swapping backends without changing core logic.

**Implications**:
- Must define common abstraction for task operations
- Adapters handle backend-specific details
- Some backend capabilities may not be portable

**Trade-offs**:
- Pro: Flexibility, future-proofing
- Con: Lowest common denominator in port interface

### Decision 2: Fresh Context Per Task

**Rationale**: Prevents context pollution and hallucinations. Each task starts with clean slate.

**Implications**:
- Tasks must be self-contained
- Context must be explicitly passed
- Startup overhead per task

**Trade-offs**:
- Pro: Reliability, predictability
- Con: Cannot leverage cross-task context (but this is a feature)

### Decision 3: DAG-Based Dependencies

**Rationale**: DAGs allow parallel execution while preserving ordering constraints.

**Implications**:
- Templates must specify dependencies
- Engine can parallelize independent tasks
- No circular dependencies allowed

**Trade-offs**:
- Pro: Explicit dependencies, parallelism
- Con: Requires careful template design

### Decision 4: Beads as Primary Backend

**Rationale**: Beads designed specifically for AI agent workflows, git-backed, supports dependencies.

**Implications**:
- Leverage beads' strengths (git backing, Dolt, compaction)
- Port interface influenced by beads capabilities
- Other adapters must match these capabilities

**Trade-offs**:
- Pro: Purpose-built for AI agents
- Con: Requires beads installation

### Decision 5: Template-First Design

**Rationale**: Reusability and standardization. Templates encode best practices.

**Implications**:
- Templates are first-class artifacts
- One-off workflows less convenient
- Template authoring requires care

**Trade-offs**:
- Pro: Reusability, consistency
- Con: Upfront template authoring cost

## Extension Points

### Custom Backends
Implement Task Backend Port for:
- Enterprise issue trackers (Jira, ServiceNow)
- Simple file-based storage
- Cloud-hosted task services

### Custom Agents
Implement Coding Agent Port for:
- Different AI coding tools
- Human-in-the-loop execution
- Hybrid AI+human workflows

### Custom Execution Strategies
Extend Execution Engine for:
- Different scheduling algorithms
- Custom retry policies
- Approval gates between tasks

### Event Streaming
Add Event Port for:
- Progress notifications
- Webhook integrations
- Monitoring dashboards

## Security Considerations

### Task Isolation
- Each agent session runs in isolation
- Limit agent permissions
- Sandbox execution environments

### Credential Management
- Secure storage for API keys
- Per-task credential injection
- Avoid credential leakage in logs

### Template Validation
- Schema validation for templates
- Prevent injection attacks in parameters
- Sanitize user inputs

### Access Control
- Authentication for workflow triggers
- Authorization for sensitive operations
- Audit logging

## Scalability Considerations

### Horizontal Scaling
- Multiple execution engines can run concurrently
- Atomic task claiming prevents conflicts
- Backend must support concurrent access

### Long-Running Workflows
- Checkpoint progress in backend
- Engine can restart from checkpoint
- Handle network failures gracefully

### Large DAGs
- Efficient DAG traversal algorithms
- Lazy loading of task details
- Pagination for task queries

## Success Criteria

The design successfully addresses the original problems if:

1. **Reliability**: Workflows execute all steps correctly without hallucinations
2. **Restartability**: Failed workflows can resume from failure point without rework
3. **Context Management**: Fresh context per task prevents context window issues
4. **Pluggability**: New backends and agents can be added without core changes
5. **Reusability**: Templates can be instantiated multiple times for different features
6. **Parallelism**: Independent tasks execute concurrently

## Future Enhancements

### Template Library
- Repository of reusable templates
- Template versioning
- Template composition

### Interactive Mode
- User approval gates between tasks
- Dynamic parameter binding during execution
- Human-AI collaboration

### Advanced Scheduling
- Priority-based task selection
- Resource-aware scheduling
- Cost optimization

### Analytics
- Workflow success rates
- Task duration metrics
- Agent performance analysis

### Template DSL
- Domain-specific language for templates
- Better IDE support
- Validation and testing tools

## Conclusion

This architecture provides a solid foundation for reliable, restartable, multi-step AI agent workflows. By separating concerns through hexagonal architecture and using pluggable interfaces, the system can evolve to support different backends and agents while maintaining a consistent execution model. The DAG-based approach enables parallel execution and explicit dependency management, solving the core problems of reliability and restartability.
