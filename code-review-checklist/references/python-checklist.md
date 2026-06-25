# Python Code Review Checklist

## 1. Security & Secrets Exposure
- [ ] No hardcoded credentials, API keys, tokens, or passwords in code
- [ ] No use of eval() or exec() on user-supplied input
- [ ] No unsafe deserialization (pickle.loads on untrusted data, yaml.load without Loader=yaml.SafeLoader)
- [ ] SQL queries use parameterized statements, not f-strings or .format()
- [ ] File paths are validated/sanitized before use (path traversal risk)
- [ ] Sensitive data not logged (passwords, tokens, PII)
- [ ] Dependencies pinned to specific versions in requirements.txt / pyproject.toml
- [ ] No use of subprocess with shell=True on user input

## 2. Performance & Resource Limits
- [ ] No N+1 query patterns (DB calls inside loops)
- [ ] Large datasets use pagination or streaming, not full loads into memory
- [ ] Expensive operations not repeated in tight loops
- [ ] Appropriate use of caching where results are reusable
- [ ] Generator expressions used instead of list comprehensions where iteration-only
- [ ] No unbounded recursion without a base case or depth limit

## 3. Test Coverage
- [ ] All public functions and methods have at least one test
- [ ] Edge cases covered: empty input, None, boundary values, error paths
- [ ] Tests use assertions on actual output, not just "no exception raised"
- [ ] Mocks are scoped correctly — not hiding real logic that should be tested
- [ ] Test names clearly describe the scenario being tested
- [ ] Integration tests present for DB/API interactions

## 4. Documentation & Comments
- [ ] All public functions/classes have docstrings
- [ ] Docstrings document args, return values, and raised exceptions
- [ ] Complex logic has inline comments explaining *why*, not just *what*
- [ ] TODO/FIXME comments have an owner or ticket reference
- [ ] Type hints present on function signatures

## 5. Coding Standards & Style
- [ ] PEP 8 compliance (line length, naming conventions, whitespace)
- [ ] Consistent use of type hints throughout the module
- [ ] No bare except: clauses — always catch specific exceptions
- [ ] Exceptions are re-raised or logged, not silently swallowed
- [ ] No mutable default arguments (def foo(x=[]) anti-pattern)
- [ ] No unused imports or dead code
- [ ] Constants defined at module level, not magic numbers inline

## 6. Rollback Safety & Operational Readiness
- [ ] Database migrations are reversible (down migration exists)
- [ ] Schema changes are backward-compatible with the previous app version
- [ ] Feature flags used for high-risk changes
- [ ] New environment variables have documented defaults
- [ ] Breaking API changes are versioned
- [ ] Logging added for new code paths at appropriate levels
