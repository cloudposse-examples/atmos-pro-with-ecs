---
name: update-demo
description: Toggle the demo app background color between blue and green. Use when the user says "update the demo", "flip the color", "change the color", or similar.
---

# Update Demo: Toggle App Color

Toggle the app's background color between `blue` and `green`.

## Instructions

1. Read `terraform/stacks/defaults/app.yaml`
2. Find the `COLOR` environment variable value (under `containers.app.environment`)
3. Toggle the value:
   - If `blue` → change to `green`
   - If `green` → change to `blue`
4. Edit the file to update the color value
5. Report: "Flipped color from **{old}** to **{new}**"
