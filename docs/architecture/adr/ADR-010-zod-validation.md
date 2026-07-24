# ADR-010: Zod for DTO Validation

**Status:** Accepted  
**Date:** 2025-07-23  
**Deciders:** Architecture Team

---

## Context

Every HTTP request entering GLU's API must be validated before it reaches business logic. Validation must be: type-safe at compile time, runtime-safe with clear error messages, and DRY (schema is the type definition, not a separate class + decorators).

---

## Decision

**Use Zod for all DTO validation via a global `ZodValidationPipe`.**

---

## Alternatives Considered

| Solution | Description |
|---|---|
| **class-validator + class-transformer** | NestJS default, decorator-based, class instances |
| **Zod** | TypeScript-first schema library, inferred types, no decorators |
| **Joi** | Mature validation library, JavaScript-origin, verbose |
| **Yup** | Similar to Joi, React ecosystem common |
| **Valibot** | Ultra-lightweight alternative to Zod |

---

## Reasons for Zod

### 1. Schema IS the type — no duplication

```typescript
// class-validator requires a class AND decorators AND a separate type:
class CreateBookDto {
  @IsString() @MinLength(1) @MaxLength(200)
  title: string;

  @IsOptional() @IsString()
  subtitle?: string;
}

// Zod: one declaration. The type is inferred from the schema.
const CreateBookSchema = z.object({
  title: z.string().min(1).max(200),
  subtitle: z.string().max(300).optional(),
});
type CreateBookDto = z.infer<typeof CreateBookSchema>;
```

With class-validator, the type and its validation rules are separate declarations that can drift. With Zod, they are one.

### 2. Composable schemas

Zod schemas compose with `.merge()`, `.extend()`, `.pick()`, `.omit()`. UpdateBookDto can be defined as `CreateBookSchema.partial()`. No copy-pasting decorators.

### 3. Better error messages

Zod validation errors are structured objects with path + message. They map cleanly to RFC 7807 Problem Details format for API error responses.

### 4. Works on both frontend and backend

The same Zod schema used to validate server-side DTOs can be exported to the frontend (React) for form validation. Single source of truth for validation rules across the entire stack.

### 5. No reflect-metadata dependency

class-validator requires `reflect-metadata` and `emitDecoratorMetadata: true`. Zod has no such dependency.

---

## Pros

- Schema is the type (DRY, no drift)
- Composable schemas (`.partial()`, `.extend()`, `.pick()`)
- Structured error output
- Shared frontend/backend validation
- No decorator magic, no `reflect-metadata`
- Excellent TypeScript inference

## Cons

- Not the NestJS default (requires custom `ZodValidationPipe`)
- Some NestJS ecosystem tooling (Swagger) expects `class-validator` decorators — mitigated by `nestjs-zod` package
- Learning curve for developers who know class-validator

---

## Consequences

- All request DTOs are Zod schemas (`z.object(...)`)
- `ZodValidationPipe` registered globally in `main.ts`
- DTO types are inferred (`z.infer<typeof Schema>`)
- Validation error responses follow RFC 7807 format
- `nestjs-zod` used for Swagger schema generation from Zod schemas
- Response DTOs remain as typed interfaces (no validation needed on output)
