---
# Paper Notes template. Copy it to _posts/YYYY-MM-DD-<conference>-<yyyy>-notes.md,
# e.g. _posts/2026-10-21-osdi-2026-notes.md (published at
# /blogs/osdi-2026-notes.html), and replace the sample text. One post per
# conference, one "## Paper title" section per paper.
# Drafts are never published; `bundle exec jekyll serve --drafts` previews this one.
title: "Paper Notes: OSDI'26"
topic: Paper Notes
description: >-
  Optional: a plain-text summary of the whole post, shown in the "AI Summary" card.
  Delete it, or ask Claude to write it once the post is done.
---

# Paper Notes: OSDI'26

One or two sentences: the conference, and how the papers below were picked.

{% comment %}
How to summarize a paper
------------------------
Give each paper one section, in the order below, in about 300-500 words. A paper
that needs more deserves its own post, linked from here.

1. Title: the "## Paper title" heading, with a short id ({: #id}) so other pages
   can link to it, e.g. /blogs/osdi-2026-notes.html#paper-short-name.
2. Meta line: the authors' affiliations (institutions, not every author; say so
   when it comes from industry), then links to the paper and its code or artifact.
3. Tags: one to three from this list, spelled exactly as here, so papers can be
   grouped by tag later. Add a new tag to the list before using it.
     Network stack, Kernel bypass, RDMA, SmartNIC/DPU, Cache injection, CXL,
     Memory tiering, Storage, Virtualization, GPU
4. In one sentence: the key idea. Readers who skim read only this line.
5. Background: the problem, and why existing systems fall short (2-3 sentences).
6. Approach: the key insight first, then the mechanism that realizes it.
7. Evaluation: the setup (real hardware or emulated, e.g. CXL memory emulated by a
   remote NUMA node; which NIC, GPU or CXL device), the baselines, and the
   headline numbers.
8. Food for thought: limitations, what you would question or try next, and how
   it relates to your own work.

(This guide stays below the first paragraph: Jekyll takes that paragraph as the
post's excerpt, and a comment cut in half there makes it print a warning.)
{% endcomment %}

## Paper Title
{: #paper-short-name}

*University A, Company B* · [Paper](https://example.org/paper.pdf) · [Code](https://example.org/code)
{: .paper-meta}

- CXL
- Memory tiering
{: .paper-tags}

**In one sentence.** The key idea.

**Background.** The problem, and why existing systems fall short.

**Approach.** The key insight, then the mechanism.

**Evaluation.** Setup, baselines, headline numbers.

**Food for thought.** Limitations, open questions, relevance to my work.
