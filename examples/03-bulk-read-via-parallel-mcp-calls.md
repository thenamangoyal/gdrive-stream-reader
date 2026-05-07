# Example 3: Read 10 Google Docs in parallel

You ask:

> Read all the `.gdoc` files in `drivedata/2026 Q1 reviews/` and tell me the common themes.

Claude (with this skill loaded) does:

1. List the directory:

   ```bash
   ls "drivedata/2026 Q1 reviews/" | grep '\.gdoc$'
   ```

2. Extract all `doc_id`s in one batch:

   ```bash
   for f in "drivedata/2026 Q1 reviews/"*.gdoc; do
     jq -r '.doc_id' < "$f"
   done
   ```

3. Issue **all `read_file_content` MCP calls in a single message** (one tool block per file, batched concurrently):

   ```
   read_file_content { fileId: "1AAA..." }
   read_file_content { fileId: "1BBB..." }
   read_file_content { fileId: "1CCC..." }
   ... (etc, all 10 in one message)
   ```

4. Synthesize the bodies it gets back.

## Why parallel matters

Each `read_file_content` call is a network round-trip to Google. Sequential reads of 10 docs take ~10× as long as 10 parallel reads. Concrete measurement on a typical setup:

- Sequential: ~30s for 10 docs.
- Parallel: ~3–4s for 10 docs.

The skill explicitly tells Claude to batch these. If you watch a session without the skill, Claude will often loop one-by-one out of caution.

## What if one doc is too large?

If any one of the 10 returns the >25k-token error, the MCP saves *that* doc's content to disk and returns the saved path. The other 9 still come back inline. Claude handles the saved one via Procedure C (`jq` decode + chunked read), which is independent.
