# Agent Templates: Golden Agent Patterns

> Standardized structures for consistent, production-ready agents

## Overview

Agent Templates (Golden Agents) provide standardized patterns for building production-quality agents. By following these templates, teams achieve consistency, reduce errors, and accelerate development.

## Single File Agent (SFA) Pattern

The most common pattern for self-contained agents with isolated dependencies.

### Structure
```
agent_name/
├── sfa_agent_name.py      # Main agent file
├── pyproject.toml         # Dependencies (UV-managed)
├── README.md              # Usage documentation
└── tests/
    └── test_agent.py      # Unit tests
```

### Template
```python
#!/usr/bin/env python3
"""
Single File Agent: [Agent Name]

Purpose: [Brief description]
BLP Alignment: [Relevant BLP properties]
Requirements: [REQ-XXX-NNN references]

Usage:
    uv run sfa_agent_name.py [arguments]
"""

import argparse
import logging
from dataclasses import dataclass
from typing import Optional

# ============================================
# CONFIGURATION
# ============================================

@dataclass
class AgentConfig:
    """Agent configuration with sensible defaults."""
    model: str = "claude-sonnet-4-20250514"
    timeout_seconds: int = 30
    max_retries: int = 3
    log_level: str = "INFO"

# ============================================
# CORE AGENT LOGIC
# ============================================

class AgentName:
    """
    [Agent description]

    Implements:
        - REQ-FUN-XXX: [Requirement description]
        - BLP-XXX: [Property alignment]
    """

    def __init__(self, config: AgentConfig):
        self.config = config
        self.logger = logging.getLogger(__name__)
        self._setup_logging()

    def _setup_logging(self):
        logging.basicConfig(
            level=self.config.log_level,
            format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
        )

    def process(self, input_data: str) -> dict:
        """
        Main processing method.

        Args:
            input_data: Input to process

        Returns:
            Processing result dictionary
        """
        self.logger.info(f"Processing input: {input_data[:50]}...")

        try:
            result = self._execute_task(input_data)
            return {"status": "success", "result": result}
        except Exception as e:
            self.logger.error(f"Processing failed: {e}")
            return {"status": "error", "error": str(e)}

    def _execute_task(self, input_data: str) -> str:
        """Core task execution logic."""
        # Implementation here
        raise NotImplementedError

# ============================================
# CLI INTERFACE
# ============================================

def parse_args():
    parser = argparse.ArgumentParser(description="[Agent description]")
    parser.add_argument("input", help="Input to process")
    parser.add_argument("--model", default="claude-sonnet-4-20250514")
    parser.add_argument("--timeout", type=int, default=30)
    parser.add_argument("--verbose", action="store_true")
    return parser.parse_args()

def main():
    args = parse_args()

    config = AgentConfig(
        model=args.model,
        timeout_seconds=args.timeout,
        log_level="DEBUG" if args.verbose else "INFO"
    )

    agent = AgentName(config)
    result = agent.process(args.input)

    print(result)

if __name__ == "__main__":
    main()
```

## Orchestrator Agent Pattern

For agents that coordinate multiple sub-agents.

### Structure
```
orchestrator/
├── orchestrator.py        # Main orchestrator
├── sub_agents/
│   ├── agent_a.py
│   ├── agent_b.py
│   └── agent_c.py
├── workflows/
│   └── workflow_config.yaml
└── tests/
```

### Template
```python
"""
Orchestrator Agent Pattern

Coordinates multiple sub-agents through defined workflows.
"""

from dataclasses import dataclass
from enum import Enum
from typing import Dict, List, Optional
import asyncio

class WorkflowPhase(Enum):
    INIT = "init"
    PROCESSING = "processing"
    VALIDATION = "validation"
    COMPLETE = "complete"
    ERROR = "error"

@dataclass
class WorkflowState:
    phase: WorkflowPhase
    context: Dict
    results: List[Dict]
    errors: List[str]

class Orchestrator:
    """
    Coordinates sub-agent execution.

    BLP Alignment:
        - BLP-051: Workflow Analysis
        - BLP-058: Coordination
    """

    def __init__(self, agents: List, workflow_config: Dict):
        self.agents = {a.name: a for a in agents}
        self.workflow = workflow_config
        self.state = WorkflowState(
            phase=WorkflowPhase.INIT,
            context={},
            results=[],
            errors=[]
        )

    async def execute(self, input_data: Dict) -> Dict:
        """Execute the full workflow."""
        self.state.context = input_data
        self.state.phase = WorkflowPhase.PROCESSING

        for step in self.workflow["steps"]:
            try:
                result = await self._execute_step(step)
                self.state.results.append(result)
            except Exception as e:
                self.state.errors.append(str(e))
                if step.get("critical", False):
                    self.state.phase = WorkflowPhase.ERROR
                    return self._build_error_response()

        self.state.phase = WorkflowPhase.VALIDATION
        if await self._validate_results():
            self.state.phase = WorkflowPhase.COMPLETE
            return self._build_success_response()
        else:
            return self._build_validation_failure_response()

    async def _execute_step(self, step: Dict) -> Dict:
        """Execute a single workflow step."""
        agent_name = step["agent"]
        agent = self.agents[agent_name]

        # Prepare input from context
        step_input = self._resolve_input(step.get("input", {}))

        # Execute with timeout
        result = await asyncio.wait_for(
            agent.process(step_input),
            timeout=step.get("timeout", 60)
        )

        # Store output in context
        if "output_key" in step:
            self.state.context[step["output_key"]] = result

        return result

    async def _validate_results(self) -> bool:
        """Validate workflow results."""
        # Custom validation logic
        return len(self.state.errors) == 0
```

## MCP Tool Agent Pattern

For agents that wrap MCP (Model Context Protocol) tools.

### Template
```python
"""
MCP Tool Agent Pattern

Wraps MCP tools with cost tracking and error handling.
"""

from dataclasses import dataclass
from typing import Any, Dict, Optional
import time

@dataclass
class MCPToolCall:
    tool_name: str
    parameters: Dict
    timestamp: float
    duration_ms: Optional[float] = None
    cost: Optional[float] = None
    success: bool = True
    error: Optional[str] = None

class MCPToolAgent:
    """
    Wrapper for MCP tool access.

    Features:
        - Cost tracking per call
        - Rate limiting
        - Error handling with retry
        - Usage metrics
    """

    def __init__(self, tool_name: str, rate_limit_per_min: int = 60):
        self.tool_name = tool_name
        self.rate_limit = rate_limit_per_min
        self.call_history: List[MCPToolCall] = []
        self._last_call_time = 0

    async def call(self, parameters: Dict) -> Any:
        """Execute MCP tool with tracking."""
        # Rate limiting
        await self._enforce_rate_limit()

        call_record = MCPToolCall(
            tool_name=self.tool_name,
            parameters=parameters,
            timestamp=time.time()
        )

        start_time = time.time()
        try:
            result = await self._execute_tool(parameters)
            call_record.duration_ms = (time.time() - start_time) * 1000
            call_record.cost = self._calculate_cost(call_record)
            call_record.success = True
        except Exception as e:
            call_record.success = False
            call_record.error = str(e)
            raise

        finally:
            self.call_history.append(call_record)

        return result

    async def _execute_tool(self, parameters: Dict) -> Any:
        """Execute the actual MCP tool call."""
        # MCP tool execution logic
        raise NotImplementedError

    async def _enforce_rate_limit(self):
        """Enforce rate limiting between calls."""
        min_interval = 60.0 / self.rate_limit
        elapsed = time.time() - self._last_call_time
        if elapsed < min_interval:
            await asyncio.sleep(min_interval - elapsed)
        self._last_call_time = time.time()

    def _calculate_cost(self, call: MCPToolCall) -> float:
        """Calculate cost for the tool call."""
        # Cost calculation logic based on tool type
        return 0.0

    def get_usage_stats(self) -> Dict:
        """Get usage statistics."""
        total_calls = len(self.call_history)
        successful_calls = sum(1 for c in self.call_history if c.success)
        total_cost = sum(c.cost or 0 for c in self.call_history)

        return {
            "total_calls": total_calls,
            "success_rate": successful_calls / total_calls if total_calls > 0 else 0,
            "total_cost": total_cost,
            "avg_duration_ms": sum(c.duration_ms or 0 for c in self.call_history) / total_calls if total_calls > 0 else 0
        }
```

## DITD Sub-Agent Pattern

For agents aligned to specific DITD phases.

### Design Agent Template
```python
"""Design Agent - DITD Phase 1"""

class DesignAgent:
    """
    BLP Alignment: BLP-001 to BLP-010 (Alignment Properties)
    """

    def analyze_requirements(self, spec: str) -> Dict:
        """Decompose specification into requirements."""
        pass

    def generate_architecture(self, requirements: List) -> Dict:
        """Generate architecture from requirements."""
        pass

    def validate_alignment(self, design: Dict) -> Dict:
        """Validate BLP alignment of design."""
        pass
```

### Implementation Agent Template
```python
"""Implementation Agent - DITD Phase 2"""

class ImplementationAgent:
    """
    BLP Alignment: BLP-011 to BLP-020 (Autonomy Properties)
    """

    def generate_code(self, design: Dict) -> str:
        """Generate code from design."""
        pass

    def add_monitoring(self, code: str) -> str:
        """Add self-monitoring hooks."""
        pass

    def self_test(self, code: str) -> Dict:
        """Self-test generated code."""
        pass
```

## Best Practices

### Do's
- Include clear BLP and requirement traceability
- Implement comprehensive error handling
- Add logging at appropriate levels
- Include usage examples in docstrings
- Write tests alongside agent code

### Don'ts
- Don't hardcode configuration values
- Don't ignore rate limits
- Don't skip validation
- Don't log sensitive data
- Don't create circular dependencies

## Related Documentation

- [DITD Framework](../architectures/ditd-framework.md)
- [BLP Framework](../frameworks/blp-framework.md)
- [MCP Orchestration](mcp-orchestration.md)

---

*Templates transform agent development from creativity to consistency.*
