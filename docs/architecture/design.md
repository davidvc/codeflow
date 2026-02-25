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
┌──────────────────────────────────────────────────────────────────┐
│                          Domain Core                              │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐        │
│  │  Workflow   │  │   Template   │  │   Execution      │        │
│  │  Templates  │  │ Instantiation│  │   Session        │        │
│  └─────────────┘  └──────────────┘  └──────────────────┘        │
└──────────────────────────────────────────────────────────────────┘
                                │
        ┌───────┬───────┬───────┼────────┬────────┬────────┬───────┐
        │       │       │       │        │        │        │       │
   ┌────▼───┐┌─▼──┐┌───▼──┐┌───▼───┐┌───▼──┐┌────▼───┐┌───▼──┐┌──▼───┐
   │ Port:  ││Port││Port: ││ Port: ││Port: ││ Port:  ││Port: ││Port: │
   │  Task  ││Agent││Work- ││  VCS  ││ Step ││Workflow││Event ││ ...  │
   │Backend ││Exec ││space ││  Ops  ││Merge ││ Merge  ││Stream││      │
   └────┬───┘└─┬──┘└───┬──┘└───┬───┘└───┬──┘└────┬───┘└───┬──┘└──────┘
        │      │       │       │     ▲  │     ▲  │        │
   ┌────┴──┐┌──┴──┐┌──┴───┐┌──┴───┐ │  │     │  │   ┌────┴───┐
   │Adapter││Adapt││Adapter││Adapter│ │  │     │  │   │Adapter │
   │       ││     ││       ││       │ │  │     │  │   │        │
   │•Beads ││•Clde││• Git  ││• Git  │ └──┼─────┘  │   │•Stdout │
   │•Linear││ Code││Worktee││       │    │        │   │• File  │
   │•Jira  ││•Aider│• Docker│• Mock │    │        │   │• HTTP  │
   │•Custom││•Cursor│Custom ││•GitLab│    │        │   │•Custom │
   │       ││•Custom│       ││•Bitbkt│    │        │   │        │
   └───────┘└─────┘└───────┘└───────┘    │        │   └────────┘
                                          │        │
                                    ┌─────┴────┐┌──┴──────┐
                                    │ Adapters ││Adapters │
                                    │          ││         │
                                    │• Auto    ││• Auto   │
                                    │• PR+Merge││• PR+Mrg │
                                    │• PR+     ││• PR+    │
                                    │  Notify  ││  Notify │
                                    └──────────┘└─────────┘

                 Step & Workflow Merge Policies
                 depend on VCS Operations Port
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

### 7. Workspace Isolation Port

**Purpose**: Abstract interface for providing isolated workspaces for task execution

**Interface Operations**:
- `createWorkspace(taskId, baseRef)`: Create isolated workspace for task
- `getWorkspacePath(workspaceId)`: Get filesystem path to workspace
- `commitChanges(workspaceId, message)`: Commit changes made in workspace
- `mergeWorkspace(workspaceId, targetBranch)`: Merge workspace changes to target
- `destroyWorkspace(workspaceId)`: Clean up workspace resources
- `getWorkspaceStatus(workspaceId)`: Get workspace state and changes

**Key Properties**:
- Filesystem isolation per task
- Independent working directories
- Conflict-free parallel execution
- Change tracking and versioning
- Clean separation of task artifacts

**Why Workspace Isolation?**
- **Parallel Execution**: Multiple tasks can run simultaneously without file conflicts
- **Clean State**: Each task starts with a known, clean codebase state
- **Rollback Safety**: Failed tasks don't pollute the main workspace
- **Merge Control**: Explicit control over when and how changes integrate
- **Reproducibility**: Each task operates on a specific code snapshot

### 8. Workspace Adapters

**Git Worktree Adapter** (Primary Implementation):
- Uses `git worktree add` to create isolated working directories
- Each worktree is a full checkout on a separate branch
- Changes can be committed and merged independently
- Automatic cleanup of worktrees after task completion
- Leverages git's native isolation mechanisms

**Implementation Details**:
- Creates worktree in `.codeflow/worktrees/<task-id>/`
- Checks out new branch `task/<task-id>` from base reference
- Agent executes within worktree directory
- On success: merge to main branch, remove worktree
- On failure: preserve worktree for debugging, allow manual cleanup

**Other Potential Adapters**:
- Docker container adapter (full OS-level isolation)
- Directory copy adapter (simple filesystem copy)
- Virtual environment adapter (Python/Node-specific)
- Remote workspace adapter (cloud-based execution)

**Adapter Responsibilities**:
- Create isolated working environment
- Manage workspace lifecycle
- Handle merge/integration operations
- Clean up resources
- Track workspace state

### 9. VCS Operations Port

**Purpose**: Abstract interface for low-level version control operations

**Interface Operations**:
- `createBranch(name, baseBranch)`: Create new branch
- `mergeBranch(sourceBranch, targetBranch)`: Perform merge operation
- `deleteBranch(name)`: Delete branch
- `getBranchStatus(name)`: Get branch state
- `hasConflicts(sourceBranch, targetBranch)`: Check for merge conflicts
- `getConflicts(sourceBranch, targetBranch)`: Get conflict details
- `createPullRequest(source, target, title, body)`: Create PR/MR
- `getPullRequestStatus(prId)`: Get PR state
- `getPullRequestChecks(prId)`: Get CI/CD check results
- `mergePullRequest(prId)`: Merge PR
- `closePullRequest(prId)`: Close PR without merging

**Key Properties**:
- VCS-agnostic operations
- Branch management
- Pull request/merge request abstraction
- Conflict detection
- Check status monitoring

**Why Separate VCS Port?**
- Merge policies focus on *when* and *how* to merge (business logic)
- VCS port handles *actual* merge operations (infrastructure)
- Enables testing with mock VCS
- Future-proofs against VCS changes (though git is dominant)

### 10. VCS Adapters

**Git Adapter** (Primary Implementation):
- Uses git CLI commands (`git branch`, `git merge`, `git push`)
- Integrates with GitHub via `gh` CLI for PR operations
- Detects conflicts via merge output
- Maps git concepts to port interface

**Implementation Details**:
- `createBranch()` → `git branch <name> <base>`
- `mergeBranch()` → `git merge <source>`
- `createPullRequest()` → `gh pr create --base <target> --head <source>`
- `getPullRequestChecks()` → `gh pr checks <pr-id>`
- `mergePullRequest()` → `gh pr merge <pr-id>`

**Other Potential Adapters**:
- GitLab adapter (uses `glab` CLI)
- Bitbucket adapter (uses `bb` CLI or API)
- Mock adapter (for testing)

**Adapter Responsibilities**:
- Execute VCS commands
- Parse VCS output
- Handle VCS-specific errors
- Map VCS concepts to port abstractions

### 11. Step Merge Policy Port

**Purpose**: Abstract interface for *policy* around merging completed task branches into workflow branch

**Interface Operations**:
- `mergeStep(stepBranch, workflowBranch, taskContext)`: Execute merge policy for step
- `getMergeStatus(mergeId)`: Check status of merge operation
- `handleMergeFailure(mergeId, error)`: Handle merge conflicts or failures
- `cancelMerge(mergeId)`: Cancel pending merge

**Key Properties**:
- Policy-driven merge strategies (when/how to merge)
- Depends on VCS Operations Port for actual merging
- CI/CD integration support
- Automated quality gates
- Conflict detection and resolution
- Separate agent session for merge operations

**Merge Lifecycle**:
1. Task completes successfully in its workspace
2. Merge policy invoked with step branch and workflow branch
3. Policy executes strategy using VCS Port:
   - Check for conflicts via VCS Port
   - Create PR via VCS Port (or direct merge)
   - Monitor checks via VCS Port
   - Merge via VCS Port
4. If checks fail: spawn fix agent to resolve issues
5. On success: step branch merged to workflow branch
6. Cleanup: remove step branch via VCS Port

### 12. Step Merge Policy Adapters

All policy adapters depend on the VCS Operations Port for actual VCS interactions.

**PR and Notify Policy** (Primary/Default Implementation):
- Creates pull request from step branch to workflow branch via VCS Port
- Waits for CI/CD checks to complete (polls via VCS Port)
- If checks pass: notifies user via Event Port, waits for approval or auto-merges
- If checks fail: spawns new agent session to fix failures
- Uses VCS Port exclusively for git/PR operations

**Implementation Flow**:
1. Call VCS Port: `createPullRequest(task/<task-id>, workflow/<workflow-id>)`
2. Poll VCS Port: `getPullRequestChecks(prId)` until complete
3. If checks fail:
   - Create new task via Task Backend Port: "Fix CI failures for task-X"
   - Return failure status
4. If checks pass:
   - Notify via Event Port
   - Wait for user approval (or auto-merge based on config)
   - Call VCS Port: `mergePullRequest(prId)`
5. Call VCS Port: `deleteBranch(task/<task-id>)`

**PR and Merge Policy**:
- Creates pull request with auto-merge enabled
- Waits for checks to pass via VCS Port
- Automatically merges when all checks green (no approval)
- If checks fail: spawn fix agent
- Uses VCS Port for all operations

**Auto Merge Policy**:
- Directly merges step branch to workflow branch via VCS Port
- No PR created
- No CI/CD gates
- Fast but risky (useful for development/testing)
- Call VCS Port: `mergeBranch(task/<task-id>, workflow/<workflow-id>)`

**Adapter Responsibilities**:
- Implement merge strategy (business logic)
- Orchestrate VCS operations via VCS Port (not direct VCS calls)
- Monitor CI/CD check status via VCS Port
- Spawn fix agents on failure via Task Backend Port
- Notify users via Event Port
- Clean up branches via VCS Port

### 13. Workflow Merge Policy Port

**Purpose**: Abstract interface for *policy* around merging completed workflow branch into main/target branch

**Interface Operations**:
- `mergeWorkflow(workflowBranch, targetBranch, workflowContext)`: Execute merge policy for workflow
- `getMergeStatus(mergeId)`: Check status of workflow merge
- `handleMergeFailure(mergeId, error)`: Handle workflow-level merge issues
- `cancelMerge(mergeId)`: Cancel pending workflow merge

**Key Properties**:
- Workflow-level quality gates
- Depends on VCS Operations Port for actual merging
- Final integration point
- Production readiness checks
- Separate agent for workflow-level fixes

**Workflow Merge Lifecycle**:
1. All workflow tasks complete
2. Workflow branch contains all merged steps
3. Workflow merge policy invoked
4. Policy executes strategy using VCS Port:
   - Check for conflicts via VCS Port
   - Create final PR via VCS Port
   - Monitor checks via VCS Port
   - Merge via VCS Port
5. If checks fail: spawn fix agent for workflow-level issues
6. On success: workflow merged to main/target
7. Workflow marked complete

### 14. Workflow Merge Policy Adapters

All policy adapters depend on the VCS Operations Port for actual VCS interactions.

**PR and Notify Policy** (Primary/Default Implementation):
- Creates final PR from workflow branch to main via VCS Port
- Runs full CI/CD suite (monitored via VCS Port)
- Waits for all checks
- Notifies user when checks pass via Event Port
- Requires approval before merge
- If checks fail: spawn workflow-level fix agent

**Implementation Flow**:
1. Call VCS Port: `createPullRequest(workflow/<workflow-id>, main)`
2. Poll VCS Port: `getPullRequestChecks(prId)` until complete
3. If checks fail:
   - Create workflow fix task via Task Backend Port
   - Return failure status
4. If checks pass:
   - Notify via Event Port
   - Wait for user approval
   - Call VCS Port: `mergePullRequest(prId)`
5. Call VCS Port: `deleteBranch(workflow/<workflow-id>)` (optional)

**PR and Merge Policy**:
- Creates PR with auto-merge via VCS Port
- Merges when checks pass (no approval)
- All operations via VCS Port

**Auto Merge Policy**:
- Direct merge to main via VCS Port
- No PR, no gates
- Call VCS Port: `mergeBranch(workflow/<workflow-id>, main)`

### 15. Branching Strategy

**Branch Hierarchy**:
```
main (production)
  └─ workflow/<workflow-id> (workflow branch)
       ├─ task/<task-1-id> (step branch in worktree)
       ├─ task/<task-2-id> (step branch in worktree)
       └─ task/<task-n-id> (step branch in worktree)
```

**Workflow Lifecycle**:
1. **Workflow Start**: Create `workflow/<workflow-id>` branch from main
2. **Task Execution**: Each task works in `task/<task-id>` branch (in worktree)
3. **Step Merge**: Task branch merged to workflow branch via Step Merge Policy
4. **Workflow Completion**: All tasks complete, workflow branch merged to main via Workflow Merge Policy

**Merge Flow**:
- **Task → Workflow**: Step Merge Policy controls integration
- **Workflow → Main**: Workflow Merge Policy controls final integration

**Benefits**:
- Isolated feature development
- Quality gates at each level
- Easy rollback of entire workflow
- Clear history of workflow progress
- Parallel task execution safety

### 16. Merge Failure Handling

**Purpose**: Automated recovery from merge failures

**Step Merge Failures**:
- Detected by Step Merge Policy adapter
- Create new task: "Fix merge/CI failures for task-X"
- Add dependency: blocked by original task
- Spawn agent session with context:
  - Original task intent
  - CI/CD failure logs
  - Merge conflict details
- Agent fixes issues in workspace
- Re-run merge policy
- If still failing: escalate to user

**Workflow Merge Failures**:
- Detected by Workflow Merge Policy adapter
- Create new task: "Fix workflow-level failures"
- Spawn agent session with:
  - Full workflow context
  - Integration test failures
  - Merge conflicts
- Agent resolves workflow-level issues
- Re-run workflow merge policy

**Fix Agent Strategy**:
- Fresh agent session (not part of original workflow)
- Access to failure context
- Can modify code, fix tests, resolve conflicts
- Reports back via task completion
- Bounded retry attempts (configurable)

### 17. Execution Engine

**Purpose**: Orchestrate DAG-based workflow execution

**Responsibilities**:
- DAG traversal and scheduling
- Task execution coordination
- Error handling and retry logic
- Progress tracking and reporting
- Concurrency management

**Execution Algorithm**:
```
# Phase 1: Workflow Initialization
1. Create workflow branch from main: workflow/<workflow-id>

# Phase 2: Task Execution Loop
while workflow has incomplete tasks:
  2. Query backend for ready tasks (no open blockers)
  3. Select next task (priority-based or user-defined)
  4. Claim task atomically
  5. Create isolated workspace for task
     - Create worktree with task branch: task/<task-id>
     - Branch from current workflow branch
  6. Create fresh agent session with workspace path
  7. Execute task via Agent Port
  8. Capture results

  9. If task succeeded:
     a. Commit changes in task workspace
     b. Invoke Step Merge Policy:
        - Merge task/<task-id> → workflow/<workflow-id>
        - Wait for checks/approval based on policy
        - If merge fails: create fix task, goto 2
     c. Mark task complete in backend
     d. Destroy workspace and task branch

  10. If task failed:
      a. Mark task failed in backend
      b. Preserve workspace for debugging
      c. Retry or create fix task

  11. Repeat until all tasks complete

# Phase 3: Workflow Completion
12. Invoke Workflow Merge Policy:
    - Merge workflow/<workflow-id> → main
    - Wait for checks/approval based on policy
    - If merge fails: create workflow fix task, goto 2
13. Mark workflow complete in backend
14. Cleanup workflow branch (optional)
15. Send completion notification
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

**Actor**: Execution Engine + Merge Policies

**Activities**:
- Create workflow branch
- Loop: find ready tasks
- Claim task
- Create isolated workspace with task branch
- Spawn fresh agent session in workspace
- Inject task context and instructions
- Execute task
- Capture results
- Commit changes to task branch
- **Invoke Step Merge Policy**
  - Create PR (or auto-merge based on policy)
  - Monitor CI/CD checks
  - If checks fail: spawn fix agent
  - If checks pass: notify/merge based on policy
- Update task status
- Clean up workspace and task branch
- Handle errors
- Repeat until all tasks complete
- **Invoke Workflow Merge Policy**
  - Create final PR to main
  - Monitor CI/CD checks
  - If checks fail: spawn workflow fix agent
  - If checks pass: notify/merge based on policy

**Artifacts**:
- Workflow branch
- Task branches (one per task)
- Pull requests (step-level and workflow-level)
- CI/CD check results
- Execution logs
- Task completion results
- Artifacts produced by agents (code, docs, etc.)
- Git commits per task
- Merged branches
- Error reports
- Fix task records

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
- Separate ports: Task Backend, Agent, Workspace, VCS, Step Merge, Workflow Merge, Event

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

### Decision 6: Workspace Isolation via Git Worktrees

**Rationale**: Each task needs its own isolated filesystem to prevent conflicts during parallel execution and enable safe rollback.

**Implications**:
- Tasks execute in separate working directories
- Git worktrees provide native version control integration
- Requires git repository for default implementation
- Merge strategy needed for integrating changes

**Trade-offs**:
- Pro: True isolation, parallel execution, git integration
- Con: Disk space overhead, worktree management complexity

**Alternative Implementations**:
- Docker containers for full OS isolation
- Simple directory copies for non-git projects
- Cloud-based workspaces for distributed execution

### Decision 7: Hierarchical Branching Strategy

**Rationale**: Workflow branch acts as integration point for all task branches, enabling quality gates at step and workflow levels.

**Implications**:
- Each workflow gets dedicated branch
- Each task gets dedicated branch (in worktree)
- Two merge points: task→workflow, workflow→main
- Clear separation of concerns

**Trade-offs**:
- Pro: Multiple quality gates, clear history, easy rollback
- Con: More branches to manage, slower than direct-to-main

### Decision 8: Two-Layer Merge Architecture

**Rationale**: Separate policy (when/how to merge) from mechanism (actual VCS operations).

**Implications**:
- **Layer 1 (VCS Operations Port)**: Low-level VCS operations (git, PRs, branches)
- **Layer 2 (Merge Policy Ports)**: High-level policy (auto, PR+notify, PR+merge)
- Merge policies depend on VCS Port
- Can swap VCS implementation independently of policy
- Policies are VCS-agnostic (work with git, GitLab, Bitbucket)

**Trade-offs**:
- Pro: Clean separation, testability, flexibility
- Con: Additional abstraction layer

### Decision 9: Pluggable Merge Policies

**Rationale**: Different teams have different quality/velocity trade-offs. Policies allow customization without changing core logic.

**Implications**:
- Separate ports for step merge vs workflow merge
- Default: PR and notify (safest but slowest)
- Can swap to auto-merge for speed (development)
- Can require approval for production workflows
- All policies use VCS Operations Port

**Trade-offs**:
- Pro: Flexibility, CI/CD integration, quality gates
- Con: Complexity, slower execution with checks

### Decision 10: Separate Merge Agent Sessions

**Rationale**: Merge operations are distinct from task execution and may require fixing failures.

**Implications**:
- Merge not performed by task agent
- Enables automated fix agents for merge failures
- Clear separation of responsibilities
- Fix agents have full context of failure

**Trade-offs**:
- Pro: Automated recovery, clear accountability
- Con: Additional agent sessions, potential for fix loops

### Decision 11: Default to PR and Notify

**Rationale**: Safety first. Human oversight prevents bad merges reaching main.

**Implications**:
- All merges create PRs by default
- User notified when checks pass
- Requires GitHub/GitLab/Bitbucket integration
- Can override with faster policies when needed

**Trade-offs**:
- Pro: Safety, code review opportunity, audit trail
- Con: Slower, requires manual intervention

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

### Custom Workspaces
Implement Workspace Isolation Port for:
- Docker/Podman containers
- Cloud-based development environments
- Language-specific virtual environments
- Remote execution environments

### Custom VCS Operations
Implement VCS Operations Port for:
- GitLab (using glab CLI or API)
- Bitbucket (using bb CLI or API)
- Mock VCS (for testing)
- Custom VCS systems

### Custom Merge Policies
Implement Step/Workflow Merge Policy Ports for:
- Custom approval workflows
- Advanced CI/CD integrations
- Blue/green deployment strategies
- Canary deployment patterns
- Security scanning gates
- Performance benchmark gates

Note: All merge policies use VCS Operations Port, so they work with any VCS adapter.

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
- Isolated workspaces prevent cross-task contamination
- Limit agent permissions to workspace boundaries
- Sandbox execution environments
- Git worktrees provide filesystem-level isolation

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
7. **Quality Gates**: Step and workflow merges enforce CI/CD checks
8. **Automated Recovery**: Merge failures trigger fix agents automatically
9. **Isolation**: Each task executes in isolated workspace on isolated branch
10. **Traceability**: Clear PR and commit history for audit

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

This architecture provides a solid foundation for reliable, restartable, multi-step AI agent workflows with robust quality gates. By separating concerns through hexagonal architecture and using pluggable interfaces, the system can evolve to support different backends, agents, and merge policies while maintaining a consistent execution model.

Key architectural elements:
- **DAG-based task dependencies** enable parallel execution and explicit ordering
- **Fresh context per task** prevents hallucinations and context pollution
- **Isolated workspaces** provide filesystem-level task isolation
- **Hierarchical branching** (main → workflow → task) enables multi-level quality gates
- **Pluggable merge policies** allow teams to balance quality vs velocity
- **Automated fix agents** recover from merge and CI/CD failures
- **Template-driven workflows** encode best practices and enable reuse

This design solves the original three problems while adding enterprise-grade features like CI/CD integration, code review workflows, and automated failure recovery.
