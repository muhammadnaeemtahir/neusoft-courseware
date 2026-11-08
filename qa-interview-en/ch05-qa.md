# Chapter 5 Q&A: Configuration & Rules

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch05-qa.md](../qa-en/ch05-qa.md).

## 5.1 Config schema and defaults

**Q:** With TypeScript types in place, why is runtime validation still needed? What is the scariest thing without validation?

**A:** Type erasure — TS types exist only at compile time; config files are JSON read in at runtime, beyond the compiler's reach. The scariest thing without validation isn't an error but **silently not taking effect**: the user thinks the config went through while the program is actually running on defaults. Schema validation is like airport security: oversized items are stopped on the spot, and you're told which piece broke which rule (error messages carrying a dotted path). Defaults solve "usable with zero configuration"; `defaultConfig()` is itself executable documentation.

## 5.2 Config layering

**Q:** What is the order of the five layers? On what grounds is this order "right"?

**A:** `default < global < project < env < cli`, with rightward layers overriding leftward. The judging standard: **the more specific the scope and the shorter the lifetime, the higher the priority** — default is the program author's general default, global is "my machine's preferences", project is "this team's conventions" (travels with the repo, reviewable), env is "this run's environment", cli is "what this one command explicitly specified". The override order must be deterministic and explainable, or users can't predict the result.

## 5.3 Deep merge and conflict diagnosis

**Q:** What is the teaching edition's merge semantics? Why don't arrays merge by index?

**A:** Objects merge recursively (deepMerge); **arrays and scalars are replaced wholesale** — deliberately choosing the most predictable semantics. The counterexample for index-based merging: the defaults have three rules, the project layer writes one; after index merging, position 0 is swapped out while the defaults' other two remain — the user can neither predict which survive nor "append one" without counting how many elements the default list has. Wholesale replacement demands writing the full list, but the result can be derived line by line from the input — explicit beats clever.

## 5.4 Project rule discovery

**Q:** Which three problems does rule discovery solve? Are rules commands or material?

**A:** Where to look (conventional locations plus config customization, avoiding scanning the whole repo), in what order (ordering must be deterministic, otherwise not reproducible across machines), and scope (root-directory rules take global effect; deep ones like `packages/foo/AGENTS.md` are only more relevant for their subtree). Rules are **prompt material**, not commands: context is a scarce resource, and injecting full text unconditionally crowds out the task budget, so there must be a length budget and a degradation path.

## 5.5 Consumers of the four config sections

**Q:** Where does the config file's value lie? Why doesn't Tool get its own config section?

**A:** The config itself is only declarative data; its value lies in **being consumed by the subsystems**: provider.* decides which model service to connect to, agent.* decides runtime parameters, permissions.* decides the permission policy, rules decides context injection. Which tools exist is statically decided by the registry, and per-tool "configuration" is just that tool's three-state mode in permissions — each consuming module reads only the subtree it cares about, so adding a module doesn't change the config structure.

## 5.6 Secret management

**Q:** Where do the secret itself and the variable name each live? Does the program crash when the key isn't injected?

**A:** Iron rule: **secrets never go into config** — a config file's destiny is to be shared (into Git, pasted into issues). Config stores only the environment variable **name** (`apiKeyEnv: "OPENCODE_LAB_API_KEY"`); the body lives in the process environment, and at runtime `env[config.provider.apiKeyEnv]` fetches the real value and hands it to the provider to place in the request headers — share the config freely, zero leak surface. Without injection, undefined is read and the provider selection chain falls back to the Mock Provider — the program doesn't crash, but what runs alongside you is the demo script, not a real model.

## 5.7 Hot reload

**Q:** Why does the teaching edition deliberately skip hot reload? What road has been paved for future hot reload?

**A:** The difficulty isn't "reading the file" but consistency: what if the policy changes while the agent is executing a tool? Whose fault is half-new-half-old? The real solution is "swap the whole document at once; running tasks finish on the old config, then swap". Adding version numbers too early, before the schema has stabilized, is a pseudo-requirement. The paved road: `loadConfig` is a **pure function with explicit inputs** (workspace/globalPath/env/cliOverrides all injected); a watcher only needs to call it again and replace the reference wholesale.

## 5.8 Prompt templates

**Q:** Which three properties must system prompt injection satisfy? Can rules force behavior?

**A:** (1) Determinism (same project, same config, the assembled result is identical every time, otherwise model behavior isn't reproducible); (2) traceability (each section is labeled with its source, so "system instructions" and "project conventions" can be told apart); (3) budgeted (dynamic sections have length caps and degradation strategies). Rules **cannot force** behavior — they are just prompt text, and the model may not comply. To make something necessarily not happen, you need the chapter 6 permission layer's hard interception, not "writing 'no' in AGENTS.md".
