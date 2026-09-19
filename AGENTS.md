# Repository guidance

This repository is a bilingual Hugo technical blog. The default language is Korean, and post bundles normally live under `content/post/<slug>/` with `index.ko.md` and, when available, `index.en.md`.

These instructions apply to the entire repository. Preserve user-authored changes already present in the worktree, and do not commit, push, publish, or edit generated output unless explicitly requested.

## What “polish the new post” means

When asked to polish, refine, or improve a newly added post:

1. Run `git status --short` and inspect the new or modified post bundle. Do not assume the newest Git commit is the target when an untracked post exists.
2. Compare the post with recent, polished posts, especially:
   - `content/post/cilium-troubleshooting-tproxy-src-valid-mark/`
   - `content/post/golang-slog/`
3. Preserve the author’s technical intent and personal voice. Improve structure and clarity without turning the article into generic marketing copy.
4. Fix clear spelling, spacing, grammar, terminology, and code typos. Do not silently change a technical claim unless it is clearly incorrect and the correction can be verified.
5. Replace forced Markdown line breaks such as trailing `\` with natural paragraphs unless the break has a deliberate visual purpose.
6. Break dense passages into short paragraphs, add brief transitions, and explain why an example matters. Avoid padding the post with unrelated background material.
7. Fill an incomplete section only when its intended content is clear from the surrounding article. Otherwise keep the scope honest and report the gap.
8. Apply the front matter, heading, emphasis, code, reference, multilingual, and validation rules below.

“Polish” includes readability, document structure, metadata, formatting consistency, and obvious correctness fixes. It does not authorize a wholesale rewrite, a change of technical conclusions, or publication outside this repository.

## Front matter

Use YAML front matter and prefer this field order for a completed post:

```yaml
---
title: "Readable post title"
description: "One concrete sentence describing what the reader will learn"
date: 2026-09-20T00:00:00+09:00
lastmod: 2026-09-20T00:00:00+09:00
slug: post-slug
image:
math: false
license:
hidden: false
comments: true
draft: false

tags:
    - Primary technology
    - Secondary topic

categories:
    - Primary category
---
```

- Preserve the original `date` when editing an existing article.
- Add or update `lastmod` for a substantive edit, using the actual edit time and the `+09:00` timezone.
- Keep the same `slug` across language variants.
- Write `description` in the document’s language and make it specific enough to work as a search/social summary.
- Use lowercase `tags` and `categories` keys. Follow the existing taxonomy spelling when a matching tag or category already exists.
- Set `math` explicitly to `true` or `false`.
- When a post contains images, use the first image in the article as the default front matter `image` so the post has a thumbnail. For numbered assets, this normally means image 1.
- An explicit user instruction to use a particular thumbnail or cover image always overrides the first-image default.
- Use the bundle-relative filename expected by Hugo, keep the thumbnail synchronized across language variants when they share the same assets, and verify that the referenced file exists.
- Do not invent an image filename. Leave `image:` empty when the post has no image asset or usable article image.
- Do not change `draft` merely to make a build pass. A fully written translation may be changed from `draft: true` to `draft: false` when the user asked to complete and publish that language version.

## Post structure and writing style

- Put a thematically appropriate emoji in every H2 heading.
- Put a horizontal rule immediately before every H2:

  ```md
  ---

  ## 🧭 Section title
  ```

- The front matter closing delimiter does not count as the first section’s horizontal rule.
- Use H3 headings for real subsections, not merely to emphasize a sentence.
- Korean prose should normally use a consistent explanatory `~이다` / `~한다` style.
- English prose should sound like natural technical English, not a sentence-by-sentence literal translation.
- Keep standard product and API casing, such as Go, Kubernetes, Cilium, Tailscale, Handler, `slog`, and `context.Context`.
- Prefer short, cohesive paragraphs with one main idea each.
- Use lists for actual sets, comparisons, prerequisites, or procedures. Do not convert ordinary prose into excessive bullets.
- Use bold selectively for conclusions, warnings, and important operational rules. A page should not feel highlighted everywhere.
- Use inline code for commands, identifiers, field names, functions, sysctls, marks, paths, and literal values.
- Use fenced code blocks with the correct language identifier.
- Introduce a code block with a sentence and explain the important result after it when the meaning is not self-evident.

## Markdown and Goldmark pitfalls

Hugo uses Goldmark/CommonMark-style delimiter rules. Bold wrapped directly around inline code can fail when a Korean particle follows it without a space.

Do not write:

```md
**`net.ipv4.conf.all.src_valid_mark=1`**과
**`accept_local=0`**이면
```

The closing `**` may be rendered literally. Prefer plain inline code:

```md
`net.ipv4.conf.all.src_valid_mark=1`과
`accept_local=0`이면
```

Inline code already has strong visual contrast in the site theme, so it normally does not need bold. If both styles are genuinely required, use explicit HTML such as `<strong><code>value</code></strong>` and verify the rendered result.

Also:

- Leave blank lines around headings, lists, horizontal rules, and fenced code blocks.
- Keep opening and closing code fences balanced.
- Avoid raw URLs in prose and reference lists; use descriptive Markdown links.
- Do not add manual HTML solely for visual decoration when normal Markdown works.

## Code excerpts and sources

- Preserve the behavior of user-provided examples while fixing obvious API spelling or syntax errors.
- For code copied or adapted from a real project, kernel tree, specification, or other source, place the source on the first line inside the code block:

  ```go
  // source: https://github.com/owner/repository/blob/main/path/to/file.go
  func example() {}
  ```

- Prefer an exact file URL or repository-relative source path when it is known. A repository root URL is acceptable when the exact path cannot be verified.
- Never fabricate a source path, revision, benchmark result, command output, or version.
- Keep source comments synchronized between Korean and English variants.
- When an excerpt is shortened, use comments such as `// ...` without implying that the omitted code is unnecessary in the real program.

## Korean and English variants

- Treat `index.ko.md` and `index.en.md` in the same bundle as translations of one article.
- When both versions contain real content, keep their section order, code examples, source comments, diagrams, conclusions, and references synchronized.
- Translate meaning and tone, not Korean word order.
- Localize titles, descriptions, section headings, transitions, and explanatory prose. Keep code, identifiers, commands, and URLs unchanged unless the code itself needs correction.
- After writing English, search for accidental Korean text. After changing a shared code example, check both files.
- Do not create or publish a missing language version unless the user asks for it. A placeholder draft is not a completed translation.

## Links and references

- Use Hugo `relref` shortcodes for links to other posts in this repository.
- Use descriptive Markdown links for external references.
- Prefer primary sources: official documentation, specifications, upstream source code, or the project repository.
- Keep a final section named `## 📚 References` when the article relies on external material.
- Do not carry a reference into a translation if it no longer supports the translated claim.

## Validation

After changing posts:

1. Inspect `git status --short` again and make sure unrelated user changes were not modified.
2. Check that every H2 has an emoji and a preceding horizontal rule.
3. Check for malformed emphasis, especially the pattern of bold wrapped around inline code.
4. Check that code fences are balanced and front matter is complete.
5. For bilingual posts, compare headings and shared code/source lines across both variants.
6. Build the site with:

   ```bash
   hugo --minify --destination /tmp/hugo-blog-render
   ```

7. Run `git diff --check` for tracked files. Remember that normal `git diff` does not show untracked posts, so inspect new files directly.

Do not edit `public/`, `resources/`, or other generated output to fix a rendering issue. Fix the source content or configuration instead.
