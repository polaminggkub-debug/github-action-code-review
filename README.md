# AI Code Review GitHub Action

Free AI-powered code review using GitHub Models (GPT-4o Mini).

## Quick Setup (One Click Copy)

Copy the entire block below and paste it to your AI assistant (Cursor, ChatGPT, Claude, etc.):

```
Please create a new file at the following exact path:

.github/workflows/ai-review.yml

Write the file using EXACTLY the YAML content below.

Do NOT modify, rewrite, or change anything.

Create the file and paste this YAML verbatim.

name: AI Code Review (GitHub Models Free)

name: AI Code Review (GitHub Models Free)

on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write
  models: read

jobs:
  review:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Fetch full history for proper diff comparison

      - name: Get Changed Files
        id: files
        run: |
          # Fetch base branch to ensure we have it for comparison
          git fetch origin ${{ github.base_ref }}:refs/remotes/origin/${{ github.base_ref }}

          # Get the merge base between the base branch and PR head
          MERGE_BASE=$(git merge-base origin/${{ github.base_ref }} HEAD)
          echo "Merge base: $MERGE_BASE"

          # Get changed files between merge base and PR head
          CHANGED=$(git diff --name-only "$MERGE_BASE"...HEAD || true)

          if [ -z "$CHANGED" ]; then
            echo "No changes detected via merge-base, trying HEAD~1"
            CHANGED=$(git diff --name-only HEAD~1 || true)
          fi

          if [ -z "$CHANGED" ]; then
            echo "No changes via HEAD~1, using diff-tree"
            CHANGED=$(git diff-tree --no-commit-id --name-only -r HEAD || true)
          fi

          echo "Changed files:"
          echo "$CHANGED"

          echo "FILES<<EOF" >> $GITHUB_ENV
          echo "$CHANGED" >> $GITHUB_ENV
          echo "EOF" >> $GITHUB_ENV

      - name: Build Review Payload
        id: payload
        run: |
          echo "Detected changed files:"
          echo "$FILES"
          
          # Get merge base for consistent diff
          MERGE_BASE=$(git merge-base origin/${{ github.base_ref }} HEAD)
          echo "Using merge base: $MERGE_BASE"
          
          FILE_COUNT=0
          PAYLOAD_FILE=$(mktemp)
          
          # Read FILES line by line to handle newlines properly
          while IFS= read -r f || [ -n "$f" ]; do
            if [ -n "$f" ] && [ -f "$f" ]; then
              FILE_COUNT=$((FILE_COUNT + 1))
              echo "Processing file $FILE_COUNT: $f"
              
              # Append file content and diff to payload file (plain text, no escaping)
              {
                echo ""
                echo "========================================"
                echo "FILE: $f"
                echo "------------ FULL FILE --------------"
                cat "$f"
                echo ""
                echo "------------ DIFF CHANGES -----------"
                git diff "$MERGE_BASE"...HEAD -- "$f" || true
                echo "========================================"
              } >> "$PAYLOAD_FILE"
            fi
          done <<< "$FILES"
          
          echo "Total files processed: $FILE_COUNT"
          
          if [ $FILE_COUNT -eq 0 ]; then
            echo "ERROR: No files to review!"
            exit 1
          fi
          
          # Write to GITHUB_ENV using heredoc (preserves newlines)
          {
            echo "REVIEW<<EOF"
            cat "$PAYLOAD_FILE"
            echo "EOF"
          } >> $GITHUB_ENV
          
          echo "Payload written to REVIEW env var ($(wc -c < "$PAYLOAD_FILE") bytes)"

      - name: Run AI Code Review (GPT-4o Mini)
        id: ai
        run: |
          # Write base JSON to file
          cat > request.json << 'JSONEOF'
          {
            "model": "gpt-4o-mini",
            "messages": [
              {
                "role": "user",
                "content": "Perform a complete and careful code review.\n\nReview Goals:\n- Read the full content of each modified file.\n- Use the diff as context.\n- Identify logic bugs, missing checks, unsafe assumptions, crash risks, security issues, concurrency problems, redundant code, and dangerous changes.\n\nRules:\n- Do NOT rewrite entire files.\n- Provide minimal, safe, actionable fixes with BEFORE/AFTER code diffs.\n- Flag anything suspicious.\n- Use the ACTUAL filename from the provided files, not placeholder text.\n- Show code changes in diff format (- for removed lines, + for added lines).\n\nRequired Output Format (use emojis exactly as shown):\n## 🔍 AI Code Review Summary\n\n### 🔴 Critical Issues\n- (list or 'None identified')\n\n### 🟠 Major Issues\n- (list or 'None identified')\n\n### 🟡 Minor Issues\n- (list or 'None identified')\n\n---\n\n## 📁 File-by-File Review\n\nFor EACH file in the diff, create a section like this:\n\n### 📄 `actual_filename.ext`\n\n| Severity | Issue |\n|----------|-------|\n| 🔴 Critical | (description or 'None') |\n| 🟠 Major | (description or 'None') |\n| 🟡 Minor | (description or 'None') |\n\n**🛠️ Suggested Fixes:**\n\nFor each issue, show the fix as a code diff:\n\n```diff\ndef function_name():\n-    old_buggy_code = \"bad\"\n-    more_old_code()\n+    new_fixed_code = \"good\"\n+    better_approach()\n```\n\n**Explanation:** Brief reason why this fix is better.\n\n---\n\n## 💡 Additional Observations\n- (optional tips, best practices, or praise for good code)\n\nModified files (full content + diff):\n\n"
              }
            ]
          }
          JSONEOF

          # Check if REVIEW env var has content
          if [ -z "$REVIEW" ]; then
            echo "ERROR: REVIEW env var is empty!"
            echo "FILES env var: $FILES"
            exit 1
          fi
          
          echo "REVIEW env var length: ${#REVIEW} characters"
          
          # Write REVIEW content to temp file (preserve newlines)
          printf '%s' "$REVIEW" > review_content.txt
          
          # Verify content file was created
          if [ ! -s review_content.txt ]; then
            echo "ERROR: review_content.txt is empty!"
            exit 1
          fi
          
          echo "Content file size: $(wc -c < review_content.txt) bytes"
          
          # Inject payload using rawfile (jq handles JSON escaping automatically)
          jq --rawfile content review_content.txt '.messages[0].content += $content' request.json > payload.json
          
          # Verify payload was created
          if [ ! -f payload.json ] || [ ! -s payload.json ]; then
            echo "ERROR: payload.json is empty or missing!"
            exit 1
          fi
          
          echo "Payload created successfully"

          # Call GitHub Models API
          RESPONSE=$(curl -s \
            -H "Content-Type: application/json" \
            -H "Accept: application/json" \
            -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
            -d @payload.json \
            https://models.inference.ai.azure.com/chat/completions \
            | jq -r '.choices[0].message.content // "No response from AI"')

          echo "AI_RESPONSE<<EOF" >> $GITHUB_ENV
          echo "$RESPONSE" >> $GITHUB_ENV
          echo "EOF" >> $GITHUB_ENV

      - name: Write Comment Body to File
        run: |
          cat > comment.md << 'EOF'
          ## 🤖 AI Code Review
          
          <table>
          <tr><td>🧠 <b>Model</b></td><td>GPT-4o Mini</td></tr>
          <tr><td>📊 <b>Status</b></td><td>✅ Review Complete</td></tr>
          </table>

          ---
          
          EOF
          echo "$AI_RESPONSE" >> comment.md
          cat >> comment.md << 'EOF'
          
          ---
          <sub>🔄 Powered by GitHub Models • Review generated automatically on PR</sub>
          EOF

      - name: Comment Review to PR
        uses: peter-evans/create-or-update-comment@v3
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          issue-number: ${{ github.event.pull_request.number }}
          body-path: comment.md

```

---

## How It Works

1. **Copy** the block above (click 📋 button)
2. **Paste** into your AI assistant (Cursor, ChatGPT, Claude)
3. **Commit & Push** the file it creates
4. **Create a Pull Request** to any branch
5. **Wait ~30 seconds** for the AI to review
6. **See the result** → Go to your Pull Request page → Scroll down to **Comments** section → AI review will appear there! 💬

![Where to find the review](https://docs.github.com/assets/cb-35103/mw-1440/images/help/pull_requests/conversation-tab.webp)

---

## Features

- 🤖 Uses GitHub Models Free (GPT-4o Mini)
- 📝 Reviews full file content + diffs
- 🔍 Identifies bugs, security issues, and code quality problems
- 💬 Comments directly on your PR
- 🔑 No API key required - uses built-in `GITHUB_TOKEN`

## License

MIT
