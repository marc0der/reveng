---
name: validate-mermaid
description: Validates all Mermaid diagram blocks in a markdown file using the Mermaid Chart MCP tool and fixes broken diagrams in place.
allowed-tools: Read, Edit, mcp__claude_ai_Mermaid_Chart__validate_and_render_mermaid_diagram
---

You validate every Mermaid diagram block in a markdown file. For each broken diagram you attempt to fix it in place, retrying up to 2 times.

## Input

The markdown file path is: `$ARGUMENTS`

## Steps

1. **Check that the Mermaid MCP is available.** Before doing anything else, confirm the `mcp__claude_ai_Mermaid_Chart__validate_and_render_mermaid_diagram` tool is present in the available tool list. If it is not available, print the following message verbatim and stop with a success exit (do not fail the caller's workflow):

   ```
   ⚠ validate-mermaid skipped: the "Mermaid Chart" connector is not enabled.

   This skill depends on the Mermaid Chart MCP hosted on claude.ai. To enable it:
     1. Sign in at https://claude.ai
     2. Open Settings → Connectors
     3. Enable "Mermaid Chart"
     4. Restart Claude Code

   Skipping validation — your other outputs are unaffected.
   ```

2. **Read the file** using the Read tool.

3. **Extract all fenced Mermaid blocks.** Identify every occurrence of a ` ```mermaid ` code fence and its closing ` ``` `. Note the line numbers and content of each block. If no Mermaid blocks are found, report "No Mermaid diagrams found in [file path]" and stop.

4. **Validate each block** by calling `mcp__claude_ai_Mermaid_Chart__validate_and_render_mermaid_diagram` with:
   - `mermaidCode`: the content between the fences (excluding the fence lines themselves)
   - `prompt`: "Validate this Mermaid diagram"
   - `diagramType`: inferred from the first line of the block (e.g. `flowchart`, `sequenceDiagram`, `gantt`, `classDiagram`)
   - `clientName`: "claude"

   If the tool call fails mid-run (e.g. transient error after the precheck passed), report the error and continue to the next block — do not fail the caller's workflow.

5. **Fix broken diagrams.** For each block that fails validation:
   - Read the error details from the tool response
   - Determine the fix based on the error message
   - Use the Edit tool to replace the broken mermaid content with the corrected version (use the full block content as `old_string` and the fixed content as `new_string`)
   - Re-validate by calling the MCP tool again
   - If it still fails, attempt one more fix (maximum 2 retries per block)
   - If a block remains broken after 2 retries, mark it as unfixable and continue to the next block

6. **Report results.** Return a summary containing:
   - Total number of Mermaid blocks found
   - Number that passed validation on first attempt
   - Number that were fixed (with a brief note of what was wrong)
   - Number that remain broken after retries (with the error details)
