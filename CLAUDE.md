# CLAUDE.md — AI Assistant Guide for delivery-launcher

## Project Overview

**PwC Delivery Launchpad** is a collection of self-contained, single-file HTML/JavaScript tools for professional services engagement planning, execution, and assessment. Each tool runs entirely in the browser with no backend server required.

## Repository Structure

```
delivery-launcher/
├── index.html                  # Main delivery engagement planning tool (~13K lines)
├── dashboards-index.html       # 30-pattern dashboard catalog (~3.6K lines)
├── strategic-execution         # Strategy execution builder (React-based, ~1.9K lines)
├── ppm-pmo-tool-selector       # Tool evaluation/comparison matrix (~1K lines)
├── pwc-design-system           # Design system component library (~3.3K lines)
├── maturity-assessment         # PPM/PMO assessment JS module (JSON config)
├── dist/
│   └── maturity-assessment.html  # Built single-file assessment wizard (~1K lines)
└── CLAUDE.md                   # This file
```

### File Descriptions

| File | Purpose | Tech |
|------|---------|------|
| `index.html` | Core delivery launchpad — guided workflow for engagement planning with prompts, form data collection, and export | Vanilla JS |
| `dashboards-index.html` | Catalog of 30 dashboard design patterns with live examples | Vanilla JS |
| `strategic-execution` | Strategy execution framework builder with PDF export | React 18 + Babel + jsPDF |
| `ppm-pmo-tool-selector` | Side-by-side comparison matrix for PPM/PMO tools | Vanilla JS |
| `pwc-design-system` | PwC-branded component library reference (buttons, cards, badges, etc.) | HTML + CSS |
| `maturity-assessment` | Assessment model config (domains, scoring rules, priorities) | JS module with embedded JSON |
| `dist/maturity-assessment.html` | Compiled single-file maturity assessment wizard | Vanilla JS |

## Architecture

### Single-File HTML Pattern

Every tool is a **self-contained HTML file** with all CSS and JavaScript embedded inline. There is no build system, bundler, or package manager. Files can be opened directly in a browser.

- No `package.json`, no npm dependencies (except CDN-loaded libs in `strategic-execution`)
- No build step required for most tools
- `dist/maturity-assessment.html` is the one built artifact, compiled from the `maturity-assessment` module

### Common UI Patterns

1. **Page/Tab Switching**: Pages use `.page { display: none; }` / `.page.active { display: block; }` with JS toggling
2. **Form Data**: Global `formData` object persisted to `localStorage` via `doSaveData()` / `doLoadData()`
3. **Modals**: `.modal-overlay` with `.hidden` class toggling for confirmations, warnings, and dialogs
4. **Workflow State**: `completedSteps` Set tracking with `unlockNextSteps()` for progressive disclosure
5. **Card States**: `active-card`, `locked-card`, `completed-card` CSS classes for workflow step cards
6. **Data Export**: CSV export, bulk prompt export, and PDF export (jsPDF + html2canvas in strategic-execution)
7. **Inline Editing**: `contenteditable="true"` for prompt customization with ID-based indexing (P1.1, P1.2, etc.)

### Design Tokens (CSS Variables)

All tools share a consistent PwC brand palette defined as CSS custom properties:

```css
--orange500: #fd5108;    /* Primary brand color */
--orange300: #ffaa72;    /* Secondary/light orange */
--orange100: #ffe8d4;    /* Background orange */
--grey100: #eeeff1;      /* Light surface */
--grey200: #dfe3e6;      /* Borders */
--grey300: #cbd1d6;
--grey400: #b5bcc4;
--grey500: #a1a8b3;      /* Secondary text */
--success: #22c55e;
--warning: #eab308;
--bgColor: #eeeff1;
--cardBg: #ffffff;
--textColor: #000000;
```

Font: `system-ui, -apple-system, sans-serif` (no custom web fonts).

## Development Workflow

### No Build System

There is no build tool, linter, test runner, or CI pipeline. Development is manual:

1. Edit the HTML file directly
2. Open in browser to test
3. Commit and push

### No Automated Tests

There are no unit tests, integration tests, or end-to-end tests. Quality is maintained through manual testing and code review.

### No Linting/Formatting

There is no ESLint, Prettier, or other formatting configuration. Follow the existing conventions described below.

## Code Conventions

### JavaScript

- **Vanilla JS** preferred (no frameworks except React in `strategic-execution`)
- **camelCase** for variables and functions: `formData`, `doSaveData()`, `unlockNextSteps()`
- **UPPER_CASE** for constants
- **2-space indentation** throughout
- Functions are defined at the top level within `<script>` tags
- DOM manipulation via `document.getElementById()`, `document.querySelector()`
- Data persistence via `localStorage`
- No ES modules (except `maturity-assessment` source) — everything is global scope within each file

### CSS

- **kebab-case** for class names: `.modal-overlay`, `.active-card`, `.header-left`
- Use CSS custom properties (design tokens) for colors — do not hardcode hex values
- Flexbox and Grid for layout
- Animations via `@keyframes` (fadeIn, slideDown, spin)
- Responsive patterns used throughout

### HTML

- Semantic HTML5 structure
- `aria-label` attributes for accessibility on interactive elements
- Inline `<style>` and `<script>` blocks (no external files except CDN dependencies)
- `data-*` attributes for JS-driven state

## Key Conventions for AI Assistants

1. **Preserve single-file architecture**: Do not split tools into multiple files. Each tool must remain a single self-contained HTML document.

2. **No new dependencies**: Avoid adding npm packages or new CDN dependencies. The project deliberately has zero dependency management overhead.

3. **Match existing code style**: 2-space indentation, camelCase JS, kebab-case CSS, design token variables for colors.

4. **Large file awareness**: `index.html` is ~13,000 lines and ~585KB. Read specific sections rather than the entire file. Use grep/search to locate relevant code sections.

5. **Prompt ID system**: The main `index.html` uses a structured prompt ID scheme (P1.1, P1.2, P2.1, etc.) for workflow prompts. When modifying prompts, maintain correct ID sequences and cross-references.

6. **Template literal safety**: Be careful with template literals in prompt text — a past bug (commit 262eb00) involved template literal delimiters causing injection. Use plain string concatenation or escape backticks in user-facing prompt content.

7. **localStorage usage**: All tools store state in localStorage. Key names should be descriptive and tool-specific to avoid collisions.

8. **No backend**: All tools are client-side only. Do not introduce server-side logic or API calls to external services.

9. **PwC branding**: Maintain the orange (#fd5108) primary color and existing design system. Do not change the core visual identity.

10. **File naming**: Tool files in the root directory do not always have `.html` extensions (e.g., `strategic-execution`, `maturity-assessment`). The `dist/` directory contains built `.html` versions.

## Git Workflow

- **Main branch**: `main` (remote) / `master` (local tracking)
- **Feature branches**: `claude/<description>-<id>` naming convention
- **Commit messages**: Descriptive, prefixed with type when appropriate (`feat:`, `fix:`, etc.)
- **No CI/CD**: No automated checks run on push or PR
- **No .gitignore**: All files are tracked; be mindful not to add generated or sensitive files
