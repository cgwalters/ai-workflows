# agenthome

Shared agent configuration for running arbitrary AI agent actions in
GitLab CI, currently with [opencode](https://opencode.ai) as the
harness. An "action" is any task you can express as an agent plus a
prompt — merge request review is the first one, but the same setup
runs triage, changelog drafting, advisory checks, or anything else a
pipeline wants an agent to do against a repository checkout.

- `opencode/` — the opencode harness configuration: providers (via
  the enmaas proxy, expecting an `ENMAAS_KEY` environment variable)
  and agent definitions, including the read-only `review` agent.
- `prompts/` — harness-agnostic task prompts, one file per action.
  A CI job passes the prompt verbatim as the user message.
- `bin/safe-output` — minimal safe-outputs tool (see below).
- `bin/selftest` — tree self-test, run at the end of the image build
  and usable from a checkout.

The container image is built here: `Containerfile.opencode`
publishes `quay.io/jotnar/opencode` via the repository's
build-and-push workflow. Pin an immutable commit-SHA tag rather than
a moving ref, so a change to this tree or to the base image can
never silently alter an action's behavior.

## Safe outputs

> **TODO**: replace this tree-local tool with generic safe-outputs
> tooling as it matures; see
> [fullsend-ai/fullsend#6614](https://github.com/fullsend-ai/fullsend/pull/6614).

Like [gh-aw safe
outputs](https://github.github.com/gh-aw/reference/safe-outputs/), the
agent never holds an API token or write credentials. It requests side
effects through a narrow, validated tool, and a separate privileged
job executes them. This tree ships exactly one output:

- `safe-output review` — records the markdown body of a review
  comment. The body comes from stdin or `--file PATH`; anything else
  (empty, placeholder-short, oversized, or a different output type)
  is rejected. It lands in `$SAFE_OUTPUTS_DIR/review.json` (default
  `/tmp/agenthome-safe-outputs`, preset in the image) as
  `{"type": "review", "body": ...}`.

Output is JSON, like gh-aw's safe outputs, so the privileged executor
can validate and dispatch on it rather than scraping agent chatter.
Review is just the first special case; future output types (comments,
created PRs, ...) will be additional records, see e.g. [gh-aw's
pull-request review
comments](https://github.github.com/gh-aw/reference/safe-outputs-pull-requests/#pr-review-comments-create-pull-request-review-comment).
A generating CI job declares `$SAFE_OUTPUTS_DIR` as an artifact, and a
small post job holding the MR token reads `review.json` and posts the
body as a note (idempotently — the previous note is edited in place,
preserving history). Extend the tool only when a consumer actually
needs another output type; keep it a validator, not a gateway to
arbitrary API calls.

## Running an action from a GitLab CI job

Preferred: the prebuilt container image,
`quay.io/jotnar/opencode:<commit-sha tag>`, built by this
repository's build-and-push workflow whenever the tree or the
Containerfile changes:

    my-agent-action:
      image: quay.io/jotnar/opencode:<commit-sha-tag>
      variables:
        ENMAAS_KEY: $MY_ENMAAS_KEY_VARIABLE
      script:
        # Task context: MR number, title, description excerpt.
        - printf 'MR !123: Update to 1.2.3\n\nRebase on the new upstream release.\n' > task.txt
        - opencode run --standalone --agent review \
            "$(cat "$XDG_CONFIG_HOME/prompts/review.md" task.txt)" --file mr.diff

The image has `XDG_CONFIG_HOME` preset to the installed `agenthome/`
tree, so opencode picks up the providers, agents, and prompts
automatically; any prompt in `prompts/` or agent in `opencode/` can
be invoked the same way. Pin the commit-SHA tag rather than a moving
ref, so a config change can never silently alter an action's
behavior.

Clone-based use (no image dependency) also works:

    git clone --depth 1 -b <ref> https://github.com/packit/ai-workflows.git /tmp/agenthome-src
    export XDG_CONFIG_HOME=/tmp/agenthome-src/opencode
    export ENMAAS_KEY="$MY_ENMAAS_KEY_VARIABLE"
    curl -fsSL https://opencode.ai/install | bash -s -- --no-modify-path
    export PATH="$HOME/.opencode/bin:$PATH"
    opencode run --standalone --agent review \
        "$(cat "$XDG_CONFIG_HOME/prompts/review.md" task.txt)" --file mr.diff

## The review action

`prompts/review.md` performs a sanity-check review of a merge request
in a git repository: does this change, on its own, make sense? The
intent is for it to generalize to any repository; RPM packaging
(CentOS Stream / RHEL dist-git) repositories are the important first
target. A consumer pipeline may attach the dist-gitlab-review
preprocessor artifacts (`codereview-summary.yaml`, plus
`codereview-git.tar.gz` extracted into the working tree as the
`_source_repo/` preprocessed-sources git repository) and the MR diff;
the prompt prefers them when present. Working consumer: the
`redhat-internal-test-package` review pipeline.

### Review action contract

The contract a consumer pipeline can rely on, so it can post/process
the output without scraping agent chatter:

- **Input**: task context as the user message (MR number, title,
  description excerpt), plus `--file` attachments: the MR diff and,
  when available, `codereview-summary.yaml`. The pipeline extracts
  `codereview-git.tar.gz` into the working tree beforehand, providing
  `_source_repo/`, a two-commit git repo of the `%prep`-applied
  sources.
- **Emission**: the agent calls `safe-output review` exactly once
  with its complete final review (markdown, via `--file` or stdin);
  the validated body lands in `$SAFE_OUTPUTS_DIR/review.json` as
  `{"type": "review", "body": ...}`. Emission is **required**: a
  skipped or broken safe-output call fails the generating job — there
  is no stdout fallback — and in the future agentic-job will poke the
  agent via ACP to ensure the tool has been run.
- **Output format** (concise, verdict first): either the single line
  `No concerns found.`, or a body starting with `Blockers:` for
  high-confidence misalignment with the change's stated intent, or
  `Concerns:` for uncertain states that depend on maintainer intent
  or missing context — each item one line, `file:line` plus a short
  explanation. A consumer should publish the body verbatim.
- **Provenance trailer**: the privileged post job appends an enforced
  trailer to every posted note — hyperlinks to the exact
  implementation (commit of the posting script and the agenthome
  image) and to the run logs — the same attribution discipline as
  gh-aw safe outputs and the hummingbird review bot. The trailer is
  added by the executor, never trusted to the agent, so no posted
  review can lack it.
- **Failure modes**: the agent never modifies files; a broken or
  skipped safe-output call fails the generating job rather than
  degrading to scraping agent stdout.

The `opencode/opencode.jsonc` is kept in sync with
https://github.com/cgwalters/homegit/blob/main/macos-csb/.config/opencode/opencode.jsonc
plus CI-specific additions such as the `review` agent.

## Related projects

**[hummingbird-agent](https://gitlab.com/redhat/hummingbird/tools/-/tree/main/hummingbird-agent)**
(redhat/hummingbird/tools) is a centralized agent service covering
some of the same ground from the opposite direction. Where agenthome
runs stateless one-shot jobs inside the consumer's CI pipeline,
hummingbird-agent is an always-on service: GitLab webhooks and slash
commands drive a Lambda-based runner that creates a placeholder
discussion, runs an agent loop (Anthropic or Gemini via its own
`hummingbird-ai` provider layer), and posts the final model text
into that discussion. Sessions persist to S3, and users can reply to
the bot note to continue the conversation — the note carries a JSON
session marker as an HTML comment — the same provenance goal as our
JSON safe-output records.

Its credential separation is gh-aw-shaped like ours: the agent's tool
registry holds only a sandbox shell, data-fetch wrappers, and
read-only GitLab/Konflux/Testing Farm queries; all writes go through
orchestrator and per-workflow action tokens that never enter the
registry. Sandboxing is stronger than ours — a network-less Podman
container by default, or an ephemeral VM — and is being extracted
into a shared library for agents and humans alike (HUM-10106).

Its `code-review` workflow is the closest analogue to our review
action, with ideas worth borrowing: a mandatory per-repo rules
lookup that overrides generic review standards (HUM-5089, born from
a real false positive), awareness of prior review discussions so
re-reviews do not re-flag addressed or justified issues, and an
LLM-as-judge eval harness (dev/holdout MR sets, blind three-way
comparison against the `ai-code-review` tool) that makes prompt
changes measurable. Tracking for what we adopt from it lives in
PACKIT-5545.
