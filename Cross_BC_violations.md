# DDD / Hexagonal architecture review — `shipstampede-platform`

**Scope:** `C:\Data_files\iac\shipstampede-platform`  
**Method:** Static analysis of package imports (`@shipstampede/*`), module loaders, and `apps/backend-api` controllers/middleware.  
**Conventions referenced:** In-repo DDD layout (`packages/{domain}/core` vs `adapters`), Lookup Service / ACL pattern (consumer **core** must not depend on another BC’s repositories; cross-BC data via interfaces implemented in **adapters**), hexagonal dependency direction (primary adapters should depend on application/domain ports, not infrastructure barrels).

**Note:** Imports from `@shipstampede/core`, `@shipstampede/core-result`, `@shipstampede/logger-core`, and same-BC `*-types` / `*-messages` are treated as allowed shared technical or integration contracts unless they pull another BC’s **core** model in.

---

## Violation list

Each item: **filename**, **violation**, **path**, **fix**.

| Filename | Violation | Path | Fix |
|----------|-----------|------|-----|
| `Shipment.ts` | Shipping **domain aggregate** imports Rating BC value objects (`RateCatalogId`, `RateQuoteId`) from `@shipstampede/rating-core`, coupling bounded contexts at the model layer. | `packages/shipping/core/src/domain/shipping/aggregates/Shipment.ts` | Define shipping-owned identifiers (e.g. `Uuid` / branded types in `shipping-core` or `shipping-types`) populated from rating via use cases or anti-corruption mapping; optionally keep foreign keys as plain UUIDs in persistence only. |
| `ShipmentRepository.ts` | Shipping repository **port** exposes `RateQuoteId` from `@shipstampede/rating-core`, leaking Rating’s type into Shipping’s core boundary. | `packages/shipping/core/src/domain/shipping/repositories/ShipmentRepository.ts` | Replace with a shipping-local type (`Uuid` / `ShipmentRateQuoteRef`) and map in adapters when calling rating use cases or DB columns. |
| `CreateShipmentFromQuoteUsecase.ts` | Shipping use case imports `RateCatalogId`, `RateQuoteId` from `@shipstampede/rating-core`. | `packages/shipping/core/src/use-cases/shipping/CreateShipmentFromQuoteUsecase.ts` | Accept neutral DTO input (UUID strings) from API/adapters or depend on a **shipping**-defined lookup/query port; resolve rating IDs only in `shipping-adapters`. |
| `PurchaseLabelUsecase.unit.test.ts` | Test couples shipping tests to `@shipstampede/rating-core` IDs. | `packages/shipping/core/_tests/shipping/use-cases/PurchaseLabelUsecase.unit.test.ts` | After decoupling production types, use shipping-local test builders or plain UUIDs. |
| `VoidShipmentUsecase.unit.test.ts` | Same cross-BC test coupling to rating-core IDs. | `packages/shipping/core/_tests/shipping/use-cases/VoidShipmentUsecase.unit.test.ts` | Same as above. |
| `ListShipmentsUsecase.unit.test.ts` | Same. | `packages/shipping/core/_tests/shipping/use-cases/ListShipmentsUsecase.unit.test.ts` | Same as above. |
| `GetShipmentByIdUsecase.unit.test.ts` | Same. | `packages/shipping/core/_tests/shipping/use-cases/GetShipmentByIdUsecase.unit.test.ts` | Same as above. |
| `Shipment.unit.test.ts` | Aggregate unit test imports rating-core IDs. | `packages/shipping/core/_tests/shipping/aggregates/Shipment.unit.test.ts` | Same as above. |
| `UpdateCarrierAccountUsecase.ts` | Carriers **core** depends on Settings BC (`EncryptionService`, `SettingsIdentifiers` from `@shipstampede/settings-core`). | `packages/carriers/core/src/use-cases/UpdateCarrierAccountUsecase.ts` | Introduce a carriers-owned port (e.g. `CredentialEncryptionPort` / `SecretEncryptionService`) in `carriers-core`; implement by delegating to settings in `carriers-adapters` (lookup / ACL). |
| `GetCarrierAccountWithCredentialsByIdUsecase.ts` | Same carriers → settings-core coupling. | `packages/carriers/core/src/use-cases/GetCarrierAccountWithCredentialsByIdUsecase.ts` | Same port + adapter implementation pattern. |
| `GetActiveCarrierAccountsForProfileUsecase.ts` | Same. | `packages/carriers/core/src/use-cases/GetActiveCarrierAccountsForProfileUsecase.ts` | Same. |
| `GetActiveCarrierAccountForShippingUsecase.ts` | Same. | `packages/carriers/core/src/use-cases/GetActiveCarrierAccountForShippingUsecase.ts` | Same. |
| `DiscoverAndSyncEpsCttChannelsUsecase.ts` | Same. | `packages/carriers/core/src/use-cases/DiscoverAndSyncEpsCttChannelsUsecase.ts` | Same. |
| `CreateCarrierAccountUsecase.ts` | Same. | `packages/carriers/core/src/use-cases/CreateCarrierAccountUsecase.ts` | Same. |
| `ValidateCarrierProfileCredentialsUsecase.ts` | Same. | `packages/carriers/core/src/use-cases/ValidateCarrierProfileCredentialsUsecase.ts` | Same. |
| `ValidateCarrierCredentialsUsecase.ts` | Same. | `packages/carriers/core/src/use-cases/ValidateCarrierCredentialsUsecase.ts` | Same. |
| `UpdateCarrierProfileUsecase.ts` | Same. | `packages/carriers/core/src/use-cases/UpdateCarrierProfileUsecase.ts` | Same. |
| `CreateCarrierProfileUsecase.ts` | Same. | `packages/carriers/core/src/use-cases/CreateCarrierProfileUsecase.ts` | Same. |
| `EncryptedCarrierCredentials.ts` | Carriers **value object** depends on `EncryptionService` type from `@shipstampede/settings-core`. | `packages/carriers/core/src/domain/value-objects/EncryptedCarrierCredentials.ts` | Move encryption behind a carriers-core interface; inject implementation from adapters. |
| `UpdateContactInfoUsecase.ts` | Settings Account **core** imports `Role` from `@shipstampede/iac-core` (identity BC leaking into settings domain). | `packages/settings/account/core/src/use-cases/UpdateContactInfoUsecase.ts` | Use a settings-local role enum/string, map from HTTP/session in adapters, or depend on a minimal `AuthorizationContext` port defined in settings-core. |
| `UpdateCompanyProfileUsecase.ts` | Same settings-account → iac-core `Role` import. | `packages/settings/account/core/src/use-cases/UpdateCompanyProfileUsecase.ts` | Same as above. |
| `AccountPricingPoliciesController.ts` | Settings HTTP controller injects **`AccountRepository`** and **`IacIdentifiers`** from `@shipstampede/iac-core` alongside rating use cases — primary adapter reaches into IAC persistence abstraction. | `apps/backend-api/src/modules/settings/pricing-policies/controllers/AccountPricingPoliciesController.ts` | Move “resolve account in tenant hierarchy” into a **settings-account** or **iac** use case (or a small ACL service) and inject that port instead of `AccountRepository`. |
| `BasePricingPoliciesController.ts` | Settings module base controller imports many types from `@shipstampede/rating-core` (application layer of another BC). | `apps/backend-api/src/modules/settings/pricing-policies/controllers/BasePricingPoliciesController.ts` | Thin controllers: add façade use cases in `settings-account-core` (or `rating` façade in settings adapters) that wrap rating operations behind settings-owned DTOs. |
| `PricingPoliciesController.ts` | Settings controller imports `@shipstampede/rating-core`. | `apps/backend-api/src/modules/settings/pricing-policies/controllers/PricingPoliciesController.ts` | Same façade / application service approach as above. |
| `PricingPolicyTemplatesController.ts` | Settings controller imports `@shipstampede/rating-core`. | `apps/backend-api/src/modules/settings/pricing-policy-templates/controllers/PricingPolicyTemplatesController.ts` | Same. |
| `pricingPolicyDtoMappers.ts` | Settings-side mapper depends on `@shipstampede/rating-core` types. | `apps/backend-api/src/modules/settings/pricing-policies/controllers/pricingPolicyDtoMappers.ts` | Map from rating **DTOs** or a neutral read model; avoid importing rating aggregate/repository symbols in the API layer if possible. |
| `PaymentIntentsController.ts` | Payments controller imports `MeUserUsecase` from `@shipstampede/iac-core` (cross-BC orchestration in HTTP layer). | `apps/backend-api/src/modules/payments/controllers/PaymentIntentsController.ts` | Expose identity on the request context only, or add a payments use case that accepts `UserIdentity` without referencing IAC use case types. |
| `PaymentCardsController.ts` | Same payments → iac-core `MeUserUsecase` coupling. | `apps/backend-api/src/modules/payments/controllers/PaymentCardsController.ts` | Same as above. |
| `InvitePartnerAccountController.ts` | Controller imports `InvitePartnerAccountUsecase` and `IacErrors` from **`@shipstampede/iac-adapters`** (infrastructure barrel) instead of `@shipstampede/iac-core`. | `apps/backend-api/src/modules/iac/controllers/InvitePartnerAccountController.ts` | Import use case and domain errors from `iac-core`; reserve `iac-adapters` for composition roots / module loaders. |
| `PartnerSignupController.ts` | Imports `PartnerSignupUsecase` from `@shipstampede/iac-adapters`. | `apps/backend-api/src/modules/iac/controllers/PartnerSignupController.ts` | Import from `iac-core`. |
| `MeUserController.ts` | Imports `MeUserUsecase` from `@shipstampede/iac-adapters`. | `apps/backend-api/src/modules/iac/controllers/MeUserController.ts` | Import from `iac-core`. |
| `PasswordAuthController.ts` | Imports auth use cases/types from `@shipstampede/iac-adapters`. | `apps/backend-api/src/modules/iac/controllers/PasswordAuthController.ts` | Import application API from `iac-core`; keep adapters for wiring only. |
| `UserSignoutController.ts` | Imports `UserSignOutUsecase` from `@shipstampede/iac-adapters`. | `apps/backend-api/src/modules/iac/controllers/UserSignoutController.ts` | Import from `iac-core`. |
| `jwt-auth.ts` | Middleware imports from `@shipstampede/iac-adapters` (driven side package). | `apps/backend-api/src/middlewares/jwt-auth.ts` | Depend on `iac-core` ports (token validation use case) or a dedicated `auth` façade package; wire implementation in composition root. |
| `PricingPolicyVersion.rate-adjustment-pipeline.test.ts` | **Rating** unit/integration-style test imports from `@shipstampede/settings-account-core`, coupling rating tests to settings account domain. | `packages/rating/core/_tests/unit/domain/aggregates/PricingPolicyVersion.rate-adjustment-pipeline.test.ts` | Test through rating’s public model/fixtures only; stub settings inputs as plain data or move shared examples to a neutral test fixture package. |
| `products/adapters/src/index.ts` | Adapters package **re-exports entire** `@shipstampede/products-core` (`export * from ...`), blurring hexagonal “adapter vs domain” package boundary. | `packages/products/adapters/src/index.ts` | Export only module loader, infra types, and mappers; require consumers to import domain from `products-core` explicitly. |

---

## Not counted as violations (context)

| Area | Rationale |
|------|-----------|
| `*-adapters` importing multiple `*-core` packages (e.g. `ShippingModule`, `RatingModule`, `BatchOperationsModule`) | Composition root wiring lookup services and use cases is expected in hexagonal architecture. |
| `settings/*/core` → `@shipstampede/settings-core` | Treated as **internal modularization** within a larger Settings bounded context (shared DI identifiers / cross-submodule services). Stricter “one package = one BC” teams might still split this further. |
| Controllers importing their **own** BC’s `*-core` use cases | Common ShipStampede pattern; acceptable if controllers stay thin and do not reach into repositories of another BC. |
| `orders-core` / `products-core` → `@shipstampede/iac-types` (e.g. `PaginationParams`) | Shared published language / technical type; acceptable trade-off, or could migrate to `@shipstampede/core` / shared `types` if you want zero reference to the IAC name from non-IAC cores. |

---

## Summary counts

| Category | Approx. distinct files |
|----------|------------------------|
| Shipping core ↔ Rating core (model + repository + use case + tests) | 9 |
| Carriers core ↔ Settings core (use cases + value object) | 9 |
| Settings account core ↔ IAC core | 2 |
| Settings API controllers ↔ Rating / IAC | 6 |
| Payments controllers ↔ IAC | 2 |
| IAC controllers / middleware ↔ `iac-adapters` barrel | 6 |
| Products adapters public surface | 1 |
| Rating tests ↔ Settings account core | 1 |

---

## Suggested remediation order

1. **Shipment / rating ID decoupling** — highest structural impact; removes aggregate-level BC merge.  
2. **Carriers encryption port** — straightforward ACL; aligns with existing lookup-service documentation.  
3. **AccountPricingPoliciesController** — remove direct `AccountRepository` usage from settings HTTP layer.  
4. **Normalize `iac-adapters` imports in controllers/middleware** to `iac-core` for clarity and dependency direction.  
5. **Settings pricing controllers** — introduce façade use cases to shrink the rating surface area at the HTTP edge.  
6. **Test and package export cleanups** (rating test coupling, `products-adapters` barrel).

---

*Generated from repository state on 2026-04-23. Re-run after refactors to validate import graphs (e.g. `pnpm exec rg "from '@shipstampede/.*-core" packages/*/core`).*

============================================================

The report is saved at **`C:\Data_files\iac\analysis_docs\Cross_BC_violations.md`**.

### What it contains

1. **Scope and method** — How imports were judged against DDD + hexagonal rules and your in-repo lookup/ACL guidance.

2. **Main violation table** — Rows with **filename**, **violation**, **path**, **fix**. Highlights:
   - **Shipping ↔ Rating:** `Shipment` and `ShipmentRepository` use `RateCatalogId` / `RateQuoteId` from `@shipstampede/rating-core` (strong BC coupling in the domain).
   - **Carriers ↔ Settings:** `carriers-core` use cases and `EncryptedCarrierCredentials` depend on `@shipstampede/settings-core` instead of a carriers-owned encryption port.
   - **Settings account ↔ IAC:** `Role` from `@shipstampede/iac-core` in two account use cases.
   - **Controllers:** `AccountPricingPoliciesController` injects `AccountRepository` / `IacIdentifiers`; several settings pricing controllers pull in `rating-core`; payments controllers use `MeUserUsecase` from `iac-core`; several IAC controllers and `jwt-auth` import **`iac-adapters`** instead of **`iac-core`**.
   - **Tests / packaging:** rating test importing `settings-account-core`; `products-adapters` re-exporting all of `products-core`.

3. **Exclusions** — Why `*-adapters` composing multiple cores, same-BC controller → core, and `settings-*` → `settings-core` are documented as acceptable or contextual.

4. **Summary counts and remediation order** — Prioritized cleanup sequence.

If you want the body switched from a table to a strict repeating block format (e.g. repeated `###` sections per file), say so and it can be reformatted in place.


================================================================

analyze and generate a report for this folder "C:\Data_files\iac\shipstampede-platform" if there is a violation of DDD and Hexagonal architecture like Cross BC calling or each packages and controller. How the report would go is use list of the violations - filename, violation, path, fix, then the list would go on. Put the generated report on this file "C:\Data_files\iac\analysis_docs\Cross_BC_violations.md"

