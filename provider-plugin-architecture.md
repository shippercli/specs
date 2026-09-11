# Spec: Provider Plugin Architecture

**Issue:** MAR-49
**Date:** 2026-05-10
**Status:** Implemented

## Overview

Shipper uses a plugin architecture for deployment providers. Providers are separate Composer packages that implement the `ShipperPluginInterface`. The core CLI discovers and loads providers at runtime.

## Plugin Interface

```php
interface ShipperPluginInterface
{
    /**
     * Register the plugin's commands with the application.
     */
    public function registerCommands(Application $app): void;

    /**
     * Boot the plugin (after all services are registered).
     */
    public function boot(): void;

    /**
     * Get the plugin's service providers.
     *
     * @return array<string, class-string>
     */
    public function providers(): array;
}
```

## Provider Service Provider

Each provider plugin has a `ProviderServiceProvider` that registers the provider:

```php
interface DeploymentProviderServiceProviderInterface
{
    public function register(): void;
    public function getProviderName(): string;
    public function createProvider(array $config): DeploymentProviderInterface;
}
```

## Discovery

Providers are discovered via:
1. `shipper.providers` in `shipper.yml` config
2. Composer plugins: packages with `shipper-plugin` type in their `extra` field
3. Manual registration via `$app->registerProvider()`

## Packages

| Package | Repository | Purpose |
|--------|------------|---------|
| `shippercli/contracts` | github.com/shippercli/contracts | Interfaces and contracts |
| `shippercli/provider-ploi` | github.com/shippercli/provider-ploi | Ploi provider implementation |
| `shippercli/provider-forge` | github.com/shippercli/provider-forge | Forge provider implementation |
| `shippercli/provider-cpanel` | github.com/shippercli/provider-cpanel | cPanel provider implementation |
| `shippercli/provider-easypanel` | github.com/shippercli/provider-easypanel | EasyPanel provider implementation |
| `shippercli/provider-cpanel` | github.com/shippercli/provider-cpanel | cPanel provider implementation |

## Configuration

Providers are configured in `shipper.yml`:

```yaml
providers:
  ploi:
    api_key: "${PLOI_API_KEY}"
    server_id: "105556"
```

Each provider package registers itself via its service provider.
