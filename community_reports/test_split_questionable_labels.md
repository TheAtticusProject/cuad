# Possibly incorrect labels in the CUAD test split

`test_split_questionable_labels.csv` lists 206 CUAD answers in 59 **test-split** contracts whose
text does not appear to meet the definition of the category it is labelled with.


**Who reviewed:** every passage was ruled by an AI model (Claude), not by a lawyer, and no human has validated the rulings. Treat each row as a candidate for expert review, not as a confirmed error.

## How the list was made

1. An automated contract-clause extractor labelled the test contracts.
2. Every CUAD answer that the extractor did not label with the same category was read by a
   reviewer against the category's definition in `category_descriptions.csv` (the written
   definition only - not the extractor's own guidance).
3. An answer is listed only when it plainly does not meet the definition. Answers that were cut
   off, redacted, or borderline were left out.
4. Each row is CUAD's own annotation: `answer_text` and `answer_start` are copied unchanged from
   the test data, so a row identifies exactly one existing label.

## Second pass

87 rows were added in a second pass, after a later run of the extractor over the same
contracts, with the same method: each CUAD answer was read in full against the written
definition and listed only when it plainly does not meet it. Twenty-two rows were later withdrawn: 21 lead-ins and fragments, and one duplicate.

## Columns

| Column | Meaning |
|---|---|
| `contract_title` | The contract's title as in the CUAD test data |
| `category` | The CUAD category the answer is labelled with |
| `answer_start` | The answer's character offset, as in CUAD |
| `answer_text` | The answer's text, as in CUAD |
| `why it does not meet the CUAD category definition` | The reviewer's one-line reason |

## Caveats

- An AI model's reading against the written definitions, not a legal review. In an earlier 100-item sample whose AI reviewers also checked how CUAD labels similar passages in the training split, fewer CUAD labels were judged wrong than in this list, so some rows may follow CUAD's own labelling practice.
- Minimum Commitment accounts for 63 rows. Its definition asks for an amount one party must
  *buy* from the counterparty; CUAD also labels minimum sales efforts, sales-force sizes and
  royalty floors. Whether those belong in the category is a dataset-policy question.
- CUAD's annotators keep a clause's lead-in and mark dropped sub-parts with "<omitted>"; lead-ins, sub-parts and other short fragments are therefore not listed, since they cannot be judged on their own.
