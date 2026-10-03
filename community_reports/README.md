# Possible missing labels in the CUAD test split

`test_split_missing_labels.csv` lists 380 passages in 73 of the 99 CUAD **test-split** contracts reviewed that
appear to meet a CUAD category definition but carry no label for that category.


**Who reviewed:** every passage was ruled by an AI model (Claude), not by a lawyer, and no human has validated the rulings. Treat each row as a candidate for expert review, not as a confirmed error.

## How the list was made

1. An automated contract-clause extractor labelled the 99 test contracts.
2. Every passage it labelled with a category that CUAD does not label there was read by a
   reviewer against the category's definition in `category_descriptions.csv` (the written
   definition only - not the extractor's own guidance, and not CUAD's other labels).
3. A passage is listed only when it plainly meets the definition. Passages that were cut off,
   redacted, or borderline were left out.
4. Each listed passage was trimmed to the sentences that carry the clause (headings and
   unrelated sentences removed) and located in the CUAD contract text, so `answer_text` is
   CUAD's own text and `answer_start` its character offset, as in CUAD's annotations.

## Second pass

168 rows were added in a second pass, after a later run of the extractor over the same 99
contracts. The method is the same: each passage was read in full against the written
definition, kept only when it plainly meets it, and trimmed to the sentences that carry the
clause. A few clauses are long in themselves (an audit right with its provisos, for example)
and are kept whole.

## Columns

| Column | Meaning |
|---|---|
| `contract_title` | The contract's title as in the CUAD test data |
| `category` | The CUAD category the passage appears to belong to |
| `answer_start` | Character offset of `answer_text` in the contract's CUAD text |
| `answer_text` | The clause text, exactly as it appears in the CUAD contract text at `answer_start` |
| `why it meets the CUAD category definition` | The reviewer's one-line reason |

## Caveats

- An AI model's reading against the written definitions, not a legal review. An earlier 100-item sample, whose AI reviewers also checked how CUAD labels similar passages in the training split, found a similar share of missing labels (44%).
- Some categories (for example Post-Termination Services) are labelled selectively in CUAD's
  training data; whether these passages should be labelled is a dataset-policy question.
- Spans follow sentence boundaries; CUAD's annotators sometimes mark shorter spans.
- One row comes from a two-column table whose text CUAD's extraction interleaves; its span is
  the table region.
