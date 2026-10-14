---
name: datacontract-editor
description: "Opens a data contract in the Data Contract Editor, the visual web editor for ODCS with a diagram, form, and YAML view, live HTML preview, validation, and a test runner: locally against a file with `datacontract edit` or `npx datacontract-editor`, in Docker, or at editor.datacontract.com, plus share links and organization-specific customization (`customization.yaml`). Use when the user wants to view, edit, review, or present a data contract visually, open the editor, share a contract with a business stakeholder, run the editor for a team, or tailor the editor's fields and custom properties."
---

# Data Contract Editor

**Persona:** anyone who reads or writes data contracts, including business stakeholders who do not read YAML
**Output:** the contract open in the editor, a share link, or an editor deployment

The [Data Contract Editor](https://github.com/datacontract/datacontract-editor) is an open-source web editor for ODCS contracts. It has a visual diagram of the schemas and relationships, a form editor, a YAML editor with completion, a live HTML preview, instant validation, and a "Run test" button that tests the contract through the Data Contract CLI. It is also part of [Entropy Data](https://entropy-data.com).

## Step 1: Pick how to run it

| Situation | Use |
|---|---|
| Edit a contract file in the user's repository | `datacontract edit <file>` (default) |
| No Python, but Node.js | `npx datacontract-editor <file>` |
| A shared editor for a team | Docker image `datacontract/editor` |
| Quick look, no install, nothing confidential | [editor.datacontract.com](https://editor.datacontract.com) |

## Step 2a: Edit a local file (default)

```bash
datacontract edit contracts/orders.odcs.yaml
```

- Needs the CLI with the `api` extra (included in `[all]`), see the `datacontract-cli` skill.
- Starts a local server on `127.0.0.1:4243` and opens the browser. The editor assets are bundled, so it works offline.
- **Saving in the editor writes directly back to the file.** "Run test" runs the contract tests locally against the contract's servers, with credentials from the environment or `.env`.
- If the file does not exist, it offers to create a new contract. If a URL is given, it offers to download a local copy.
- Options: `--port`, `--host`, `--no-open`, `--editor-version <npm version|latest>`.

The command blocks while the server runs. Start it in the background (for example with your shell tool's background mode) and tell the user the URL. Stop it when the user is done.

Alternative without Python: `npx datacontract-editor contracts/orders.odcs.yaml` (port auto-detected from 9090, `-p <port>` to set it).

## Step 3: Work together on the same file

The user edits visually while you may edit the YAML. To avoid overwriting each other:

- Before each of your edits, re-read the file: the user may have saved from the editor since you last read it.
- After your edit, run `datacontract lint`, and tell the user to reload the editor to see your change.
- Keep changes small and say what you changed, so the user can review it in the diagram or preview.

Good uses: you generate descriptions, quality rules, or a new schema from the user's request, and the user reviews and fine-tunes them in the form view; or the user sketches the model in the diagram and you fill in the details.

## Step 2b: The hosted editor and share links

[editor.datacontract.com](https://editor.datacontract.com) runs in the browser and keeps the contract in the browser session. Features such as the AI assistant send the contract to their backend. The user can paste YAML or open a file there.

A link can pre-load a contract: `https://editor.datacontract.com/?data=<base64 of the UTF-8 YAML, URL-encoded>`.

```bash
python3 -c 'import base64,sys,urllib.parse; print("https://editor.datacontract.com/?data=" + urllib.parse.quote(base64.b64encode(open(sys.argv[1],"rb").read()).decode(), safe=""))' contracts/orders.odcs.yaml
```

Before creating a share link, warn the user: the whole contract is inside the URL, so it ends up in browser history, chat logs, and anywhere the link is pasted. Do not create one for contracts with confidential content (internal hostnames, real example data, sensitive business rules) unless the user explicitly accepts that. Very large contracts may exceed URL length limits.

## Step 2c: Run the editor for a team (Docker)

```bash
docker run -d -p 4173:4173 datacontract/editor
```

Then open `http://localhost:4173`. Optional settings as environment variables:

- AI assistant: `AI_ENABLED=true`, `AI_ENDPOINT` (OpenAI-compatible chat completions URL), `AI_API_KEY`, `AI_MODEL`, `AI_AUTH_HEADER` (`bearer` or `api-key`). The key is used by the browser, so only use a key scoped for this purpose.
- Customization: mount a `customization.yaml` and set `CUSTOMIZATION_CONFIG` to its path.

For `datacontract edit`, the AI assistant uses `DATACONTRACT_EDITOR_AI_ENDPOINT`, `DATACONTRACT_EDITOR_AI_API_KEY`, `DATACONTRACT_EDITOR_AI_MODEL`, `DATACONTRACT_EDITOR_AI_PROVIDER` (`openai` or `anthropic`), and `DATACONTRACT_EDITOR_AI_AUTH_HEADER`, see [Edit your Contract](https://docs.datacontract.com/editor).

## Step 4: Customize the editor for an organization (optional)

A `customization.yaml` in the directory where the editor starts (or passed with `--customization` / `CUSTOMIZATION_CONFIG`) tailors the forms to the organization's conventions:

```yaml
dataContract:
  root:
    standardProperties:
      - property: domain
        required: true
        enum: [sales, marketing, finance]
    customProperties:
      - property: costCenter
        title: Cost Center
        type: string
        required: true
  schema.properties:
    standardProperties:
      - property: classification
        enum: [public, internal, confidential, restricted]
      - property: encryptedName
        hidden: true
```

- Levels: `root`, `description`, `schema`, `schema.properties`, `servers`, `team`, `team.members`, `roles`, `support`.
- `standardProperties` change labels, help texts, allowed values, defaults, and visibility of ODCS fields. `customProperties` add fields that are stored in the contract's `customProperties`. `customSections` group them.
- Keep `pattern` regular expressions simple; complex ones can freeze the browser.

Read the full reference in [CUSTOMIZATION.md](https://github.com/datacontract/datacontract-editor/blob/main/CUSTOMIZATION.md) before writing one, and check each field you restrict against the levels and properties listed there.
