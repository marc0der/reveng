---
name: validate-mermaid
description: Validates all Mermaid diagram blocks in a markdown file and fixes broken diagrams in place. Uses the local mermaid-cli (mmdc) when available, falling back to the Mermaid Chart MCP tool.
allowed-tools: Read, Edit, Bash(mmdc*), Bash(command -v*), Bash(mktemp*), Bash(rm*), mcp__claude_ai_Mermaid_Chart__validate_and_render_mermaid_diagram
---

You validate every Mermaid diagram block in a markdown file. For each broken diagram you attempt to fix it in place, retrying up to 2 times.

## Input

The markdown file path is: `$ARGUMENTS`

## Choosing a validator

Two validators are possible. Pick one **before** doing anything else, in this order:

1. **`mmdc` (local mermaid-cli)** — check with `command -v mmdc`. Prefer this whenever it is present. It works headlessly and needs no authentication, so it is the validator that works on reveng's primary path (`reveng synth` running non-interactively with an `ANTHROPIC_API_KEY`).
2. **Mermaid Chart MCP** — use only if `mmdc` is absent and `mcp__claude_ai_Mermaid_Chart__validate_and_render_mermaid_diagram` is present in the available tool list. This connector is hosted on claude.ai and is reachable only in sessions authenticated through a claude.ai account; it is unavailable when running on an API key.

If **neither** is available, print the following message verbatim and stop with a success exit (do not fail the caller's workflow):

```
⚠ validate-mermaid skipped: no Mermaid validator available.

This skill needs one of:
  • mermaid-cli — install with: npm install -g @mermaid-js/mermaid-cli
    (works headlessly; the reveng sandbox image ships it preinstalled)
  • the "Mermaid Chart" connector on claude.ai — Settings → Connectors,
    then restart Claude Code (claude.ai sessions only, not API keys)

Skipping validation — your other outputs are unaffected.
```

State which validator you are using in your final report.

## Steps

1. **Choose the validator** as described above.

2. **Read the file** using the Read tool.

3. **Extract all fenced Mermaid blocks.** Identify every occurrence of a ` ```mermaid ` code fence and its closing ` ``` `. Note the line numbers and content of each block. If no Mermaid blocks are found, report "No Mermaid diagrams found in [file path]" and stop.

4. **Validate each block.**

   **Using `mmdc`:** write the block's content (excluding the fence lines) to a temporary `.mmd` file and render it:

   ```
   mmdc -i /tmp/block-N.mmd -o /tmp/block-N.svg
   ```

   **`mmdc` exits 0 even when a diagram fails to parse**, so never judge the result by exit code. A block is valid only if **both**:
   - the output `.svg` file exists and is non-empty, **and**
   - the combined output contains no `Parse error`, `Expecting`, `Syntax error`, or `error` line

   Otherwise treat it as broken, and take the error text from the output for use in step 5. Clean up the temporary files when done.

   **Using the MCP tool:** call `mcp__claude_ai_Mermaid_Chart__validate_and_render_mermaid_diagram` with:
   - `mermaidCode`: the content between the fences (excluding the fence lines themselves)
   - `prompt`: "Validate this Mermaid diagram"
   - `diagramType`: inferred from the first line of the block (e.g. `flowchart`, `sequenceDiagram`, `gantt`, `classDiagram`)
   - `clientName`: "claude"

   If validation fails mid-run for an environmental reason (a transient MCP error, or `mmdc` crashing rather than reporting a parse error), report it and continue to the next block — do not fail the caller's workflow.

5. **Fix broken diagrams.** For each block that fails validation:
   - Read the error details from the validator output
   - Determine the fix based on the error message
   - Use the Edit tool to replace the broken mermaid content with the corrected version (use the full block content as `old_string` and the fixed content as `new_string`)
   - Re-validate using the same validator
   - If it still fails, attempt one more fix (maximum 2 retries per block)
   - If a block remains broken after 2 retries, mark it as unfixable and continue to the next block

6. **Report results.** Return a summary containing:
   - Which validator was used (`mmdc` or Mermaid Chart MCP)
   - Total number of Mermaid blocks found
   - Number that passed validation on first attempt
   - Number that were fixed (with a brief note of what was wrong)
   - Number that remain broken after retries (with the error details)
