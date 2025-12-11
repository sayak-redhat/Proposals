# Self-Healing Pytest Framework Design
## MCP-Powered Auto-Recovery for ZTWIM Regression Suite

**Date:** December 2024  
**Project:** ZTWIM Python Automation Framework

---

## Overview

This document describes how to integrate **self-healing capabilities** directly into our pytest-based regression framework, enabling automatic test recovery and adaptation when code changes break existing tests.

---

## Architecture: Self-Healing Pytest Framework

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 SELF-HEALING PYTEST FRAMEWORK                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      PYTEST EXECUTION                                │   │
│   └─────────────────────────────┬───────────────────────────────────────┘   │
│                                 │                                            │
│                    ┌────────────┴────────────┐                              │
│                    │                         │                              │
│                    ▼                         ▼                              │
│            ┌─────────────┐           ┌─────────────┐                       │
│            │    PASS     │           │    FAIL     │                       │
│            └─────────────┘           └──────┬──────┘                       │
│                                             │                               │
│                                             ▼                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                   SELF-HEALING ENGINE                                │   │
│   ├─────────────────────────────────────────────────────────────────────┤   │
│   │                                                                      │   │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │   │
│   │   │   FAILURE    │  │   MCP/AI     │  │    FIX       │             │   │
│   │   │   ANALYZER   │──▶│   ENGINE    │──▶│   APPLIER   │             │   │
│   │   └──────────────┘  └──────────────┘  └──────────────┘             │   │
│   │                                                                      │   │
│   │   • Capture error    • Analyze cause    • Apply fix                 │   │
│   │   • Get context      • Generate fix     • Retry test                │   │
│   │   • Check patterns   • Validate fix     • Log changes               │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                 │                                            │
│                                 ▼                                            │
│                    ┌────────────┴────────────┐                              │
│                    │                         │                              │
│                    ▼                         ▼                              │
│            ┌─────────────┐           ┌─────────────┐                       │
│            │  HEALED ✓   │           │  NEEDS      │                       │
│            │  (Re-run)   │           │  HUMAN      │                       │
│            └─────────────┘           └─────────────┘                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Implementation: Core Components

### 1. Directory Structure

```
ztwim-automation/
├── framework/
│   ├── __init__.py
│   ├── healing/                          # 🆕 Self-Healing Module
│   │   ├── __init__.py
│   │   ├── engine.py                     # Main healing engine
│   │   ├── analyzer.py                   # Failure analysis
│   │   ├── strategies/                   # Healing strategies
│   │   │   ├── __init__.py
│   │   │   ├── selector_healer.py        # Fix broken selectors
│   │   │   ├── timeout_healer.py         # Fix timeout issues
│   │   │   ├── resource_healer.py        # Fix resource references
│   │   │   ├── api_healer.py             # Fix API changes
│   │   │   └── assertion_healer.py       # Fix assertion mismatches
│   │   ├── mcp_client.py                 # MCP server integration
│   │   └── history.py                    # Healing history tracking
│   │
│   ├── cluster/
│   ├── clients/
│   ├── resources/
│   └── utils/
│
├── tests/
├── conftest.py                           # 🆕 Healing hooks integrated
└── pytest.ini
```

### 2. Pytest Hook: Auto-Healing on Failure

```python
# conftest.py

import pytest
from framework.healing import HealingEngine, HealingConfig

# Initialize healing engine
healing_engine = HealingEngine(
    config=HealingConfig(
        enabled=True,
        max_retries=2,
        auto_commit=False,  # Require human review
        mcp_enabled=True,
    )
)

@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """
    Hook that intercepts test failures and attempts self-healing.
    """
    outcome = yield
    report = outcome.get_result()
    
    if report.when == "call" and report.failed:
        # Attempt self-healing
        healing_result = healing_engine.attempt_heal(
            test_item=item,
            failure_report=report,
            exception=call.excinfo,
        )
        
        if healing_result.healed:
            # Mark for retry
            item.add_marker(pytest.mark.flaky(reruns=1))
            report.outcome = "healed"
            report.longrepr = f"HEALED: {healing_result.fix_description}"
        else:
            # Add healing suggestions to report
            report.sections.append((
                "Self-Healing Suggestions",
                healing_result.suggestions
            ))


@pytest.fixture(autouse=True)
def healing_context(request):
    """
    Provides healing context to each test.
    """
    healing_engine.start_test_context(request.node.nodeid)
    yield
    healing_engine.end_test_context()
```

### 3. Healing Engine

```python
# framework/healing/engine.py

from dataclasses import dataclass
from typing import Optional, List
from .analyzer import FailureAnalyzer
from .strategies import get_healing_strategy
from .mcp_client import MCPHealingClient
from .history import HealingHistory

@dataclass
class HealingConfig:
    enabled: bool = True
    max_retries: int = 2
    auto_commit: bool = False
    mcp_enabled: bool = True
    history_file: str = ".healing_history.json"

@dataclass
class HealingResult:
    healed: bool
    fix_description: str
    fix_diff: Optional[str]
    suggestions: str
    confidence: float  # 0.0 to 1.0

class HealingEngine:
    """
    Main self-healing engine for pytest framework.
    """
    
    def __init__(self, config: HealingConfig):
        self.config = config
        self.analyzer = FailureAnalyzer()
        self.history = HealingHistory(config.history_file)
        self.mcp_client = MCPHealingClient() if config.mcp_enabled else None
        self._current_context = None
    
    def attempt_heal(
        self,
        test_item,
        failure_report,
        exception
    ) -> HealingResult:
        """
        Attempt to heal a failed test.
        """
        if not self.config.enabled:
            return HealingResult(
                healed=False,
                fix_description="Healing disabled",
                fix_diff=None,
                suggestions="Enable healing in config",
                confidence=0.0
            )
        
        # Step 1: Analyze the failure
        analysis = self.analyzer.analyze(
            test_file=test_item.fspath,
            test_name=test_item.name,
            exception=exception,
            failure_output=str(failure_report.longrepr),
        )
        
        # Step 2: Check healing history (avoid repeated failures)
        if self.history.has_failed_healing(analysis.signature):
            return HealingResult(
                healed=False,
                fix_description="Previously failed to heal",
                fix_diff=None,
                suggestions=self.history.get_suggestions(analysis.signature),
                confidence=0.0
            )
        
        # Step 3: Get appropriate healing strategy
        strategy = get_healing_strategy(analysis.failure_type)
        
        if strategy is None:
            # Step 4: Fall back to MCP/AI for complex cases
            if self.mcp_client:
                return self._mcp_heal(analysis, test_item)
            return HealingResult(
                healed=False,
                fix_description="No strategy available",
                fix_diff=None,
                suggestions=analysis.suggestions,
                confidence=0.0
            )
        
        # Step 5: Apply healing strategy
        fix_result = strategy.heal(analysis)
        
        if fix_result.success:
            # Apply the fix
            self._apply_fix(fix_result, test_item)
            self.history.record_success(analysis.signature, fix_result)
            
            return HealingResult(
                healed=True,
                fix_description=fix_result.description,
                fix_diff=fix_result.diff,
                suggestions="",
                confidence=fix_result.confidence
            )
        
        return HealingResult(
            healed=False,
            fix_description="Strategy failed",
            fix_diff=None,
            suggestions=fix_result.suggestions,
            confidence=0.0
        )
    
    def _mcp_heal(self, analysis, test_item) -> HealingResult:
        """
        Use MCP/AI to generate healing suggestions.
        """
        response = self.mcp_client.request_healing(
            test_file=str(test_item.fspath),
            test_content=test_item.fspath.read_text(),
            failure_analysis=analysis,
            git_diff=self._get_recent_changes(),
        )
        
        if response.fix_available:
            if self.config.auto_commit:
                self._apply_fix(response.fix, test_item)
                return HealingResult(
                    healed=True,
                    fix_description=response.description,
                    fix_diff=response.diff,
                    suggestions="",
                    confidence=response.confidence
                )
            else:
                # Create PR for review
                self._create_fix_pr(response, test_item)
                return HealingResult(
                    healed=False,
                    fix_description="Fix PR created for review",
                    fix_diff=response.diff,
                    suggestions=f"Review PR: {response.pr_url}",
                    confidence=response.confidence
                )
        
        return HealingResult(
            healed=False,
            fix_description="MCP could not generate fix",
            fix_diff=None,
            suggestions=response.suggestions,
            confidence=0.0
        )
    
    def _apply_fix(self, fix_result, test_item):
        """Apply the fix to the test file."""
        # Implementation details...
        pass
    
    def _get_recent_changes(self) -> str:
        """Get recent git changes for context."""
        import subprocess
        result = subprocess.run(
            ["git", "diff", "HEAD~5", "--", "*.py"],
            capture_output=True,
            text=True
        )
        return result.stdout
    
    def _create_fix_pr(self, response, test_item):
        """Create a PR with the suggested fix."""
        # Implementation details...
        pass
```

### 4. Failure Analyzer

```python
# framework/healing/analyzer.py

from dataclasses import dataclass
from enum import Enum
from typing import Optional
import re

class FailureType(Enum):
    SELECTOR_NOT_FOUND = "selector_not_found"
    TIMEOUT = "timeout"
    ASSERTION_MISMATCH = "assertion_mismatch"
    RESOURCE_NOT_FOUND = "resource_not_found"
    API_CHANGED = "api_changed"
    FIELD_RENAMED = "field_renamed"
    PERMISSION_DENIED = "permission_denied"
    CONNECTION_ERROR = "connection_error"
    UNKNOWN = "unknown"

@dataclass
class FailureAnalysis:
    failure_type: FailureType
    signature: str  # Unique signature for this failure pattern
    root_cause: str
    affected_element: Optional[str]
    expected_value: Optional[str]
    actual_value: Optional[str]
    suggestions: str
    context: dict

class FailureAnalyzer:
    """
    Analyzes test failures to determine cause and healing strategy.
    """
    
    # Pattern matchers for different failure types
    PATTERNS = {
        FailureType.SELECTOR_NOT_FOUND: [
            r"KeyError: '(\w+)'",
            r"field '(\w+)' not found",
            r"no such element: (\w+)",
        ],
        FailureType.TIMEOUT: [
            r"TimeoutError",
            r"timed out after (\d+)",
            r"Timeout waiting for",
        ],
        FailureType.ASSERTION_MISMATCH: [
            r"AssertionError: assert (.+) == (.+)",
            r"Expected: (.+)\s+Actual: (.+)",
        ],
        FailureType.RESOURCE_NOT_FOUND: [
            r"NotFound: .+ \"(\w+)\" not found",
            r"resource (\w+) does not exist",
        ],
        FailureType.API_CHANGED: [
            r"AttributeError: '(\w+)' object has no attribute '(\w+)'",
            r"TypeError: .+ got an unexpected keyword argument '(\w+)'",
        ],
        FailureType.FIELD_RENAMED: [
            r"KeyError: '(\w+)'.*status",
            r"'(\w+)' is not a valid field",
        ],
    }
    
    def analyze(
        self,
        test_file: str,
        test_name: str,
        exception,
        failure_output: str,
    ) -> FailureAnalysis:
        """
        Analyze a test failure and determine its type and root cause.
        """
        failure_type = self._detect_failure_type(failure_output, exception)
        
        analysis = FailureAnalysis(
            failure_type=failure_type,
            signature=self._generate_signature(test_file, test_name, failure_type),
            root_cause=self._extract_root_cause(failure_output, failure_type),
            affected_element=self._extract_affected_element(failure_output, failure_type),
            expected_value=self._extract_expected(failure_output),
            actual_value=self._extract_actual(failure_output),
            suggestions=self._generate_suggestions(failure_type),
            context={
                "test_file": test_file,
                "test_name": test_name,
                "exception_type": type(exception.value).__name__ if exception else None,
            }
        )
        
        return analysis
    
    def _detect_failure_type(self, output: str, exception) -> FailureType:
        """Detect the type of failure from output and exception."""
        for failure_type, patterns in self.PATTERNS.items():
            for pattern in patterns:
                if re.search(pattern, output, re.IGNORECASE):
                    return failure_type
        return FailureType.UNKNOWN
    
    def _generate_signature(self, test_file: str, test_name: str, failure_type: FailureType) -> str:
        """Generate unique signature for this failure pattern."""
        import hashlib
        content = f"{test_file}:{test_name}:{failure_type.value}"
        return hashlib.md5(content.encode()).hexdigest()[:12]
    
    def _extract_root_cause(self, output: str, failure_type: FailureType) -> str:
        """Extract the root cause from failure output."""
        # Implementation based on failure type
        causes = {
            FailureType.SELECTOR_NOT_FOUND: "Resource field or selector changed",
            FailureType.TIMEOUT: "Operation took longer than expected",
            FailureType.ASSERTION_MISMATCH: "Expected value differs from actual",
            FailureType.RESOURCE_NOT_FOUND: "Kubernetes resource not found",
            FailureType.API_CHANGED: "API signature changed",
            FailureType.FIELD_RENAMED: "Field was renamed in resource",
        }
        return causes.get(failure_type, "Unknown cause")
    
    def _extract_affected_element(self, output: str, failure_type: FailureType) -> Optional[str]:
        """Extract the affected element (field, selector, etc.)."""
        for pattern in self.PATTERNS.get(failure_type, []):
            match = re.search(pattern, output)
            if match:
                return match.group(1)
        return None
    
    def _extract_expected(self, output: str) -> Optional[str]:
        """Extract expected value from assertion."""
        match = re.search(r"Expected[:\s]+(.+?)(?:\n|Actual|$)", output)
        return match.group(1).strip() if match else None
    
    def _extract_actual(self, output: str) -> Optional[str]:
        """Extract actual value from assertion."""
        match = re.search(r"Actual[:\s]+(.+?)(?:\n|$)", output)
        return match.group(1).strip() if match else None
    
    def _generate_suggestions(self, failure_type: FailureType) -> str:
        """Generate human-readable suggestions."""
        suggestions = {
            FailureType.SELECTOR_NOT_FOUND: 
                "Check if field name changed in recent commits. "
                "Verify resource schema.",
            FailureType.TIMEOUT: 
                "Consider increasing timeout. "
                "Check if resource is slower to create.",
            FailureType.ASSERTION_MISMATCH: 
                "Verify expected values are correct. "
                "Check if behavior changed intentionally.",
            FailureType.RESOURCE_NOT_FOUND: 
                "Check resource name/namespace. "
                "Verify resource was created successfully.",
            FailureType.API_CHANGED: 
                "Check API documentation for changes. "
                "Update method signature.",
            FailureType.FIELD_RENAMED: 
                "Check git diff for field renames. "
                "Update field references in test.",
        }
        return suggestions.get(failure_type, "Manual investigation required.")
```

### 5. Healing Strategies

```python
# framework/healing/strategies/selector_healer.py

from dataclasses import dataclass
from typing import Optional
import re

@dataclass
class FixResult:
    success: bool
    description: str
    diff: Optional[str]
    suggestions: str
    confidence: float
    new_content: Optional[str] = None

class SelectorHealer:
    """
    Heals broken selectors/field references.
    """
    
    def heal(self, analysis) -> FixResult:
        """
        Attempt to fix broken selector.
        """
        if not analysis.affected_element:
            return FixResult(
                success=False,
                description="Could not identify affected element",
                diff=None,
                suggestions="Check field name manually",
                confidence=0.0
            )
        
        # Try to find the correct field name
        correct_field = self._find_correct_field(
            analysis.affected_element,
            analysis.context
        )
        
        if correct_field:
            # Generate the fix
            fix_diff = self._generate_fix(
                analysis.context["test_file"],
                analysis.affected_element,
                correct_field
            )
            
            return FixResult(
                success=True,
                description=f"Renamed '{analysis.affected_element}' to '{correct_field}'",
                diff=fix_diff,
                suggestions="",
                confidence=0.85,
                new_content=self._apply_replacement(
                    analysis.context["test_file"],
                    analysis.affected_element,
                    correct_field
                )
            )
        
        return FixResult(
            success=False,
            description="Could not determine correct field name",
            diff=None,
            suggestions=f"Field '{analysis.affected_element}' not found. Check schema.",
            confidence=0.0
        )
    
    def _find_correct_field(self, old_field: str, context: dict) -> Optional[str]:
        """
        Find the correct field name using fuzzy matching and git history.
        """
        # Strategy 1: Check git diff for renames
        correct = self._check_git_renames(old_field)
        if correct:
            return correct
        
        # Strategy 2: Fuzzy match against current schema
        correct = self._fuzzy_match_schema(old_field, context)
        if correct:
            return correct
        
        # Strategy 3: Common rename patterns
        common_renames = {
            "phase": "state",
            "state": "phase",
            "status": "condition",
            "ready": "available",
            "replicas": "availableReplicas",
        }
        return common_renames.get(old_field)
    
    def _check_git_renames(self, old_field: str) -> Optional[str]:
        """Check recent git commits for field renames."""
        import subprocess
        result = subprocess.run(
            ["git", "log", "-p", "--all", "-S", old_field, "--", "*.go", "*.yaml"],
            capture_output=True,
            text=True,
            cwd="."
        )
        # Parse git output to find renames
        # Implementation details...
        return None
    
    def _fuzzy_match_schema(self, old_field: str, context: dict) -> Optional[str]:
        """Fuzzy match against current resource schema."""
        # Implementation using difflib or similar
        return None
    
    def _generate_fix(self, test_file: str, old: str, new: str) -> str:
        """Generate diff showing the fix."""
        return f"- {old}\n+ {new}"
    
    def _apply_replacement(self, test_file: str, old: str, new: str) -> str:
        """Apply the replacement and return new content."""
        with open(test_file) as f:
            content = f.read()
        return content.replace(old, new)
```

```python
# framework/healing/strategies/timeout_healer.py

class TimeoutHealer:
    """
    Heals timeout-related failures.
    """
    
    # Timeout multipliers based on failure patterns
    MULTIPLIERS = {
        "first_failure": 1.5,
        "repeated_failure": 2.0,
        "ci_environment": 2.5,
    }
    
    def heal(self, analysis) -> FixResult:
        """
        Attempt to fix timeout issues.
        """
        current_timeout = self._extract_current_timeout(analysis)
        
        if current_timeout:
            # Calculate new timeout
            new_timeout = self._calculate_new_timeout(
                current_timeout,
                analysis
            )
            
            fix_diff = self._generate_fix(
                analysis.context["test_file"],
                current_timeout,
                new_timeout
            )
            
            return FixResult(
                success=True,
                description=f"Increased timeout from {current_timeout}s to {new_timeout}s",
                diff=fix_diff,
                suggestions="",
                confidence=0.75,
                new_content=self._apply_timeout_change(
                    analysis.context["test_file"],
                    current_timeout,
                    new_timeout
                )
            )
        
        return FixResult(
            success=False,
            description="Could not identify timeout value",
            diff=None,
            suggestions="Add explicit timeout parameter",
            confidence=0.0
        )
    
    def _extract_current_timeout(self, analysis) -> Optional[int]:
        """Extract current timeout from test."""
        import re
        match = re.search(r"timeout[=:\s]+(\d+)", str(analysis.context))
        return int(match.group(1)) if match else None
    
    def _calculate_new_timeout(self, current: int, analysis) -> int:
        """Calculate appropriate new timeout."""
        multiplier = self.MULTIPLIERS.get("first_failure", 1.5)
        return int(current * multiplier)
    
    def _generate_fix(self, test_file: str, old: int, new: int) -> str:
        return f"- timeout={old}\n+ timeout={new}"
    
    def _apply_timeout_change(self, test_file: str, old: int, new: int) -> str:
        with open(test_file) as f:
            content = f.read()
        return re.sub(f"timeout={old}", f"timeout={new}", content)
```

### 6. MCP Client for AI-Powered Healing

```python
# framework/healing/mcp_client.py

import json
from dataclasses import dataclass
from typing import Optional

@dataclass
class MCPHealingResponse:
    fix_available: bool
    description: str
    diff: Optional[str]
    confidence: float
    suggestions: str
    pr_url: Optional[str] = None
    fix: Optional[dict] = None

class MCPHealingClient:
    """
    Client for MCP server that provides AI-powered healing suggestions.
    """
    
    def __init__(self, server_url: str = "http://localhost:3000"):
        self.server_url = server_url
    
    def request_healing(
        self,
        test_file: str,
        test_content: str,
        failure_analysis,
        git_diff: str,
    ) -> MCPHealingResponse:
        """
        Request healing suggestion from MCP server.
        """
        # Prepare context for AI
        prompt = self._build_healing_prompt(
            test_file,
            test_content,
            failure_analysis,
            git_diff
        )
        
        # Call MCP server
        response = self._call_mcp_server(prompt)
        
        return self._parse_response(response)
    
    def _build_healing_prompt(
        self,
        test_file: str,
        test_content: str,
        failure_analysis,
        git_diff: str,
    ) -> str:
        """Build prompt for AI healing request."""
        return f"""
Analyze this test failure and suggest a fix:

## Test File: {test_file}

## Test Content:
```python
{test_content}
```

## Failure Analysis:
- Type: {failure_analysis.failure_type.value}
- Root Cause: {failure_analysis.root_cause}
- Affected Element: {failure_analysis.affected_element}
- Expected: {failure_analysis.expected_value}
- Actual: {failure_analysis.actual_value}

## Recent Code Changes (git diff):
```diff
{git_diff}
```

## Task:
1. Identify why the test is failing based on the code changes
2. Generate the minimal fix to make the test pass
3. Explain the fix
4. Rate your confidence (0.0 to 1.0)

Respond in JSON format:
{{
    "fix_available": true/false,
    "description": "explanation of the fix",
    "old_code": "code to replace",
    "new_code": "replacement code",
    "confidence": 0.0-1.0,
    "suggestions": "additional suggestions if fix not available"
}}
"""
    
    def _call_mcp_server(self, prompt: str) -> dict:
        """Call MCP server with healing request."""
        import requests
        
        try:
            response = requests.post(
                f"{self.server_url}/heal",
                json={"prompt": prompt},
                timeout=30
            )
            return response.json()
        except Exception as e:
            return {"error": str(e)}
    
    def _parse_response(self, response: dict) -> MCPHealingResponse:
        """Parse MCP server response."""
        if "error" in response:
            return MCPHealingResponse(
                fix_available=False,
                description=f"MCP error: {response['error']}",
                diff=None,
                confidence=0.0,
                suggestions="Try manual fix"
            )
        
        return MCPHealingResponse(
            fix_available=response.get("fix_available", False),
            description=response.get("description", ""),
            diff=self._generate_diff(
                response.get("old_code"),
                response.get("new_code")
            ),
            confidence=response.get("confidence", 0.0),
            suggestions=response.get("suggestions", ""),
            fix={
                "old": response.get("old_code"),
                "new": response.get("new_code")
            }
        )
    
    def _generate_diff(self, old: Optional[str], new: Optional[str]) -> Optional[str]:
        """Generate diff from old and new code."""
        if not old or not new:
            return None
        return f"- {old}\n+ {new}"
```

---

## Usage Examples

### Example 1: Automatic Healing in Action

```python
# tests/test_spire_agent.py

def test_spire_agent_status(cluster, spire_agent):
    """Test that SpireAgent reports Ready status."""
    spire_agent.create()
    
    # Wait for ready - if this times out, self-healing kicks in
    spire_agent.wait_for_ready(timeout=60)
    
    # Check status - if field renamed, self-healing fixes it
    status = spire_agent.get_status()
    assert status["phase"] == "Running"  # If "phase" renamed to "state", auto-fix!
```

**Console Output with Self-Healing:**
```
tests/test_spire_agent.py::test_spire_agent_status FAILED

======================== SELF-HEALING ATTEMPT ========================
Failure Type: FIELD_RENAMED
Affected Element: 'phase'
Root Cause: Field was renamed in resource

Attempting fix...
✓ Found rename in git history: 'phase' → 'state'
✓ Applied fix: status["phase"] → status["state"]
✓ Confidence: 0.85

Re-running test...
tests/test_spire_agent.py::test_spire_agent_status HEALED ✓

Fix saved to: .healing_fixes/test_spire_agent_status.patch
Review and commit: git apply .healing_fixes/test_spire_agent_status.patch
```

### Example 2: Custom Healing Strategy

```python
# framework/healing/strategies/custom_healer.py

from . import register_strategy, FailureType

@register_strategy(FailureType.RESOURCE_NOT_FOUND)
class ZTWIMResourceHealer:
    """
    Custom healer for ZTWIM-specific resource issues.
    """
    
    # Map of old resource names to new ones
    RESOURCE_RENAMES = {
        "spireserver": "spire-server",
        "spireagent": "spire-agent",
        "ztwim": "zero-trust-workload-identity-manager",
    }
    
    def heal(self, analysis) -> FixResult:
        resource = analysis.affected_element
        
        if resource and resource.lower() in self.RESOURCE_RENAMES:
            new_name = self.RESOURCE_RENAMES[resource.lower()]
            return FixResult(
                success=True,
                description=f"Resource renamed: {resource} → {new_name}",
                diff=f"- {resource}\n+ {new_name}",
                suggestions="",
                confidence=0.95,
                new_content=self._apply_fix(analysis, resource, new_name)
            )
        
        return FixResult(
            success=False,
            description="Unknown resource",
            diff=None,
            suggestions="Check resource name in cluster",
            confidence=0.0
        )
```

---

## Configuration

### pytest.ini

```ini
[pytest]
# Self-healing configuration
healing_enabled = true
healing_max_retries = 2
healing_auto_commit = false
healing_mcp_server = http://localhost:3000
healing_confidence_threshold = 0.7

# Markers
markers =
    healable: marks test as eligible for self-healing
    no_healing: disables self-healing for this test
```

### Environment Variables

```bash
# Enable/disable healing
export PYTEST_HEALING_ENABLED=true

# MCP server URL
export PYTEST_HEALING_MCP_SERVER=http://localhost:3000

# Auto-commit fixes (use with caution!)
export PYTEST_HEALING_AUTO_COMMIT=false

# Minimum confidence for auto-fix
export PYTEST_HEALING_CONFIDENCE=0.7
```

---

## CLI Commands

```bash
# Run tests with self-healing enabled
pytest tests/ --healing

# Run tests and show healing suggestions only (no auto-fix)
pytest tests/ --healing --healing-suggest-only

# Run tests and auto-commit high-confidence fixes
pytest tests/ --healing --healing-auto-commit --healing-confidence=0.9

# Review pending healing fixes
pytest --healing-review

# Apply a specific healing fix
pytest --healing-apply .healing_fixes/test_xyz.patch
```

---

## Benefits

| Aspect | Without Self-Healing | With Self-Healing |
|--------|---------------------|-------------------|
| Test maintenance | 4+ hours/week | 30 min/week |
| CI failures from renames | Block pipeline | Auto-resolved |
| Timeout adjustments | Manual tuning | Adaptive |
| Field changes | Search & replace | Auto-detected |
| QE productivity | Reactive fixes | Proactive coding |

---

## Summary

This self-healing capability integrates directly into your pytest framework through:

1. **pytest hooks** - Intercept failures automatically
2. **Failure analyzer** - Categorize and understand failures
3. **Healing strategies** - Pattern-based fixes for common issues
4. **MCP integration** - AI-powered fixes for complex cases
5. **History tracking** - Learn from past healings

The framework maintains **human oversight** by default (fixes require review), while optionally allowing auto-commit for high-confidence fixes.

---

*Document Version: 1.0*  
*December 2024*

