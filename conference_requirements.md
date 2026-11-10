# Conference requirements — CSMD-UTM (article template only)

**Venue:** Technical Scientific Conference of Undergraduate, Master and PhD Students, Technical University of Moldova (CSMD-UTM).
**Source:** official article template, `Paper_template-Conf-SMD-UTM-2026-eng.docx`, downloaded 2026-11-10 from
<https://csmd.utm.md/en/templates-2026/> (direct link: <https://csmd.utm.md/wp-content/uploads/2026/04/Paper_template-Conf-SMD-UTM-2026-eng.docx>).
Local copy: [conference_templates/csmd2026_paper_en/](conference_templates/csmd2026_paper_en/). Zotero citation style: [conference_templates/csmd2026_zotero_style/](conference_templates/csmd2026_zotero_style/).

Scope: only the **article** template. Abstract and presentation templates, deadlines and other UTM venues are deliberately out of scope.

## 1. Status of the template

| Item | Finding |
|---|---|
| Edition the template belongs to | 2026 edition (footer: "Chisinau, May 13-16, 2026, Vol. N"). That edition is past; this is the most recent published template. A 2027 template may change details. |
| Official LaTeX template | **None.** Word (.docx) only. The template says "The use of this template is mandatory." |
| Consequence | The LaTeX project will be an **unofficial reproduction** of the Word template's layout. It must not be presented as official. Final submission may require transferring the text into the .docx. |

## 2. Page limit — conflict and decision

- Template text: *"The paper will contain maximum 4 pages for article and 1-2 pages for thesis."*
- Published 2025 CSMD papers are 4 pages long (observed by the earlier search before it was stopped).
- **Decision (user, 2026-11-10):** write a full article of at least 10 pages (target 10–12) in CSMD formatting, ignoring the 4-page cap. **Open risk:** the organizers may reject or require shortening a paper over 4 pages. Getting explicit approval from the supervisor or organizers is the user's responsibility.

## 3. Page layout and typography (as stated in the template)

| Element | Requirement | Template evidence |
|---|---|---|
| Paper size / orientation | A4, portrait | text + `pgSz 11906×16838` twips |
| Margins | top, bottom, right 2.0 cm; left 2.5 cm | text + `pgMar` (1134/1134/1134/1418 twips) |
| Header | "Technical Scientific Conference of Undergraduate, Master and PhD Students, Technical University of Moldova" | header1.xml |
| Footer | "Chisinau, <dates>, Vol. N" + centred page number "- n -" | footer1.xml |
| Body font | Times New Roman 12 pt | text |
| Line spacing | 1.5 lines in body paragraphs (`w:line="360"`) | document.xml |
| Paragraph indent | First line 1.25 (template text says "1.25pt"; actual value 708 twips = **1.25 cm**) | document.xml |
| Page numbering | Automatic | text |
| Title | UPPERCASE, centred, TNR 14 pt, bold, single spaced | text |
| Authors | Given name FAMILY NAME, centred, TNR 12 pt, bold; numeric superscript affiliation markers; `*` for corresponding author | text |
| Affiliations | "Department, group/PhD school, Faculty, University, City, Country", 10 pt, italic, centred | text |
| Corresponding author | "*Corresponding author: Given name Family name, email", 10 pt, normal, centred | text |
| Supervisor line | "Tutor/coordinator: Given name FAMILY NAME, scientific title, affiliation" | text |
| Abstract | Label "Abstract."; 150–200 words; TNR 12 pt italic | text |
| Keywords | 4–6 keywords, alphabetical, lowercase, comma-separated, 12 pt italic, justified; must not repeat terms from the title | text |
| Section headings | Template is inconsistent. In the XML, headings are TNR 12 pt, **unnumbered**, first-line indent 1.25 cm, not all caps. "Introduction", "Figures and Tables" and "References" are bold; the "Title of the section 1" and "Conclusions" placeholders are not. The text under "Conclusions" says "12pt. Bold, all capital letters", but that reads as a title instruction. **Assumption:** bold, 12 pt, sentence case; numbered subsections allowed for readability (to confirm). | text / document.xml |

## 4. Structure

Required order: Title → Authors → Affiliations → Abstract → Keywords → Introduction and problem formulation → Experimental/methodological part → Results and discussion → Conclusions → Acknowledgments (optional) → References.

Additional: *"Original scientific works … should have compulsory original findings and enough elements allowing the reproducibility of the experiments."*

A **Declaration of Originality and Responsibility** page (signed by the corresponding author) is part of the template. It is a separate signed form, not counted as article content.

Our planned sections (Background/Related Work, Architecture, Methodology, Limitations) fit within this structure.

## 5. Figures, tables, equations, units

| Element | Requirement |
|---|---|
| Figure captions | **Below** the figure; "Figure n." in bold, 11 pt; centred; cited as "Fig. 1" |
| Table titles | **Above** the table; 11 pt bold ("Table 1" on its own line, then the title); cited as "Tab. 1" |
| Tables | Centred, full page width, fit an A4 portrait page; bold header row with units; **no vertical lines**; continued tables repeat the headings |
| Equations | Centred, numbered "(1)" at the right margin, 3 pt space above/below; cited as "Eq. (1)" |
| Symbols | Italic for symbols, bold for vectors, upright for functions (log, exp) and text subscripts; define each symbol on first use; SI units; decimal point (not comma); units written as "J·kg⁻¹·K⁻¹" |
| Large formulas/schemas | Present as figures |
| Reused material | Cite the source at the end of the caption |

## 6. Citations and references

| Item | Requirement |
|---|---|
| In-text | Numeric, square brackets, numbered in order of first citation, placed before punctuation: [1], [1–3], [1,3] |
| Consistency | Every reference is cited and every citation has a reference |
| List | TNR 12 pt, left-aligned, numbered |
| Journal names | **Full** names; the template says "abbreviations are not allowed". (The template's example line and the Zotero CSL use the short journal title. We follow the explicit text rule: full names.) |
| Journal format | `AUTHOR, A.B.; AUTHOR, C.D. Title. Journal Year, Volume, pages, doi` (CSL: year in bold, volume in italic, DOI as `https://doi.org/...`) |
| Conference format | `AUTHOR, A.B.; … Title. In Proceedings of …, Location, Date; Publisher: City, Year; pages` |
| Websites | `Title of Site. Available online: URL (accessed on Day Month Year).` |
| Author names | FAMILY NAME in uppercase, initials, separated by "; " |

LaTeX implementation plan: BibTeX with a custom `.bst`, or biblatex with a customised style, reproducing the CSL output as closely as practical. Exact fidelity to the CSL is not guaranteed; any differences will be documented in the README.

## 7. Are design-oriented papers without experiments allowed?

Sources checked on 2026-11-10: the template text, plus the 2026 call for papers (`Call-Conf-UTM-2026-ENG.pdf`, linked from <https://csmd.utm.md/en/call-for-papers-2026/>, full text extracted).

| Source | Statement | Implication |
|---|---|---|
| Call for papers | "Full paper submission in accordance with the template"; "Submitted papers will be subject to a peer-review process." Section C covers Computers, Informatics and Microelectronics, subsections C1 Software Engineering and Security, C2 Systems Engineering, C3 Computer Science. | Nothing on paper type. Suitable section: **C2 or C3**. |
| Template | "Original scientific works and short communications should have compulsory **original findings** and enough elements allowing the reproducibility of the experiments." | Original findings are mandatory. Experiments are not stated as mandatory, but reproducibility is expected *where experiments exist*. |
| Template | "All papers **normally** contain the Title, …, an **Experimental (or methodological part)**, Results and discussion, Conclusions …" | A methodological part is explicitly an alternative to an experimental part. A "results and discussion" part is normally expected. |

**Conclusion.** No rule prohibits a design-oriented paper. The submission constraint is that the paper must contain **original findings** and a **results-and-discussion-type section**. The article meets both through:
1. the source-verified analysis of Karmada v1.19.1 scheduling semantics, with file and line references, so it can be reproduced;
2. the formal design;
3. the analytical worked example in the "Design Analysis and Discussion" section, explicitly labelled as calculation, not measurement.

**Residual risk:** reviewers may still expect empirical results. Confirm with the supervisor; this cannot be settled from the published rules.

## 8. Open items for the user (not resolved by the template)

1. The page-limit override needs confirmation from the supervisor or organizers.
2. Whether the organizers accept a PDF generated from LaTeX or require the .docx.
3. Author block details: group, supervisor name and title, email.
