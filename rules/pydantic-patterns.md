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

## SecretStr — Sensitive Fields

29. Use `SecretStr` for passwords, API keys, tokens, and any field that must not appear in logs or repr:
    ```python
    from pydantic import SecretStr

    class Settings(BaseModel):
        database_url: SecretStr
        anthropic_api_key: SecretStr

    # repr shows SecretStr('**********') — key never leaks to logs
    # Access actual value only when needed: settings.anthropic_api_key.get_secret_value()
    ```
30. NEVER use `str` for credentials in settings models — a plain `str` will be logged in full on any debug print, exception traceback, or Sentry breadcrumb

## Mutable Defaults

31. NEVER use a mutable literal as a field default — Pydantic shares the object across instances:
    ```python
    # BAD — all instances share the same list
    class Note(BaseModel):
        tags: list[str] = []

    # GOOD
    class Note(BaseModel):
        tags: list[str] = Field(default_factory=list)
    ```

## model_rebuild() — Forward References

32. Call `Model.model_rebuild()` after defining all models when you have forward references (circular or mutually-referencing models):
    ```python
    class User(BaseModel):
        posts: list["Post"] = []

    class Post(BaseModel):
        author: User

    User.model_rebuild()  # resolves the "Post" forward reference
    ```
33. FastAPI calls `model_rebuild()` automatically for response models — only call it manually in non-FastAPI code or when you define models in separate modules

## AliasGenerator — API Naming Conventions

34. Use `AliasGenerator` to auto-convert snake_case Python fields to camelCase JSON without decorating every field:
    ```python
    from pydantic import ConfigDict, AliasGenerator
    from pydantic.alias_generators import to_camel

    class NoteResponse(BaseModel):
        model_config = ConfigDict(
            alias_generator=AliasGenerator(serialization_alias=to_camel),
            populate_by_name=True,
        )
        created_at: datetime   # serializes as "createdAt" in JSON output
        is_pinned: bool        # serializes as "isPinned"
    ```
35. Use `validation_alias` + `serialization_alias` separately when input and output naming conventions differ (e.g., external API sends `snake_case` but your client expects `camelCase`)
