---
excerpt: "Are you tired of repeatedly pressing 'allow' in Claude Code? There is a solution"
title: "A Sensible Allowlist for Claude"
tags: [claude, llm]
author: addamsson
short_title: "A Sensible Allowlist for Claude"
comments: true
updated_at: 2026-03-09
---

> If you're tired of pressing "allow" all the time when working with Claude Code but you still want to retain safety then
> this quick article will help you tremendously.


## The Problem

I realized the other day that I'm getting more and more annoyed at all the "do you allow `x`" prompts given by Claude so I decided to look for a solution.

Naturally...I asked Claude to give me a sensible default allowlist. The result was a long list of `Bash(...)` items in the `"allow"` section.

I quickly realized that this is not enough as there are top level functions that it executes **a lot**, so I added `"Read"`, `"Glob"`, `"Grep"` and `"WebFetch"` too. Now it complains **a lot less**. Here is the complete list:


## The Solution
> 📘 This list belongs in `~/.claude/settings.json`

```json

{
    "permissions": {
        "allow": [
            "Read",
            "Glob",
            "Grep",
            "WebFetch",

            "Bash(cat *)",
            "Bash(curl *)",
            "Bash(cut *)",
            "Bash(date)",
            "Bash(df *)",
            "Bash(du *)",
            "Bash(echo *)",
            "Bash(file *)",
            "Bash(find *)",
            "Bash(free *)",
            "Bash(git log *)",
            "Bash(git show *)",
            "Bash(git status)",
            "Bash(git status *)",
            "Bash(git branch *)",
            "Bash(git diff *)",
            "Bash(git tag *)",
            "Bash(git remote *)",
            "Bash(grep *)",
            "Bash(head *)",
            "Bash(jq *)",
            "Bash(ls *)",
            "Bash(ls *)",
            "Bash(mvn *)",
            "Bash(npx *)",
            "Bash(npm *)",
            "Bash(pnpm *)",
            "Bash(ps *)",
            "Bash(pwd)",
            "Bash(rg *)",
            "Bash(sort *)",
            "Bash(stat *)",
            "Bash(tail *)",
            "Bash(tree *)",
            "Bash(uname *)",
            "Bash(uniq *)",
            "Bash(wc *)",
            "Bash(whoami)",

            "Bash(* --version)",
            "Bash(* --help *)"
        ],
    },
}
```


Let me know if this doesn't work or if I missed something!
