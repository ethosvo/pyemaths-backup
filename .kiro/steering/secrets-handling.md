---
inclusion: always
---

# Secrets handling

This project keeps credentials in `.env` (gitignored, never committed).

When a task requires AWS or other credentials stored in `.env`:

- Never `read_file` or `cat` the `.env` file to view raw secret values in chat.
- Never echo, print, or include secret values in tool output, commands, or responses.
- Reference credentials only by their variable name (e.g. `AWS_MAIN_ACCESS_KEY`).
- To use credentials in a shell command, `source .env` inside the same bash
  invocation and export mapped variables (e.g. `AWS_ACCESS_KEY_ID`) without
  printing them, e.g.:

  ```bash
  set -a; source .env; set +a
  export AWS_ACCESS_KEY_ID="$AWS_MAIN_ACCESS_KEY"
  export AWS_SECRET_ACCESS_KEY="$AWS_MAIN_SECRET_ACCESS_KEY"
  export AWS_DEFAULT_REGION="$AWS_MAIN_REGION"
  aws sts get-caller-identity
  ```

  All of this happens in one non-interactive bash command so the values are
  never displayed, only used by the invoked process.
- If a command's output could leak a secret (e.g. an error message echoing
  back a key), redact it before including it in the response.
