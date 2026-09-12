---
description: "**Issue:** MAR-28 **Date:** 2026-04-23 **Status:** In Review"
---

# Spec: Network Rules

**Issue:** MAR-28
**Date:** 2026-04-23
**Status:** In Review

## Overview

Network rules manage firewall configurations per project, specifying ports, protocols (TCP/UDP), rule types (allow/deny), and optional source IP restrictions. Each provider implements `NetworkRuleManagerInterface` to translate `NetworkRuleConfig` objects into provider-specific API calls.

## Current Implementation

### Key Classes / Files

| File | Role |
|------|------|
| `app/Config/NetworkRuleConfig.php` | Value object with name, port, type, ruleType, fromIp |
| `app/Deployment/Contracts/NetworkRuleManagerInterface.php` | Interface defining `plan()` and `apply()` |
| `app/Deployment/Providers/Ploi/PloiNetworkRuleManager.php` | Ploi implementation |
| `app/Deployment/Providers/Forge/ForgeNetworkRuleManager.php` | Forge implementation |
| `tests/Unit/Config/NetworkRuleConfigTest.php` | Unit tests |

### NetworkRuleConfig

```php
final class NetworkRuleConfig
{
    public function __construct(
        private readonly string $name,
        private readonly int $port,
        private readonly string $type = 'tcp',
        private readonly string $ruleType = 'allow',
        private readonly ?string $fromIp = null,
    ) {}

    public function name(): string { return $this->name; }
    public function port(): int { return $this->port; }
    public function type(): string { return $this->type; }
    public function ruleType(): string { return $this->ruleType; }
    public function fromIp(): ?string { return $this->fromIp; }
}
```

### NetworkRuleManagerInterface

```php
interface NetworkRuleManagerInterface
{
    /**
     * @return array<string>
     */
    public function plan(DeploymentContext $context): array;

    /**
     * @param array<string, NetworkRuleConfig> $rules
     */
    public function apply(SiteContext $site, array $rules): OperationResult;
}
```

## Functional Requirements

**FR-001 — Plan iterates all project network rules**
`plan()` iterates `$context->project->networkRules()` and generates a summary string per rule: `"Create network rule: NAME (port PORT/TYPE, RULETYPE)"`.

**FR-002 — Apply creates rules via provider API**
Both providers accept the full `NetworkRuleConfig` object and extract individual fields:
- Ploi: `$server->networkRules()->create($name, $port, $type, $fromIp, $ruleType)`
- Forge: `$forge->createFirewallRule($serverId, ['name' => ..., 'port' => ..., 'type' => ..., 'ip_address' => $fromIp])`

**FR-003 — Rule name is used as array key in apply**
The `apply()` method receives `array<string, NetworkRuleConfig>` — the string key is the rule name and is used in error messages.

**FR-004 — Apply stops on first failure**
If a rule creation throws an exception, `apply()` returns `OperationResult::fail()` immediately with the rule name in the message.

## Configuration Interface

```yaml
networkRules:
  Allow HTTP:
    port: 80
    type: tcp
    ruleType: allow
  Allow Redis:
    port: 6379
    type: tcp
    ruleType: allow
    fromIp: "10.0.0.0/8"
  Block SSH:
    port: 22
    type: tcp
    ruleType: deny
    fromIp: "0.0.0.0/0"
```

## Data Contracts

```php
// Config object
NetworkRuleConfig { name: string, port: int, type: string, ruleType: string, fromIp: ?string }

// Interface signatures
plan(DeploymentContext $context): array<string>
apply(SiteContext $site, array<string, NetworkRuleConfig> $rules): OperationResult
```

## Edge Cases

- **Empty networkRules:** `plan()` returns `[]` (iterating an empty array)
- **fromIp is null:** Provider APIs receive `null` — some providers may reject or interpret as "any"
- **Rule type is 'deny':** Ploi passes it as a parameter; Forge may not support deny rules (not checked at Shipper level)

## Acceptance Criteria

- [ ] `NetworkRuleConfig` has correct default values: `type='tcp'`, `ruleType='allow'`, `fromIp=null`
- [ ] `plan()` generates one summary string per network rule
- [ ] `apply()` calls the correct provider API method for each rule
- [ ] Exceptions are caught and returned as `OperationResult::fail()` with rule name
- [ ] Provider implementations are symmetric (same interface, same behavior)