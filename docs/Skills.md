## SKILLS

How the agent sees skills (Discovery)
At the start of a session the agent does not load every skill fully.
It only loads lightweight metadata for every available skill:
YAML---
name: pdf
description: Process, extract text, fill forms, or create PDFs. Use when the user asks about PDF files, forms, or document extraction.
---

Cost: ~50–100 tokens per skill.
This list is injected into the system prompt (or exposed via a list_skills tool).
The agent now knows “these skills exist and roughly when they are relevant.”

This is called Level 1 / progressive disclosure. It scales to dozens or hundreds of skills without blowing the context window.


2. How the agent picks the right skill (Activation / Routing)
When the user gives a task, the LLM itself acts as the router:

It looks at the user request.
It compares it against the short description of every skill it knows.
If a description matches (semantic similarity + keywords), the agent decides “this skill is relevant.”
It then loads the full body of that skill (usually by reading the SKILL.md file with a read / bash tool or a dedicated load_skill tool).

Only the matched skill(s) enter the context. Everything else stays out.
This is Level 2. The full instructions (recommended < 5k tokens) now guide the agent’s reasoning and tool use.
Key design rule: The quality of the description field is the single most important factor in whether the skill is ever selected.


3. How the agent implements / follows the skill (Execution)
Once the full SKILL.md body is in context, the agent simply treats it as authoritative procedural instructions.
Typical flow:

Follow the numbered steps / decision tree in the skill.
When the skill says “read references/forms.md” or “run the script in scripts/extract.py”, the agent uses its normal tools (read_file, bash, etc.) to pull those resources on demand (Level 3).
The skill can also declare preferred tools or constraints.
The agent continues using its normal tool-calling loop, but now guided by the skill’s playbook.

Skills are not a separate runtime or protocol. They are just high-quality, on-demand instructions + optional code/resources that the same agent already knows how to execute.