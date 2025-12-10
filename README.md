# AI Code Review GitHub Action

Free AI-powered code review using GitHub Models (GPT-4o Mini).

## Quick Setup (Copy & Paste)

Just copy the prompt below and paste it to your AI assistant (Cursor, ChatGPT, Claude, etc.):

---

```
Please create a new file at the following exact path:

.github/workflows/ai-review.yml

Write the file using EXACTLY the YAML content below.

Do NOT modify, rewrite, or change anything.

Create the file and paste this YAML verbatim.
```

```yaml
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
          ref: ${{ github.head_ref }}

      - name: Get Changed Files
        id: files
        run: |
          git fetch --all || true

          CHANGED=$(git diff --name-only origin/${{ github.base_ref }}...HEAD || true)

          if [ -z "$CHANGED" ]; then
            CHANGED=$(git diff --name-only HEAD~1 || true)
          fi

          if [ -z "$CHANGED" ]; then
            CHANGED=$(git diff-tree --no-commit-id --name-only -r HEAD || true)
          fi

          echo "FILES<<EOF" >> $GITHUB_ENV
          echo "$CHANGED" >> $GITHUB_ENV
          echo "EOF" >> $GITHUB_ENV

      - name: Build Review Payload
        id: payload
        run: |
          PAYLOAD=""
          for f in ${{ env.FILES }}; do
            if [ -f "$f" ]; then
              FULL=$(sed 's/"/\\"/g' "$f")
              DIFF=$(git diff origin/${{ github.base_ref }}...HEAD -- "$f" | sed 's/"/\\"/g')

              PAYLOAD+="\n\n========================================\n"
              PAYLOAD+="FILE: $f\n"
              PAYLOAD+="------------ FULL FILE --------------\n$FULL\n"
              PAYLOAD+="------------ DIFF CHANGES -----------\n$DIFF\n"
              PAYLOAD+="========================================\n"
            fi
          done

          echo "REVIEW<<EOF" >> $GITHUB_ENV
          echo -e "$PAYLOAD" >> $GITHUB_ENV
          echo "EOF" >> $GITHUB_ENV

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
                "content": "Perform a complete and careful code review.\n\nReview Goals:\n- Read the full content of each modified file.\n- Use the diff as context.\n- Identify logic bugs, missing checks, unsafe assumptions, crash risks, security issues, concurrency problems, redundant code, and dangerous changes.\n\nRules:\n- Do NOT rewrite entire files.\n- Provide minimal, safe, actionable fixes.\n- Flag anything suspicious.\n\nRequired Output Format:\n## AI Code Review Summary\n\n### Critical Issues\n- (list)\n\n### Major Issues\n- (list)\n\n### Minor Issues\n- (list)\n\n---\n\n## File-by-File Review\n\n### <filename>\n#### Critical\n- \n#### Major\n- \n#### Minor\n- \n#### Suggested Fixes\n- \n\n---\n\n## Additional Observations\n- (optional)\n\nModified files (full content + diff):\n\n"
              }
            ]
          }
          JSONEOF

          # Inject payload
          jq --arg content "${{ env.REVIEW }}" '.messages[0].content += $content' request.json > payload.json

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

      - name: Comment Review to PR
        uses: peter-evans/create-or-update-comment@v3
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            ### AI Code Review (GitHub Models Free)
            Model: GPT-4o Mini

            ---
            ${{ env.AI_RESPONSE }}
```

---

## Features

- 🤖 Uses GitHub Models Free (GPT-4o Mini)
- 📝 Reviews full file content + diffs
- 🔍 Identifies bugs, security issues, and code quality problems
- 💬 Comments directly on your PR
- 🔑 No API key required - uses built-in `GITHUB_TOKEN`

## License

MIT
