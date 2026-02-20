# Gatekeeper Policy Skill

用于编写和验证 Gatekeeper 策略的完整 Skill，支持 ConstraintTemplate、Constraint 和 gator 测试。

## 目录结构

```
gatekeeper-policy-skill/
├── SKILL.md              # 技能定义和指南
├── README.md             # 使用说明
├── agents/
│   └── openai.yaml       # Agent 配置
├── templates/            # ConstraintTemplate 示例
│   └── k8srequiredlabels.yaml
├── constraints/          # Constraint 示例
│   └── k8srequiredlabels.yaml
└── test/                 # gator 测试文件
    └── test.yaml
```

## 如何使用

### 基本用法

在对话中请求创建策略：

```
请帮我创建一个策略，要求所有 Pod 必须有 app 标签
```

或指定更详细的需求：

```
使用 gatekeeper-policy-skill 创建一个策略：
- 限制容器镜像只能来自 registry.example.com
- 需要应用于所有 Namespace 除了 kube-system
- 需要 gator 测试验证
```

## 输出内容

Skill 会生成以下文件：

### 1. ConstraintTemplate

定义策略逻辑和参数 schema：

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        
        violation[{"msg": msg}] {
          required := input.parameters.labels[_]
          not input.review.object.metadata.labels[required]
          msg := sprintf("Resource must have label: %v", [required])
        }
```

### 2. Constraint

策略实例，指定具体参数：

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pod-must-have-app-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    labels: ["app", "environment"]
```

### 3. Gator 测试

验证策略正确性：

```yaml
kind: TestSuite
apiVersion: test.gatekeeper.sh/v1alpha1
metadata:
  name: required-labels-test
templates:
  - <template-content>
constraints:
  - <constraint-content>
tests:
  - name: label-tests
    cases:
      - name: pod-with-required-labels
        object:
          apiVersion: v1
          kind: Pod
          metadata:
            name: valid-pod
            labels:
              app: myapp
              environment: prod
        assertions:
          - violations: no
      - name: pod-missing-labels
        object:
          apiVersion: v1
          kind: Pod
          metadata:
            name: invalid-pod
        assertions:
          - violations: yes
```

## 常见策略模式

### 1. 必需标签 (Required Labels)

```bash
gatekeeper-policy-skill/templates/k8srequiredlabels.yaml
gatekeeper-policy-skill/constraints/k8srequiredlabels.yaml
```

### 2. 镜像限制 (Image Restrictions)

限制容器镜像只能来自指定 registry：

```
请创建一个策略，只允许镜像来自 registry.cn-hangzhou.aliyuncs.com
```

### 3. 资源限制 (Resource Limits)

要求容器必须设置 CPU/memory 限制：

```
请创建一个策略，要求所有容器必须有 CPU 和 memory 的 limits
```

### 4. 安全上下文 (Security Context)

强制安全设置：

```
请创建一个策略，要求所有 Pod 必须以非 root 用户运行
```

### 5. 命名规范 (Naming Conventions)

强制资源命名模式：

```
请创建一个策略，要求 Deployment 名称必须符合 <team>-<app> 格式
```

### 6. 禁止的字段 (Forbidden Fields)

阻止特定配置：

```
请创建一个策略，禁止使用 latest 标签的镜像
```

## 验证策略

### 安装 Gator

```bash
# 从 Gatekeeper 发布页下载
# https://github.com/open-policy-agent/gatekeeper/releases

# 或使用 go install
go install github.com/open-policy-agent/gatekeeper/v3/cmd/gator@latest
```

### 运行测试

```bash
# 运行所有测试
gator test test/

# 运行特定测试文件
gator test test/test.yaml

# 验证语法
gator verify templates/k8srequiredlabels.yaml
```

### 预期输出

```
✓ passing: pod-with-required-labels
✓ passing: pod-missing-labels

2 passing
```

## 部署到集群

```bash
# 1. 应用 ConstraintTemplate
kubectl apply -f templates/k8srequiredlabels.yaml

# 2. 应用 Constraint
kubectl apply -f constraints/k8srequiredlabels.yaml

# 3. 验证策略状态
kubectl get constrainttemplate
kubectl get k8srequiredlabels

# 4. 测试策略（应该被拒绝）
kubectl run test-pod --image=nginx
```

## 故障排除

### Gator 测试失败

```bash
# 查看详细错误
gator test test/ -v

# 检查 YAML 语法
yamllint templates/*.yaml constraints/*.yaml test/*.yaml
```

### Rego 语法错误

常见错误：
- 缺少 `violation[{"msg": msg}]` 格式
- package 名称与 template 名称不匹配
- 参数访问错误（使用 `input.parameters`）

### 策略不生效

检查：
1. Gatekeeper 是否正确安装
2. ConstraintTemplate 是否创建成功
3. Constraint 的 match 条件是否正确
4. 是否有 namespace 排除配置

## 最佳实践

### Rego 编写

- ✅ 使用具体的错误消息，包含资源名称
- ✅ 处理缺失字段的情况
- ✅ 避免复杂的嵌套循环
- ✅ 使用 `sprintf` 格式化消息

### 测试覆盖

- ✅ 至少一个通过用例
- ✅ 至少一个失败用例
- ✅ 边界条件测试
- ✅ namespace 匹配测试

### 性能优化

- ✅ 使用 `break` 提前退出
- ✅ 避免不必要的迭代
- ✅ 使用集合操作代替循环
- ✅ 限制 match 范围

## 参考资源

- [Gatekeeper 官方文档](https://open-policy-agent.github.io/gatekeeper/)
- [Rego 语言指南](https://www.openpolicyagent.org/docs/latest/policy-language/)
- [Gator CLI 文档](https://open-policy-agent.github.io/gatekeeper/website/docs/gator/)
- [Gatekeeper 示例库](https://github.com/open-policy-agent/gatekeeper/tree/master/charts/gatekeeper/templates)
- [OPA Rego 最佳实践](https://github.com/open-policy-agent/opa/blob/main/docs/best-practices.md)
