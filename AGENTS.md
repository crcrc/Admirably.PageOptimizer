# AGENTS.md

This file provides guidance to Codex (Codex.ai/code) when working with code in this repository.

## Project Overview

**Toodle.PageOptimizer** is a .NET NuGet library for ASP.NET Core that manages SEO metadata, HTTP Link headers for resource hinting, static file caching, JSON-LD structured data (breadcrumbs), sitemap.xml generation, and response compression.

Published at: https://www.nuget.org/packages/Toodle.PageOptimizer  
Current version: 1.2.1 (see `CHANGELOG.md`)

## Build Commands

```bash
# Build
dotnet build

# Release build + generates NuGet package automatically (GeneratePackageOnBuild=true)
dotnet build --configuration Release

# Pack explicitly
dotnet pack --configuration Release Toodle.PageOptimizer/Toodle.PageOptimizer.csproj
```

**No test projects exist** — this library is validated through consuming applications.

## Target Frameworks

Multi-targeted: `net6.0`, `net7.0`, `net8.0`, `net9.0`, `net10.0`  
C# 11, nullable enabled, implicit usings **disabled** (explicit `using` statements required).

## Architecture

### Registration vs. Configuration Split

There are two distinct phases, both required in `Program.cs`:

1. **`AddPageOptimizer()`** — DI registration (before `app.Build()`). Registers `IPageOptimizerService` as scoped and creates the singleton `PageOptimizerConfig`.
2. **`ConfigurePageOptimizer()`** — App configuration (after `app.Build()`). Returns a fluent `PageOptimizerApp` builder. Calling `UsePageOptimizer()` locks the config via `PageOptimizerConfig.Lock()`, preventing further modification.

### Core Components

| Component | Lifetime | Role |
|-----------|----------|------|
| `PageOptimizerConfig` | Singleton | Global defaults (site name, base URL, preload/preconnect resources). Locked at startup. |
| `IPageOptimizerService` | Scoped | Per-request metadata (title, description, canonical URL, breadcrumbs, resource hints). |
| `PageOptimizerApp` | Builder | Fluent API for startup configuration. |

### Middleware Pipeline

`UsePageOptimizer()` registers middleware in this order:
1. `AppendLinkHeaderMiddleware` — Merges per-request and global Link headers into HTTP response
2. `StaticFileCacheHeaderMiddleware` — Applies `Cache-Control` headers to static file responses
3. `SitemapMiddleware` — Intercepts requests to `/sitemap.xml` (if `.ServeSitemap()` called)
4. Optional: `RequestLocalization`, `ResponseCompression` (Brotli/Gzip)

### Sitemap Subsystem (`Sitemap/`)

- `ISitemapSource` — Plugin interface; implement or use `FuncSitemapSource` (adapter wrapping a `Func`)
- `SitemapService` — Orchestrates generation with `SemaphoreSlim`-based double-checked locking to prevent cache stampede
- `SitemapGenerator` — Stateless XML serialization
- Register sources via `.AddSitemapSource(...)` on the `IServiceCollection` builder

### Tag Helper

`<page-optimizer />` renders `<meta>` tags, canonical `<link>`, and JSON-LD breadcrumb `<script>`. Must be registered in `_ViewImports.cshtml`.

## Key Patterns

- **Config locking**: `PageOptimizerConfig` throws `InvalidOperationException` if mutated after `Lock()`. Always check `EnsureNotLocked()` / `EnsureConfigNotLocked()` when adding new config properties.
- **Fluent builder**: `PageOptimizerApp` methods return `this` for chaining. New features should follow this pattern.
- **Explicit usings**: No implicit usings — always add `using` directives explicitly.
- **XML docs**: `GenerateDocumentationFile=true` — public API members should have `/// <summary>` comments.
