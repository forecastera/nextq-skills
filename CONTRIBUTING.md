# Adding and changing skills

## Add a skill to an existing domain

1. Create `plugins/<plugin>/skills/<skill-name>/SKILL.md`. The folder name and the `name` field must be identical.
2. Use **only** these frontmatter keys. claude.ai and the Skills API reject a skill with any other top-level key:

   ```yaml
   ---
   name: nextq-example-skill          # lowercase, digits, hyphens; ≤ 64 chars; starts with nextq-
   description: >-                    # ≤ 1024 chars: what it does + when to load it + trigger phrases
     ...
   metadata:                          # anything else goes here, as quoted strings
     author: "Next Quarter"
     suite: "..."
     version: "1.0.0"
   ---
   ```

3. Bump `version` in `plugins/<plugin>/.claude-plugin/plugin.json`. Published copies update only when it changes.
4. Add the skill to the table in the plugin's `README.md`.
5. Validate:

   ```
   claude plugin validate plugins/<plugin>
   claude plugin validate .
   ```

## Add a new domain (for example FP&A)

1. Copy the layout of `plugins/nextq-demand-planning/` to `plugins/nextq-<domain>/`, including `.claude-plugin/plugin.json` (with `"license": "Apache-2.0"`) and `README.md` (at least 40 words, stating what the plugin runs and sends). The root `LICENSE` covers all plugins.
2. Add an entry to `.claude-plugin/marketplace.json`.
3. **Choose the plugin name carefully.** After it is listed, the name is a permanent ID. `displayName` can change later.
4. A new domain needs **one** new submission in the Claude directory portal. New skills in an existing domain do not.

## Rules

- **No secrets, credentials, customer data or real PII** anywhere, including examples. Use synthetic data.
- Names must not start with `claude-` or `anthropic-`, and must not call the plugin "official".
- Only text files in plugin folders. No symlinks, binaries, `.DS_Store` or `Thumbs.db`.
- Anthropic's security scanner reads skill text like shell code. In formulas, write multi-letter names as `$\mathit{SS}=...$`, not `$SS=...$`, and don't use bare shell words such as `export`, `env` or `printenv` as labels. Otherwise the scan puts the plugin on hold. `claude plugin validate` does not run this check; only the portal does.
- A skill must work on its own. Inline the rules it relies on and do not point at files outside its folder.
