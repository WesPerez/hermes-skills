---
name: apple-ecosystem
description: "Use for Apple/macOS personal-productivity and device workflows: Notes, Reminders, Find My, iMessage/SMS, and macOS computer-use automation through local CLIs or GUI control."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [macos]
metadata:
  hermes:
    tags: [apple, macos, notes, reminders, imessage, findmy, computer-use]
    related_skills: []
---

# Apple Ecosystem Workflows

## Overview

Use this umbrella for Apple-local workflows on macOS: managing Notes, Reminders, iMessage/SMS, Find My devices/AirTags, or driving the GUI when no CLI exists. First confirm the current host is macOS and the needed helper CLI/app is installed.

## Routing Table

| Task | Backend pattern |
|---|---|
| Create/search/edit Apple Notes | `memo` CLI |
| Add/list/complete reminders | `remindctl` CLI |
| Send/read iMessage/SMS | `imsg` CLI |
| Locate devices/AirTags | FindMy.app / Find My automation |
| Operate arbitrary macOS UI | screenshot + accessibility/computer-use loop |

## Notes (`memo`)

- Use for quick capture, search, and note edits.
- Prefer exact title searches before creating duplicates.
- Report created/edited note title and folder/account when available.

## Reminders (`remindctl`)

- Include due date/time and list name when the user supplies them.
- For destructive changes or bulk completion, confirm the target list/items first.
- After adding or completing items, list the affected reminder(s) to verify.

## iMessage/SMS (`imsg`)

- Never send a message to a person/contact without user confirmation of recipient and text.
- For ambiguous names, resolve candidate handles before sending.
- Report delivery command status and message id/thread when available.

## Find My

- Use only for the user's own devices/items or when they explicitly have permission.
- Treat location data as sensitive; return concise location/time/battery facts.
- If the Find My app requires interactive login, ask the user to authenticate locally.

## macOS Computer Use

- Use GUI automation when no reliable CLI/API exists.
- Work from screenshots, accessibility labels, and small verified steps.
- Avoid irreversible GUI actions without confirmation.
- Keep the final answer grounded in observed state, not assumed UI state.

## Common Pitfalls

- These workflows are macOS-only; do not try them on Linux hosts.
- Apple apps often require the user to be logged into iCloud locally.
- Local CLIs may need Accessibility, Full Disk Access, Contacts, or Automation permissions.
- Contact names can be ambiguous; resolve before messaging.

## Verification Checklist

- [ ] macOS host and required helper are available
- [ ] Permissions/authentication are in place
- [ ] User confirmed any send/delete/bulk action
- [ ] Result was read back or otherwise verified
