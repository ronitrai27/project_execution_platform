agent/
├── .env
├── pyproject.toml
├── uv.lock
└── src/
    └── app/
        ├── config/
        │   └── __init__.py
        ├── api/
        │   ├── __init__.py
        │   ├── schemas/
        │   │   └── __init__.py
        │   └── v1/
        │       ├── __init__.py
        │       └── endpoints/
        │           └── __init__.py
        ├── core/
        │   ├── __init__.py
        │   ├── guardrails/
        │   │   └── __init__.py
        │   ├── memory/
        │   │   └── __init__.py
        │   ├── vectorstore/
        │   │   └── __init__.py
        │   └── utils/
        │       └── __init__.py
        ├── agents/
        │   ├── __init__.py
        │   ├── nodes/
        │   │   └── __init__.py
        │   └── tools/
        │       └── __init__.py
        └── workflows/
            ├── __init__.py
            ├── activities/
            │   └── __init__.py
            └── workflows/
                └── __init__.py
