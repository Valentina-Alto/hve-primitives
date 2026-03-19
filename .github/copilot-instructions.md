---
applyTo: "**"
---

# Project Conventions

## Tech Stack

- **Frontend:** HTML, CSS, vanilla JavaScript — no frameworks, no build tools
- **Backend:** Python (Flask or similar lightweight framework)
- **Data:** Synthetic/mock data generated in Python — no database required

Keep it simple. No npm, no TypeScript, no preprocessors. The prototype should run with a single `pip install` and `python app.py`.

## README — Always Generate at Project End

Every project must end with a `README.md` that includes:

1. **Project overview** — what it does in one paragraph
2. **Getting started** — how to install and run
3. **Prototype architecture** — a Mermaid diagram showing the current prototype's components and data flow
4. **Production architecture** — a second Mermaid diagram showing the future-state, production-ready architecture (managed services, scalability, auth, observability, etc.)
