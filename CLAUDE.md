# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install       # install dependencies
npm run dev       # Vite dev server on http://localhost:3000 (opens browser automatically)
npm run build     # production build into build/ (not dist/)
```

There is no test runner, linter, or `tsconfig.json` configured. Vite (SWC) strips types without type-checking, so type errors will not fail the build.

## Architecture

A single-page React 18 + Vite treasure-hunt game, originally exported from Figma Make.

- **All game logic and UI live in `src/App.tsx`.** State is `boxes` (3 boxes, one randomly holding treasure), `score`, and `gameEnded`. Opening the treasure box gives +$100, a skeleton box -$50. The game ends when the treasure is found or every box is open. `initializeGame()` runs on mount and on "Play Again".
- Animations use `motion/react` (Framer Motion's successor), not `framer-motion`.
- Images (`src/assets/*.png`) and sounds (`src/audios/*.mp3`) are imported directly as ES modules and resolved by Vite. `src/assets/key.png` is meant as a custom cursor for hovering over closed chests. Credits for third-party assets go in `Attributions.md`.
- `src/components/ui/` is a vendored shadcn/ui component set (Radix + `class-variance-authority` + `cn()` from `ui/utils.ts`). Only `Button` is currently used. These files import packages with a pinned version suffix (e.g. `"@radix-ui/react-slot@1.1.2"`). `vite.config.ts` aliases these back to the real package names, so if you add a new versioned import you also have to add a matching alias.
- `@` is aliased to `src/`.

## Styling caveat

Tailwind is **not** installed or run as part of the build. `src/index.css` is a pre-compiled, static Tailwind v4 output that only contains the utility classes used when the project was exported. A new Tailwind class that is not already in `index.css` (e.g. `bg-purple-500`) will silently have no effect. Before using a class, grep `src/index.css` for it. If it's missing, use inline `style={{...}}`, add a hand-written rule to `index.css`, or set up Tailwind properly. `src/styles/globals.css` (the theme tokens) is not imported anywhere.

## Project conventions

- The README is a Claude Code tutorial script for this repo. Later steps add sound effects, win/tie/loss results, a key-cursor hover, SQLite sign-up/sign-in with per-user scores plus a guest mode, and `.claude/commands/` deploy commands for Vercel and GitHub Pages.
- `src/guidelines/Guidelines.md` is an empty Figma Make template and has no active rules.
