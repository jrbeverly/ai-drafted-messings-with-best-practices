# Best Practices Documentation

## Overview

Best practice reference for a modern solo developer tech stack. Each file should be a **concise cheat sheet** — not a tutorial or implementation guide.

**Tech Stack:** Hugo, Vue.js, TypeScript, C#/.NET, AWS (Lambda, DynamoDB, S3, CloudFront), Terraform, Gitea Actions

**Target format per file:**

- ~30-80 lines max
- Tagline: one sentence explaining what it is
- Why it matters: 2-3 bullet points
- Key recommendations: bullet points with package names, config snippets, or short patterns
- Pitfalls to avoid: brief list
- No full implementation dumps — just the "what" and "which", not the "how to build it from scratch"

**Status legend:** `[x]` = file exists (needs condensing), `[ ]` = not yet created

---

## 1. Hugo Static Sites

### 1.1 Hugo Basics

> `docs/01-hugo-static-sites/01-hugo-basics/`

- [x] hugo-structure-organization.md
- [x] hugo-content-management.md
- [x] hugo-theme-development.md
- [x] hugo-build-optimization.md
- [x] hugo-shortcodes-partials.md
- [x] hugo-multilingual-sites.md
- [x] hugo-asset-pipeline.md
- [x] hugo-taxonomies-sections.md
- [x] hugo-self-hosted-dependencies.md
- [x] hugo-resource-fingerprinting.md
- [x] hugo-data-files.md
- [x] hugo-modules.md
- [x] hugo-deployment-strategies.md

### 1.2 SEO & Social Media

> `docs/01-hugo-static-sites/02-seo-social-media/`

- [x] open-graph-meta-tags.md
- [x] opengraph-article-extensions.md
- [x] twitter-card-images.md
- [x] twitter-card-types.md
- [x] structured-data-schema-org.md
- [x] schema-org-types.md
- [x] sitemaps-robots-txt.md
- [x] sitemap-index.md
- [x] canonical-urls.md
- [x] hreflang-tags.md
- [x] meta-descriptions-titles.md
- [x] meta-tags-comprehensive.md
- [x] social-media-image-generation.md
- [x] rss-feeds.md
- [x] atom-feeds.md
- [x] json-feed.md
- [x] dublin-core-metadata.md
- [x] geo-meta-tags.md
- [x] linkedin-specific-meta.md
- [x] pinterest-rich-pins.md
- [x] web-monetization.md
- [x] rel-attributes.md
- [x] link-prefetching-dns-prefetch.md
- [x] apple-smart-app-banner.md

### 1.3 Web Standards & Discovery

> `docs/01-hugo-static-sites/03-web-standards/`

- [x] well-known-directory.md
- [x] security-txt.md
- [x] robots-txt-advanced.md
- [x] ads-txt.md
- [x] app-ads-txt.md
- [x] sellers-json.md
- [x] favicon-modern-formats.md
- [x] apple-touch-icons.md
- [x] browserconfig-xml.md
- [x] webmanifest-pwa.md
- [x] opensearch-xml.md
- [x] humans-txt.md
- [x] canonical-urls.md
- [x] hreflang-tags.md
- [x] open-graph-protocol.md
- [x] twitter-cards.md
- [x] schema-org-markup.md
- [x] structured-data-json-ld.md
- [x] rss-atom-feeds.md
- [x] sitemap-xml-advanced.md
- [x] meta-tags-seo.md
- [x] security-headers.md
- [x] csp-content-security-policy.md
- [x] permissions-policy.md

### 1.4 Hugo on AWS

> `docs/01-hugo-static-sites/04-hugo-on-aws/`

- [x] hugo-s3-deployment.md
- [x] hugo-cloudfront-integration.md
- [x] hugo-cicd-aws.md
- [x] hugo-lambda-edge-rewrites.md
- [x] hugo-prerendering-strategies.md

### 1.5 Performance

> `docs/01-hugo-static-sites/05-performance/`

- [x] image-optimization-formats.md
- [x] lazy-loading-images.md
- [x] critical-css.md
- [x] font-loading-strategies.md
- [x] asset-bundling-minification.md
- [x] cdn-configuration.md
- [x] http2-http3.md
- [x] preload-prefetch-preconnect.md
- [x] resource-hints.md
- [x] brotli-compression.md

### 1.6 Accessibility

> `docs/01-hugo-static-sites/06-accessibility/`

- [x] aria-labels-roles.md
- [x] semantic-html.md
- [x] keyboard-navigation.md
- [x] screen-reader-optimization.md
- [x] color-contrast-wcag.md
- [x] focus-management.md
- [x] skip-navigation-links.md
- [x] accessible-forms.md

---

## 2. Vue.js & TypeScript Frontend

### 2.1 Vue.js

> `docs/06-frontend/01-vue/`

- [x] composition-api-basics.md
- [x] vue-component-structure.md
- [x] vue-composables.md
- [x] vue-router-patterns.md
- [x] vue-testing-vitest.md
- [x] vue-performance-optimization.md
- [x] vue-ssr-considerations.md
- [x] vue-error-handling.md

### 2.2 State Management

> `docs/06-frontend/03-state-management/`

- [x] pinia-state-management.md — Client state only (UI prefs, selections)
- [x] tanstack-query-vue.md — Server state (queries, mutations, caching)

### 2.3 TypeScript

> `docs/06-frontend/02-typescript/`

- [x] typescript-strict-mode.md
- [x] typescript-type-safety.md
- [x] typescript-generics.md
- [x] typescript-utility-types.md
- [x] typescript-module-resolution.md
- [x] typescript-declaration-files.md
- [x] typescript-vue-integration.md
- [x] typescript-branded-types.md
- [x] typescript-discriminated-unions.md
- [x] typescript-internal-libraries.md
- [x] typescript-monorepo-packages.md

### 2.4 Build & Tooling

> `docs/06-frontend/04-build-tooling/`

- [x] vite-configuration.md
- [x] eslint-prettier-setup.md
- [x] package-json-structure.md
- [x] npm-vs-pnpm-vs-yarn.md
- [x] configuration-files-vs-env-vars.md
- [x] typescript-path-aliases.md
- [x] bundling-third-party-deps.md
- [x] cicd-validation-over-hooks.md
- [x] frontend-error-boundaries.md
- [x] vue-production-builds.md
- [x] vue-environment-configs.md

### 2.5 Vue.js on AWS

> `docs/06-frontend/05-vue-on-aws/`

- [x] vue-s3-deployment.md
- [x] spa-routing-s3-cloudfront.md
- [x] vue-cloudfront-optimization.md

---

## 3. C# & .NET Backend

### 3.1 Modern C# Practices

> `docs/03-csharp/01-modern-practices/`

- [x] csharp-12-features.md
- [x] csharp-13-features.md
- [x] nullable-reference-types.md
- [x] record-types.md
- [x] pattern-matching.md
- [x] async-await-best-practices.md
- [x] linq-optimization.md
- [x] collections-performance.md
- [x] span-memory.md
- [x] string-interpolation-handlers.md
- [x] init-only-properties.md
- [x] file-scoped-namespaces.md
- [x] global-usings.md
- [x] raw-string-literals.md
- [x] required-members.md
- [x] primary-constructors.md
- [x] collection-expressions.md
- [x] ref-readonly-parameters.md
- [x] with-expressions.md
- [x] top-level-statements.md
- [x] target-typed-new.md
- [x] default-interface-methods.md
- [x] generic-attributes.md
- [x] lambda-improvements.md

### 3.2 .NET Core

> `docs/03-csharp/03-dotnet/`

- [x] dependency-injection.md
- [x] dependency-injection-lifetimes.md
- [x] configuration-options-pattern.md
- [x] strongly-typed-configuration.md
- [x] logging-ilogger.md
- [x] logging-structured-serilog.md
- [x] middleware-aspnet.md
- [x] health-checks.md
- [x] background-services.md
- [x] httpclient-factory.md
- [ ] dependency-injection-keyed-services.md
- [ ] logging-log-scopes.md
- [ ] hosted-services.md
- [ ] httpclient-resilience.md — Polly, retry policies
- [ ] mediatr-cqrs.md
- [ ] fluentvalidation.md
- [ ] rate-limiting-aspnetcore.md
- [ ] response-compression.md
- [ ] result-pattern.md — Avoid exceptions for control flow
- [ ] exception-filters.md

### 3.3 Testing

> `docs/03-csharp/02-testing/`

- [x] xunit-basics.md
- [x] unit-testing-best-practices.md
- [x] integration-testing.md
- [x] mocking-with-nsubstitute.md
- [x] test-data-builders.md
- [ ] test-naming-conventions.md
- [ ] code-coverage.md
- [ ] data-driven-tests.md — Theory, InlineData, MemberData
- [ ] file-based-test-data.md
- [ ] snapshot-testing.md — Verify
- [ ] test-fixtures-shared-context.md
- [ ] test-parallelization.md
- [ ] integration-tests-webapplicationfactory.md
- [ ] testcontainers.md
- [ ] architectural-tests.md — NetArchTest
- [ ] contract-testing.md — Pact
- [ ] mutation-testing.md — Stryker.NET

### 3.4 ASP.NET Core Web APIs

> `docs/03-csharp/04-aspnet-web-api/`

- [x] minimal-api-basics.md
- [x] error-codes-pattern.md
- [x] api-error-responses.md
- [x] error-code-catalog.md
- [x] api-versioning-strategies.md
- [x] mime-type-versioning.md
- [x] content-negotiation.md
- [x] model-binding.md
- [x] custom-model-binders.md
- [x] action-filters.md
- [x] result-types-aspnet.md
- [x] response-formatters.md
- [x] input-formatters.md
- [x] attribute-routing.md
- [x] route-constraints.md
- [x] link-generation.md
- [x] api-controllers-vs-minimal.md
- [x] controller-conventions.md
- [x] status-code-pages.md
- [x] file-upload-aspnet.md
- [x] file-download-streaming.md
- [x] static-files-middleware.md
- [x] endpoint-filters.md
- [x] middleware-pipeline.md
- [x] problem-details.md
- [x] cors-configuration.md
- [x] custom-middleware.md
- [x] request-validation.md
- [x] response-formatting.md
- [x] route-groups.md
- [x] http-client-patterns.md

### 3.5 ASP.NET Minimal APIs (Alternative View)

> `docs/03-csharp/05-aspnet-web-apis/`

- [x] minimal-api-routing.md
- [x] endpoint-organization.md
- [x] parameter-binding.md
- [x] endpoint-filters.md
- [x] results.md
- [x] route-constraints.md
- [x] api-error-responses.md
- [x] mime-versioning.md
- [x] request-response-pattern.md
- [x] domain-exceptions.md
- [x] policy-services.md
- [x] value-objects.md
- [x] openapi-configuration.md

### 3.6 Authentication & Authorization

> `docs/03-csharp/05-auth/`

- [x] jwt-authentication.md
- [x] api-key-authentication.md
- [x] oauth-oidc-integration.md
- [x] claims-based-authorization.md
- [x] policy-based-authorization.md
- [x] role-based-authorization.md
- [x] resource-based-authorization.md
- [x] multi-factor-authentication.md
- [x] auth-security-best-practices.md

### 3.7 ASP.NET Core Advanced

> `docs/03-csharp/06-aspnet-advanced/`

- [x] signalr-basics.md
- [x] signalr-authentication.md
- [x] grpc-services.md
- [x] grpc-vs-rest.md
- [x] request-response-pipeline.md
- [x] httpcontext-features.md
- [x] kestrel-configuration.md
- [x] kestrel-performance.md
- [x] hosting-aspnet.md
- [x] application-lifetime.md
- [x] session-state.md
- [x] tempdata-aspnet.md
- [x] cors-aspnet-detailed.md
- [x] url-rewriting.md
- [x] response-caching-middleware.md
- [x] output-caching.md
- [x] background-services.md
- [x] health-checks.md
- [ ] websockets-aspnet.md
- [ ] http-logging.md
- [ ] w3c-logging.md
- [ ] request-decompression.md

### 3.8 Source Generation

> `docs/03-csharp/07-source-generation/` — **not yet created**

- [ ] source-generators-overview.md
- [ ] incremental-source-generators.md
- [ ] source-generation-debugging.md
- [ ] roslyn-analyzers.md
- [ ] custom-analyzers.md
- [ ] lambda-annotations-source-gen.md

---

## 4. AWS DynamoDB

> `docs/04-database/01-dynamodb/`

- [x] dynamodb-best-practices.md
- [x] single-table-design.md
- [x] data-modeling-patterns.md
- [x] dynamodb-repository-pattern.md
- [x] dynamodb-transactions.md
- [ ] dynamodb-partition-keys.md
- [ ] dynamodb-gsi-lsi.md
- [ ] dynamodb-access-patterns.md
- [ ] dynamodb-capacity-modes.md
- [ ] dynamodb-streams.md
- [ ] dynamodb-ttl.md
- [ ] dynamodb-batch-operations.md
- [ ] dynamodb-conditional-writes.md
- [ ] dynamodb-pagination.md
- [ ] dynamodb-dotnet-sdk.md
- [ ] dynamodb-local-development.md

---

## 5. AWS Infrastructure

> `docs/05-infrastructure/`

### 5.1 Terraform

> `docs/05-infrastructure/01-terraform/`

- [x] terraform-basics.md
- [x] terraform-best-practices.md
- [x] terraform-modules.md
- [ ] terraform-state-management.md
- [ ] terraform-vs-cloudformation.md

### 5.2 AWS Lambda

> `docs/05-infrastructure/02-aws-lambda/`

- [x] lambda-best-practices.md
- [x] lambda-dotnet-deployment.md
- [ ] lambda-cold-starts.md
- [ ] lambda-native-aot.md
- [ ] lambda-layers.md
- [ ] lambda-environment-variables.md
- [ ] lambda-powertools-dotnet.md

### 5.3 Serverless Patterns — **not yet created**

- [ ] api-gateway-rest-vs-http.md
- [ ] api-gateway-lambda-proxy.md
- [ ] api-gateway-custom-authorizers.md
- [ ] lambda-sqs-integration.md
- [ ] lambda-eventbridge.md
- [ ] step-functions.md

### 5.4 Core AWS Services — **not yet created**

- [ ] s3-best-practices.md
- [ ] cloudfront-distribution.md
- [ ] route53-dns.md
- [ ] acm-certificates.md
- [ ] cloudwatch-logs-metrics.md
- [ ] iam-least-privilege.md
- [ ] secrets-manager.md
- [ ] cognito-authentication.md
- [ ] ses-email-sending.md
- [ ] sns-sqs-patterns.md
- [ ] waf-security.md

---

## 6. Architecture & API Design

### 6.1 Architecture

> `docs/07-architecture/`

- [x] clean-architecture.md
- [x] cqrs-pattern.md
- [x] repository-pattern.md

### 6.2 API Design

> `docs/08-api-design/`

- [x] rest-api-principles.md
- [x] api-versioning.md
- [x] api-error-handling.md
- [x] openapi-documentation.md

---

## 7. Security

> `docs/09-security/`

- [x] authentication.md
- [x] authorization.md
- [x] owasp-security.md
- [x] secrets-management.md
- [ ] content-security-policy.md
- [ ] cors-configuration.md
- [ ] input-validation.md
- [ ] encryption-at-rest-transit.md
- [ ] dependency-vulnerability-scanning.md

---

## 8. Performance & Observability

> `docs/10-performance/`

- [x] caching-strategies.md
- [x] monitoring-observability.md
- [x] performance-optimization.md
- [ ] structured-logging.md
- [ ] distributed-tracing.md
- [ ] cloudwatch-dashboards.md
- [ ] alerting-sla-slo.md

---

## 9. CI/CD & Deployment — **not yet created**

- [ ] gitea-actions-workflows.md
- [ ] deployment-strategies.md
- [ ] blue-green-canary.md
- [ ] rollback-strategies.md
- [ ] infrastructure-as-code.md

---

## 10. Containers — **not yet created**

- [ ] dockerfile-best-practices.md
- [ ] multi-stage-builds.md
- [ ] docker-dotnet-optimization.md
- [ ] ecr-image-registry.md
- [ ] ecs-fargate.md
- [ ] ecs-vs-lambda.md

---

## Summary

| Section                | Files Exist | Still Needed |
| ---------------------- | ----------- | ------------ |
| 1. Hugo Static Sites   | 84          | 0            |
| 2. Vue.js & TypeScript | 35          | 0            |
| 3. C# & .NET Backend   | 104         | ~28          |
| 4. DynamoDB            | 5           | ~11          |
| 5. Infrastructure      | 5           | ~18          |
| 6. Architecture & API  | 7           | 0            |
| 7. Security            | 4           | ~5           |
| 8. Performance         | 3           | ~4           |
| 9. CI/CD               | 0           | ~5           |
| 10. Containers         | 0           | ~6           |
| **Total**              | **~253**    | **~87**      |

**Next step:** Condense all 253 existing files from verbose code dumps (~500-1400 lines each) down to concise cheat sheets (~30-80 lines each).
