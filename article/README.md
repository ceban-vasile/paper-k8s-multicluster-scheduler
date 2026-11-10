# Article: Design of a Resource- and Network-Aware Cluster Selection Strategy for Karmada-Based Workload Placement

A design-oriented paper for the CSMD-UTM student conference. It contains **no implementation and no experimental results**; those belong to the master's thesis.

## Build

Option A (used here): [Tectonic](https://tectonic-typesetting.github.io/) 0.17.0, installed at `C:\Users\VasileCeban\tools\tectonic\tectonic.exe`.

```
cd article
C:\Users\VasileCeban\tools\tectonic\tectonic.exe -X compile main.tex
```

The first run downloads the required TeX packages, so it needs internet access. Later runs use the cache. BibTeX runs automatically.

Option B: TeX Live or MiKTeX with `latexmk` (configuration in `latexmkrc`): `latexmk main.tex`. XeLaTeX is required because of `fontspec` (Times New Roman).

Output: `main.pdf`.

## Structure

| Path | Content |
|---|---|
| `main.tex` | Preamble, title block, section includes |
| `csmd.sty` | **Unofficial** LaTeX reproduction of the CSMD-UTM 2026 Word template, including the biblatex reference style |
| `sections/00_abstract.tex` … `10_conclusions.tex` | One file per section, in the structure required for this article |
| `figures/architecture.tex`, `pipeline.tex`, `regions.tex` | TikZ/pgfplots figures (vector). `regions` plots analytic functions, not data. |
| `references.bib` | 35 entries, all cited; metadata verified, see `../literature_matrix.csv` |

## Formatting assumptions (no official LaTeX template exists)

Taken from the Word template:
- A4; margins 2/2/2 cm, 2.5 cm left;
- Times New Roman 12 pt, 1.5 line spacing, 1.25 cm first-line indent;
- title 14 pt bold uppercase; affiliations 10 pt italic;
- bold 11 pt captions, figures below and tables above;
- no vertical lines in tables;
- numeric citations [n] in order of appearance;
- "Fig. n", "Tab. n", "Eq. (n)" references.

Assumptions (see `../conference_requirements.md`):
- Section headings are numbered so that sections can be cross-referenced. Use `\setcounter{secnumdepth}{0}` to remove the numbers.
- The footer layout (text on the left, page number in the centre) is approximate.
- The reference style approximates the template and the Zotero CSL. Differences: website entries end with "accessed on …" in a separate sentence; conference entries follow biblatex's standard order.
- The template is mandatory in Word form. The organisers may require the final text in the `.docx` template.

## Fill in before submission

- `main.tex`: department, group, email, tutor/coordinator, and the footer dates of the target edition (`\setcsmdfooter`).
- Check the author name format.
- Confirm with the supervisor or organisers that (a) more than 4 pages and (b) a design paper without experiments are accepted (see `../conference_requirements.md`, §2 and §7).
- The template's "Declaration of Originality and Responsibility" is a separate signed form and is not part of this PDF.

## Integrity notes

- Every statement about Karmada refers to release v1.19.1 (commit c64ba38) and was checked in the source code (see `../notes/karmada_v1.19.1_verification.md`).
- The worked example (Tab. 6–7, Fig. 3) uses chosen input values and is labelled as such. **No measured results appear in the paper.**
- Sources with metadata-only verification are not used for specific claims.

## Remaining limitations and future research (thesis)

Implementation of the plugins and the latency probe; the multi-cluster laboratory with netem latency emulation and k6 traffic; comparison against default Karmada and resource-only selection; dynamic rebalancing; measured-utilisation signals; several client populations; service-to-service latency for split workloads; performance, scalability and reliability evaluation. See `../future_validation_plan.md`.
