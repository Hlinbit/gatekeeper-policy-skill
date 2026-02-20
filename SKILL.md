---
name: gatekeeper-policy-skill
description: 编写符合 Gatekeeper 规范的 ConstraintTemplate、Constraint 和 Rego 策略，支持 gator 测试验证
---

# Gatekeeper Policy Skill

Use this skill to create production-ready Gatekeeper policies with proper ConstraintTemplate definitions, Constraints, and Rego rules that pass gator validation.

## Output Format

Always produce:

1. **ConstraintTemplate** - CRD definition with embedded Rego
2. **Constraint** - Instance of the template with parameters
3. **Test Cases** - gator test files for validation
4. **Test Results** - Expected pass/fail scenarios

## Workflow

1. **Understand the policy requirement** - Identify what Kubernetes resources and rules to enforce
2. **Design the ConstraintTemplate** - Define parameters, CRD schema, and Rego logic
3. **Create the Constraint** - Instantiate template with specific values
4. **Write gator tests** - Create test cases covering pass and fail scenarios
5. **Validate with gator** - Ensure all tests pass before delivery

## ConstraintTemplate Guidelines

### Structure

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: <template-name>
spec:
  crd:
    spec:
      names:
        kind: <ConstraintKind>
      validation:
        openAPIV3Schema:
          type: object
          properties:
            <parameters>
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package <package-name>
        
        violation[{"msg": msg}] {
          # violation logic
        }
```

### Rego Best Practices

- **Package naming**: Use `package <template_name>` matching the template name
- **Violation rule**: Always use `violation[{"msg": msg}]` format
- **Input access**: Use `input.review.object` for the resource being validated
- **Parameters**: Access via `input.parameters.<param_name>`
- **Error messages**: Be specific and actionable, include resource name and violation reason
- **Performance**: Use efficient iteration, avoid nested loops when possible

### Common Patterns

```rego
# Check required field exists and is not empty
violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  not container.image
  msg := sprintf("Container <%v> must have an image specified", [container.name])
}

# Check field against allowed values
violation[{"msg": msg}] {
  allowed := {p | p := input.parameters.allowedImages[_]}
  container := input.review.object.spec.containers[_]
  not startswith(container.image, allowed[_])
  msg := sprintf("Container <%v> has disallowed image <%v>", [container.name, container.image])
}

# Check required label exists
violation[{"msg": msg}] {
  required := input.parameters.requiredLabels[_]
  not input.review.object.metadata.labels[required]
  msg := sprintf("Resource must have label: %v", [required])
}
```

## Constraint Guidelines

### Structure

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: <ConstraintKind>
metadata:
  name: <constraint-name>
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["default"]
  parameters:
    <parameter-values>
```

### Match Patterns

```yaml
# Match all Pods in specific namespaces
match:
  kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  namespaces: ["default", "production"]

# Match Deployments and ReplicaSets
match:
  kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment", "ReplicaSet"]

# Exclude certain namespaces
excludedNamespaces: ["kube-system", "gatekeeper-system"]
```

## Gator Test Guidelines

### Test Structure

```yaml
# test/test.yaml
kind: TestSuite
apiVersion: test.gatekeeper.sh/v1alpha1
metadata:
  name: <test-suite-name>
templates:
  - <template-content>
constraints:
  - <constraint-content>
tests:
  - name: <test-name>
    template: <template-name>
    kind: <constraint-kind>
    constraint: <constraint-name>
    cases:
      - name: <case-name>
        object: <test-object>
        assertions:
          - violations: no
          # or
          - violations: yes
            message: <expected-message>
```

### Assertion Types

```yaml
# No violations expected
assertions:
  - violations: no

# Violations expected with specific message
assertions:
  - violations: yes
    message: "must have"

# Specific number of violations
assertions:
  - violations: 2
```

### Test Best Practices

- **Cover both pass and fail cases** - Test valid and invalid resources
- **Test edge cases** - Empty values, missing fields, boundary conditions
- **Verify error messages** - Ensure messages are clear and actionable
- **Test match/exclude logic** - Verify namespace and kind matching works

## Quality Bar

### ConstraintTemplate
- [ ] CRD schema properly defines all parameters
- [ ] Rego package name matches template name
- [ ] Violation messages are specific and actionable
- [ ] Handles missing fields gracefully
- [ ] Follows Rego best practices

### Constraint
- [ ] Match criteria clearly defined
- [ ] Parameters match template schema
- [ ] Namespace exclusions considered
- [ ] Enforcement action specified (deny/dryrun)

### Gator Tests
- [ ] At least one passing test case
- [ ] At least one failing test case
- [ ] Edge cases covered
- [ ] All tests pass with `gator test`

## Common Policy Patterns

### 1. Required Labels
Enforce specific labels on resources.

### 2. Image Restrictions
Restrict container images to allowed registries.

### 3. Resource Limits
Require CPU/memory limits on containers.

### 4. Security Context
Enforce security settings (runAsNonRoot, readOnlyRootFilesystem).

### 5. Naming Conventions
Enforce resource naming patterns.

### 6. Forbidden Fields
Block specific field values or configurations.

## Validation Commands

```bash
# Run gator tests
gator test test/

# Verify ConstraintTemplate syntax
gator verify <template.yaml>

# Test against cluster (if available)
kubectl apply -f template.yaml
kubectl apply -f constraint.yaml
```

## Reference

- [Gatekeeper Documentation](https://open-policy-agent.github.io/gatekeeper/)
- [Rego Documentation](https://www.openpolicyagent.org/docs/latest/policy-language/)
- [Gator CLI](https://open-policy-agent.github.io/gatekeeper/website/docs/gator/)
- [ConstraintTemplate Examples](https://github.com/open-policy-agent/gatekeeper/tree/master/charts/gatekeeper/templates)
