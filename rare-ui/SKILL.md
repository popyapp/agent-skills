---
name: rare-ui
description: Install and use Rare UI, a shadcn registry of animated React components (Motion + Tailwind + TypeScript). Picks the right component from a catalog of nineteen, gets the namespaced shadcn install command right, flags the ones that drag in heavy dependencies, and keeps the registry's own contract - cn() merged className, props spread on the root, data-slot, and prefers-reduced-motion left alone. Use for "rare ui", "rareui", "animated React component", "fluid orb", "gooey nav", "family drawer", "otp input".
---

# Rare UI

[Rare UI](https://rareui.com) by [Swami Malode](https://github.com/swamimalode07)
is a **shadcn registry**, not a package: `npx shadcn add` copies a component's
source into your repo and you own it from then on. Next.js + Tailwind +
TypeScript, animated with [Motion](https://motion.dev). MIT, and the source of
truth for what is installable is
[`registry.json`](https://github.com/swamimalode07/rare-ui/blob/main/registry.json).

This skill is Popy's packaging of someone else's library. The components,
their design and their code are the author's.

## 0 · Before installing anything

The project needs shadcn set up already — a `components.json` at the root. If
there isn't one:

```bash
npx shadcn@latest init
```

Rare UI is a **namespaced registry**, so the component argument is
`owner/repo/component` — not a URL, and not a bare name:

```bash
npx shadcn@latest add swamimalode07/rare-ui/fluid-orb
```

Getting that shape wrong is the most common failure. `add fluid-orb` resolves
against the default shadcn registry and installs something else, or nothing.

Every component depends on the registry's `utils` item (the `cn()` helper), so
the first install also writes `lib/utils.ts` if it is missing.

## 1 · Pick by intent, not by name

Nineteen components. Read the row, not the label — several names do not
describe what they do.

| Component | For |
|---|---|
| `fluid-orb` | An animated WebGL orb with drifting fluid shading, in the manner of a voice-mode indicator. No dependencies. |
| `grid-reveal` | A loading state for an AI-generated image that resolves into the real picture when it arrives. |
| `gooey-nav` | A nav bar where the selected item separates from the group with a gooey pull. |
| `bounce-sidebar` | Vertical nav, spring-animated active indicator. |
| `hook-sidebar` | Vertical nav, dashed rail marking the active item. |
| `proximity-sidebar` | Sidebar that appears on scroll and reacts to pointer proximity and scroll intensity. |
| `scroll-progress` | A reading-progress pill that expands into a squircle menu of jump targets. |
| `folder-component` | A folder whose cards fan out on hover and lift open on click. `color`, `size`. |
| `family-drawer` | A bottom drawer that morphs between stacked views. Built on Vaul. **Installable but not listed on the site** — see §5. |
| `duration-picker` | Gooey hours-and-minutes duration entry. |
| `otp-input` | One-time-code entry; digits roll into place behind a sliding caret. |
| `delete-button` | Destructive confirm in place, no dialog. |
| `code-block` | A code block that derives its whole theme from one accent hex. |
| `gravity-letters` | A gravity field where letters, emoji or arbitrary children fall and pile up. |
| `github-activity` | Contribution heatmap with an expanding panel ranking top repositories. |
| `emoji-reaction` | Tapback-style reaction bar; picks float up out of the button. |
| `notification-bell` | Bell with an unread-count badge. |
| `step-player` | Stepped progress track with play, pause and replay; the active step fills as it plays. |
| `animated-counter` | Odometer-style digit wheel counting to a new value. |

## 2 · Check the dependency before you install

Each component declares its own npm dependencies, and a few are heavier than
the component looks. In a bundle-sensitive app, read them first:

- `motion` — almost everything. Assume it.
- `duration-picker` — `figma-squircle`, `flubber`, `react-use-measure`, `@radix-ui/react-slot`
- `step-player` — `flubber`
- `family-drawer` — `vaul`
- `emoji-reaction` — `react-apple-emojis`, `lucide-react`, `@radix-ui/react-slot`
- `code-block` — `prism-react-renderer`, `lucide-react`
- `fluid-orb`, `gravity-letters` — none

`flubber` (path interpolation) and `react-apple-emojis` (a whole emoji image
set) are the two that cost real weight for one visual effect. Name that
trade-off to the user rather than installing quietly.

## 3 · The code is yours — edit it, don't wrap it

The component lands in `components/ui/<name>.tsx`. There is no package to
upgrade and no upstream to wait for.

So restyle by editing the file. Do **not** add a wrapper whose only job is to
override styles you could have changed in place — that is the habit a registry
exists to remove, and it leaves two sources of truth.

If you do extend one, match the hygiene every shipped component follows:

- `cn()` merges an incoming `className` onto the root
- remaining props are spread onto the root element
- the root carries a `data-slot` attribute

## 4 · Motion is already handled

Every interactive component honours `prefers-reduced-motion`. Do not
reimplement that check, and do not strip it while "simplifying" an animation —
it is the accessibility contract of the library, and it is already correct.

## 5 · Two things that will waste your time

- **`family-drawer` is not on the website.** It is commented out of the site's
  component list but is still a live entry in `registry.json`, so the install
  command works. If a user asks for it and the docs page 404s, install it
  anyway.
- **`npm run registry:build` is not your job.** That script regenerates
  `public/r/*.json` and only matters when contributing to Rare UI itself.
  Consuming projects never run it.

## 6 · Credit, and what the author asks

The repo is MIT. The author additionally asks, in the site's own words: free in
personal and commercial projects, attribution appreciated, and don't resell the
components as your own kit.

He also states plainly that most components are recreations of work from around
the web rather than original inventions, and credits sources per component. If
you lift one into a public showcase, carry that credit through — link
[rareui.com](https://rareui.com) and the
[repo](https://github.com/swamimalode07/rare-ui).
