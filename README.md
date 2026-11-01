# Telescope plugin for Claude Code

Preview your web app (Next.js or Vite) on your phone, at real desktop size, while Claude Code edits it.

```
/plugin marketplace add frederickrohn/telescope-plugin
/plugin install telescope@telescope
```

Then, inside a repo on your machine, ask Claude Code to "set up this repo on Telescope" (or run `/telescope:onboard`). It installs the `telescope` CLI (`npm i -g usetelescope`) if needed, signs you in, inspects the repo, and gets a preview onto your phone.
