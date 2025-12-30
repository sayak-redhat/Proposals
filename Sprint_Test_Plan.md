# Test Plan

## Overview

This test plan follows a **sprint-based testing strategy** where QE work begins when the EP is approved. Testing is executed in parallel with development, with test preparation happening before PRs arrive and test execution occurring as PRs are submitted.

**Sprint Duration**: 3 Weeks (21 Days)  
**QA Team**: 2 Engineers (QE1 & QE2)  
**PRs per EP**: Varies (typically 4-8 PRs)

---

## Testing Strategy

### When Does QE Work Start?

QE work begins **when the EP is approved**, not when PRs are created. This enables parallel preparation while developers code.

```
EP APPROVED ──► TEST PREPARATION ──► PR READY ──► TEST EXECUTION
                (Parallel with Dev)
```

### Testing Phases

| Phase | When | Duration | Scope |
|-------|------|----------|-------|
| **Test Preparation** | EP Approved (Week 0) | 5-7 days | Test plan, test cases, E2E scripts |
| **Premerge Testing** | PRs Ready (Week 1-3) | 2-3 days/PR | PR functionality + affected components |
| **Integration Testing** | After every 2 PRs | 0.5 day | PR interactions |
| **Regression Testing** | End of Sprint | 2-3 days | Full product |
| **Buffer Period** | Final days | 2-3 days | Bug fixes, sign-off |

---

## Test Preparation Phase (Before Sprint)

Once EP is approved, QE prepares while developers code:

| Activity | Duration | Deliverable |
|----------|----------|-------------|
| Test Plan Design | 2-3 days | Test plan document |
| Test Case Writing | 3-4 days | Test cases per PR |
| E2E Script Development | 5-7 days | Automated E2E tests |
| Test Environment Setup | 1-2 days | Test cluster ready |

---

## Premerge Testing (Per PR)

Each PR undergoes the following testing before merge:

### Functional Testing (1 day)

| IN SCOPE | OUT OF SCOPE |
|----------|--------------|
| New code/features in PR | Existing unchanged features |
| Modified functionality | Unrelated functionality |
| New API fields/parameters | APIs not touched by PR |
| PR-specific validations | System-wide validations |

### Component Testing (0.5-1 day)

| IN SCOPE | OUT OF SCOPE |
|----------|--------------|
| Components modified by PR | Components not changed |
| Controllers affected | Unaffected controllers |
| ConfigMaps/Secrets changed | Unchanged configs |

### E2E Sanity Testing (0.5 day)

| IN SCOPE | OUT OF SCOPE |
|----------|--------------|
| Core happy-path flows | All edge cases |
| Basic "does it work?" checks | Comprehensive coverage |
| Installation sanity | All install scenarios |
| Main workflow sanity | All workflows |

---

## Integration Testing

**When**: After every 2 PRs merged  
**Duration**: 0.5 day per checkpoint

| IN SCOPE | OUT OF SCOPE |
|----------|--------------|
| PR interactions | Individual PR testing (already done) |
| Combined config effects | Isolated behavior |
| Data flow between PRs | Single PR validation |

### Integration Checkpoints

| Checkpoint | When | Focus |
|------------|------|-------|
| INT-1 | Day 7 | First 2-3 PRs work together |
| INT-2 | Day 12 | Extended PR set integration |
| INT-3 | Day 15 | Complete EP functionality |

---

## Regression Testing

**When**: All PRs merged to master test branch (Day 16-18)  
**Duration**: 2-3 days

| IN SCOPE | OUT OF SCOPE |
|----------|--------------|
| ALL product functionality | New feature development |
| All supported configurations | Experimental configs |
| Upgrade/downgrade paths | Unsupported versions |
| All topologies (SNO, HA, HyperShift) | Unsupported topologies |
| Performance sanity | Performance optimization |
| Recovery scenarios | Chaos engineering |

### Regression Test Categories

| Category | Priority |
|----------|----------|
| Installation & Setup | P1-P2 |
| Core CRUD Operations | P1 |
| End-to-End Workflows | P1 |
| EP-Specific Features | P1 |
| Upgrade & Compatibility | P1-P2 |
| Recovery & Reliability | P2 |
| Scale & Performance | P2-P3 |
| Security | P2 |

---

## Sprint Timeline

```
WEEK 0 (Prep)        WEEK 1-2 (Execution)     WEEK 3 (Regression)
─────────────        ────────────────────     ───────────────────
                     
Test Plan Design     PR1 ████ (QE1)           PRn ██ (QE1)
Test Cases Writing   PR2 ████ (QE2)              │
E2E Script Dev       PR3 ████ (QE1)              ▼
Environment Setup    PR4 ████ (QE2)          ┌──────────┐
       │             PR5 ████ (QE1)          │REGRESSION│
       │                 │                   │ Day16-18 │
       ▼                 ▼                   └──────────┘
┌──────────────┐     Integration                  │
│ READY FOR    │     Checkpoints                  ▼
│ PR TESTING   │     (Day 7, 12)             ┌──────────┐
└──────────────┘                             │ BUFFER   │
                                             │ Day19-21 │
                                             └──────────┘
```

---

## Exit Criteria

### Test Preparation Exit Criteria
- [ ] Test plan document completed
- [ ] Test cases written for all expected PRs
- [ ] E2E scripts implemented and merged
- [ ] Test environment provisioned

### Premerge Exit Criteria (Per PR)
- [ ] All functional tests pass
- [ ] All component tests pass
- [ ] E2E sanity passes
- [ ] No P1/P2 bugs open

### Integration Exit Criteria
- [ ] All merged PRs work together
- [ ] No integration conflicts
- [ ] E2E flow works with combined changes

### Regression Exit Criteria
- [ ] All P1 tests pass (100%)
- [ ] P2 tests pass (>95%)
- [ ] No blocker/critical bugs
- [ ] Upgrade path validated

### Sprint Exit Criteria
- [ ] All EP PRs merged and tested
- [ ] Regression complete
- [ ] Sign-off from both QEs
- [ ] All P1/P2 bugs resolved

---

## E2E and Integration Tests

### E2E Tests
- E2E tests will be developed during test preparation phase (Week 0)
- Scripts will cover core workflows and EP-specific functionality
- E2E sanity suite runs per PR (subset of full suite)
- Full E2E suite runs during regression

### Integration Tests
- Integration tests verify PR interactions
- Run at checkpoints (Day 7, 12, 15)
- Focus on data flow and combined functionality

### Unit Tests
- Unit tests are developer responsibility
- QE verifies unit test coverage during PR review
- All unit tests must pass before premerge testing begins

---

## Challenging Areas to Test

Based on EP complexity, identify:

1. **Complex interactions**: Components that interact in non-obvious ways
2. **State management**: Areas where state transitions are complex
3. **Error handling**: Failure scenarios and recovery paths
4. **Upgrade paths**: Configuration migration and compatibility
5. **Scale scenarios**: Performance under load
6. **Security boundaries**: RBAC, secrets, network policies

---

## Additional Testing for Managed OpenShift

For managed service offerings (ROSA, ARO, etc.):

- [ ] Test with managed cluster configurations
- [ ] Verify compatibility with service limitations
- [ ] Test upgrade paths specific to managed services
- [ ] Validate SRE operational procedures
