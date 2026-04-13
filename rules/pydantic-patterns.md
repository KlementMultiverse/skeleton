---
description: Pydantic v2 rules. Loads for all Python projects.
paths: ["*.py", "app/**/*.py", "schemas/**/*.py"]
---

# Pydantic v2 Rules

## Field Definitions

1. Use `Field(min_length=1)` on all required string fields — empty string is not valid
2. Use `Field(ge=0)` and `Field(le=N)` for numeric range constraints
3. Use `str | None = None` for optional fields — NEVER `Optional[str]`
4. Use `Annotated[str, Field(min_length=1, max_length=255)]` to attach constraints without losing type info

## Model Config

5. Use `ConfigDict(from_attributes=True)` on ALL response models wrapping SQLAlchemy ORM objects
6. Use `ConfigDict(strict=True)` on models validating LLM output — disables coercion, "1" is not 1
7. Use `ConfigDict(populate_by_name=True)` when using field aliases

## PATCH Handling

8. Use `model_dump(exclude_unset=True)` for ALL PATCH endpoint processing
   — only includes fields explicitly sent, leaves rest untouched
9. NEVER use `model_dump()` on PATCH models without `exclude_unset=True`
   — every unset optional field becomes null, wiping real data

## Validators

10. Use `@field_validator('field', mode='before')` for single-field pre-processing — requires `@classmethod`
    ```python
    @field_validator('email', mode='before')
    @classmethod                          # ← required, easy to forget
    def normalize_email(cls, v: str) -> str:
        return v.strip().lower()
    ```
11. Use `@model_validator(mode='after')` for cross-field validation (e.g. passwords match, end > start)
12. Use `@computed_field` for properties that appear in `model_dump()` without being input fields

## LLM Output Validation

13. Use `model_validate_json()` to validate LLM output before saving to database
14. Always wrap `model_validate_json()` in `try/except ValidationError` — LLM output is unreliable
15. Use `ConfigDict(strict=True)` on models validating LLM output
16. On `ValidationError`: retry with error details appended to the next prompt
    ```python
    try:
        result = MyModel.model_validate_json(llm_output)
    except ValidationError as e:
        # append e.json() to next prompt and retry
    ```

## LLM Tool Definitions

17. Use `model_json_schema()` to generate tool input schemas for Claude
    — same format Anthropic SDK expects for tool definitions

## Method Names (v2 only)

18. NEVER use `.dict()` — use `model_dump()`
19. NEVER use `.json()` — use `model_dump_json()`
20. NEVER use `parse_obj()` — use `model_validate()`
21. NEVER use `parse_raw()` — use `model_validate_json()`
22. `model_validate(dict)` — takes a Python dict as input
    `model_validate_json(str)` — takes a raw JSON string as input — use for LLM output
23. Use `model_dump(mode='json')` when you need JSON-compatible Python types (datetime → ISO string, UUID → str)
    — different from `model_dump_json()` which returns a string; this returns a dict safe for JSON serialization

## Immutable Updates

24. Use `model_copy(update={"field": new_value})` to create a modified copy without mutating the original
    — preferred over constructing a new model from scratch when most fields stay the same

## RootModel (list or scalar as top-level response)

25. Use `RootModel` when the response IS the list — not wrapped in an object
    ```python
    from pydantic import RootModel

    class NoteIds(RootModel[list[int]]):
        pass
    # serializes as [1, 2, 3] not {"root": [1, 2, 3]}
    ```
    — use when the API returns a plain array, not `{"items": [...]}`

## Discriminated Unions (LLM tool responses)

26. Use `Annotated[Union[TypeA, TypeB], Field(discriminator='type')]` for polymorphic LLM tool responses
    — routes validation by discriminator field value, much faster than trying each type

## Schema Separation (mass assignment prevention)

27. NEVER use the same Pydantic model for both input (request body) and output (response) — they have different trust levels
    - `NoteCreate` — input: only user-supplied fields, no `id`, `user_id`, `created_at`
    - `NoteResponse` — output: includes `id`, `created_at`, excludes internal fields
    - `NoteUpdate` — PATCH input: all fields optional, no protected fields
28. NEVER pass `request.body` or `model_dump()` directly to ORM constructor — always map explicitly

## See Also
- `database-patterns.md` — how to use `model_dump(exclude_unset=True)` with setattr loop in PATCH routes
- `auth-patterns.md` — password validation with `@model_validator`
- `security.md` — mass assignment prevention (separate input/output schemas)
