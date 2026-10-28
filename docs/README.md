# RAG documents

Every Markdown/PDF/text file here goes into the **policy** index used by the coordinator, except the files named in
`HIPAA_DOCS` (default `summary_HIPAA.md`). Those form the **HIPAA** index used by the compliance agent. README files
are not indexed.

The documents were converted from PDF to Markdown and adapted so the RAG speaks the language of `claim_metrics.csv`.
Each **document type** is written for **several of the dataset's (fictional) insurers**. Every adapted file says so in
its header. **None of them is a real insurance policy.**

| Document type | Insurers offering the plan | Derived from |
|---|---|---|
| Inpatient Hospital Services | KiwiCare · Golden Pear Assurance · Redwood Apple Health Plans | CMS Medicare Benefit Policy Manual, Ch. 1 — Inpatient Hospital Services |
| Eligibility and Enrollment | Citrus Shield Insurance · Papaya Health Network · Blueberry Benefit Partners | CMS — Medicare eligibility and enrollment |
| Skilled Nursing Facility Care | Evergreen Orchard Payers · Harvest Grove Insurance · KiwiCare | Medicare.gov — Skilled nursing facility care |
| Hospice Care | MangoSure · Apricot Risk Management · Citrus Shield Insurance | Medicare.gov — Hospice care |
| Claims Denials and Appeals — Marketplace Plans 2023 | all payers (real insurers renamed) | KFF — Claims denials and appeals in ACA Marketplace plans (2023) |
| `summary_HIPAA.md` | — (law, unchanged) | HHS OCR — Summary of the HIPAA Privacy Rule |
| `state-of-claims-2025.md` | — (unchanged) | State of claims 2025 report |

The insurers are in each file's name and title. The contextual-retrieval step therefore puts them in every chunk's
context, so a question about one insurer finds the documents of that insurer.

**Replacement rules**
- **Coverage documents:** Medicare / CMS → *the plan* (the coverage of any insurer listed in the header) or *the plan administrator*.
  Part A / B / C / D → Hospital / Medical / Advantage / Drug Plan. medicare.gov / cms.gov links → "your insurer's member portal".
- **Medicaid** → "the state medical assistance program".
- **KFF analysis:** CMS is the regulator and is kept. Medicare → Golden Pear Assurance. The real insurers it names are renamed:
  Blue Cross Blue Shield → Blueberry Benefit Partners, UnitedHealthcare → Redwood Apple Health Plans,
  Cigna → Apricot Risk Management, Centene → Harvest Grove Insurance, Oscar → KiwiCare, Molina → MangoSure.

> **Licensing:** U.S. federal government works (CMS, HHS) are generally public domain. The KFF analysis is
> copyrighted, even in adapted form. Check its terms (or replace it with a link and a short summary) before
> publishing. Add links to the original sources here for your readers.
