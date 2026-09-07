# Tooling

## `pandoc-guide-template.html`

Pandoc HTML template for long-form reference docs (sidebar TOC, mermaid rendering,
dark syntax highlighting) matching the house dashboard styling.

Used to generate `2026-08-31/contract-integration-testing-implementation-guide.html`
from its markdown. Regenerate after editing the `.md`:

```bash
cd 2026-08-31
pandoc contract-integration-testing-implementation-guide.md \
  --from gfm --to html5 --standalone \
  --template ../_tooling/pandoc-guide-template.html \
  --toc --toc-depth=3 \
  --syntax-highlighting=breezedark \
  --metadata title="Contract & Integration Testing — Implementation Guide" \
  -o contract-integration-testing-implementation-guide.html
```

Notes:
- `--syntax-highlighting`, not the deprecated `--highlight-style`.
- Pandoc wraps fenced ```mermaid blocks as `<pre class="mermaid"><code>`; the
  template's inline script unwraps the `<code>` before calling `mermaid.run()`.
- The tabbed gap-analysis dashboards are hand-written, not generated — this
  template is only for the linear guide-style docs.

### Pre-publish checks

```bash
# category table row counts must match between .md and .html
# P0/P1 badge counts must sum to the figure claimed in the summary
# all relative hrefs must resolve:
for f in <folder>/*.html; do d=$(dirname "$f")
  grep -o 'href="[^"#][^"]*"' "$f" | sed 's/href="//;s/"$//' | grep -v '^http' | sort -u |
  while read -r l; do t="${l%%#*}"; [ -z "$t" ] && continue; [ -e "$d/$t" ] || echo "BROKEN $f -> $l"; done
done
```
