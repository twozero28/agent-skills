# API Contracts

## Transport boundary

- Keep controllers, resolvers, transport DTOs, and response mapping in the owning feature's `api` directory.
- Let an endpoint call the application Service, QueryService, or UseCase that owns its operation. Do not access repositories or coordinate independent business services in the endpoint.
- Convert transport input into an application command/query or a matching domain-owned `Draft/Patch`. Service and UseCase signatures must not import API `Request/Input` classes.
- Pass the trusted actor and tenant context separately from caller-controlled payloads. Reject or ignore attempted ownership changes according to the API contract.
- Validate transport shape at entry; enforce business invariants in the domain and application authorization at the operation boundary, including calls from jobs and consumers.
- Preserve the installed HTTP adapter, validation library, API response format, and error conventions. Do not introduce a replacement stack for a small change.

## Runtime validation

- Use the project's established runtime validator; TypeScript annotations and assertions do not validate external input.
- With `ValidationPipe` and class-validator, use concrete DTO classes with validation metadata. Import these classes as runtime values; `import type`, interfaces, and generic types cannot supply that metadata.
- Choose `transform`, unknown-property handling, and error formatting deliberately. `whitelist` strips properties without validation metadata; `forbidNonWhitelisted` rejects them. Neither option authorizes otherwise valid fields.
- Validate route and query parameters explicitly. Bound page sizes and bulk input sizes; parse booleans and numbers according to the transport contract instead of relying on truthiness or broad coercion.
- Validate nested objects and array elements using the adopted schema or explicit class-validator/class-transformer metadata. A `CreateItemRequest[]` parameter alone does not describe its element validator.
- Use the appropriate mapped-type helpers: `@nestjs/graphql` for GraphQL inputs, `@nestjs/swagger` for Swagger DTOs, or `@nestjs/mapped-types` when neither integration applies.
- Keep validation and transformation configuration shared between production bootstrap and tests that claim to verify it.

See [Nest validation](https://docs.nestjs.com/techniques/validation) and [GraphQL mapped types](https://docs.nestjs.com/graphql/mapped-types).

## Partial updates

Define each field's update contract before deriving a patch type:

| Input state | Required behavior |
|---|---|
| Omitted | Leave the existing value unchanged |
| Supplied value | Validate, authorize, and apply the value |
| Explicit `null` | Clear the value only when the contract allows it; otherwise reject |

- Preserve these states through DTO conversion, commands, domain methods, and persistence mapping. Do not use truthiness or `value ?? oldValue` when it loses a valid update or explicit clearing request.
- Do not assume `Partial<T>` or `PartialType()` defines the business patch contract. Required identity, version, or operation fields may remain required.
- With class-validator, `@IsOptional()` skips both `undefined` and `null`. Default `PartialType()` behavior also permits skipping null validation. For non-nullable updates, use the installed helper's supported null option or explicit conditional validation.
- Inspect inherited field initializers and GraphQL default values. They must not turn omitted patch fields into writes. Use supported options to omit inherited defaults or define the patch explicitly.
- Allow only mutable fields. Do not derive update inputs mechanically from an entity containing IDs, ownership, authorization, audit, or internal state fields.

## Guards, pipes, interceptors, and filters

| Component | Responsibility |
|---|---|
| Middleware | Request setup and transport concerns needed before handler selection |
| Guard | Authentication and entry-point access decisions |
| Pipe | Runtime input validation and conversion |
| Interceptor | Cross-cutting execution and response behavior |
| Exception filter | Mapping uncaught failures into the transport error contract |

- Remember that guards run before pipes. Validate any raw request data a guard consumes; do not assume handler DTO transformation already ran.
- Keep object-level ownership and tenant checks in application contracts. A valid token, permitted route, or whitelisted DTO does not authorize access to a particular object.
- Register global enhancers that need injected dependencies through the supported module provider mechanism, such as `APP_GUARD` or `APP_FILTER`.
- Keep domain errors independent of HTTP and GraphQL objects. Map known failures at the appropriate adapter; retain unexpected failures and their causes for diagnostics without exposing internals to clients.
- Do not swallow a failure merely to return a successful response. Caught exceptions do not reach exception filters; filters do not form a chain that reprocesses the same handled exception.
- Check each transport's actual execution path. HTTP setup does not establish GraphQL field, WebSocket, queue, or hybrid microservice behavior automatically.

See [Nest request lifecycle](https://docs.nestjs.com/faq/request-lifecycle) and [GraphQL execution features](https://docs.nestjs.com/graphql/other-features).

## Responses and compatibility

- Return the existing public response contract. Avoid exposing persistence records, ORM wrappers, or every enumerable property by accident.
- Use explicit output mapping or the project's verified serializer. A return-type annotation alone does not remove sensitive properties at runtime.
- When using `ClassSerializerInterceptor`, supply the expected class instances or supported explicit conversion options; verify nested and plain-object responses.
- For MikroORM entities, follow [MikroORM](mikro-orm.md) and its serialization rules. Do not assume class-transformer understands ORM relation wrappers.
- Define identifier, date/time, decimal, large integer, optional-field, and null serialization consistently. Preserve existing wire representations when changing internal types.
- Keep REST serialization and GraphQL schema/field resolution separate. Apply [GraphQL](graphql.md) when the endpoint uses GraphQL.

See [Nest serialization](https://docs.nestjs.com/techniques/serialization).

## Verification

- Exercise invalid raw requests, unknown or forbidden fields, malformed route/query values, nested inputs, and oversized pages where relevant.
- Check omitted, explicit-null, and supplied patch values independently, including inherited defaults.
- Verify authorization through the real guard and application path when claiming security coverage.
- Inspect the actual serialized response and error shape, including sensitive-field exclusion and installed adapter behavior.
