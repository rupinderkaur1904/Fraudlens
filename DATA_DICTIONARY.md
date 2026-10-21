# Data Dictionary: IEEE-CIS Fraud Detection (Vesta)

Status: first draft, 2026-10-21. Numbers marked **verified** were computed on the files from
`ieee-fraud-detection.zip` (Kaggle competition download, files dated 2019-12-11). Field meanings
marked **organiser** come from the competition's data description; everything else is our own
interpretation and is marked **assumption**.

Raw files are never committed. Expected local path: `data/raw/`.

## Files

| File | Rows | Columns | Used? |
| --- | --- | --- | --- |
| `train_transaction.csv` | 590,540 | 394 | Yes: all train, validation and test periods come from here |
| `train_identity.csv` | 144,233 | 41 | Yes, joined on `TransactionID` (left join) |
| `test_transaction.csv`, `test_identity.csv` | 506,691 / 141,907 | 393 / 41 | No: labels are not public |
| `sample_submission.csv` | | | No |

## Verified summary

| Fact | Value |
| --- | --- |
| Labelled transactions | 590,540, no duplicate `TransactionID` |
| Frauds | 20,663 (3.499%) |
| Accuracy of predicting "legitimate" for everything | 96.5% |
| Time span | 182 days (`TransactionDT` 86,400 to 15,811,131 s); file is sorted by time |
| Transactions per day | min 2,048, median 3,049, max 6,852 |
| Rows with an identity record | 24.4% (fraud rate 7.85% with, 2.09% without) |
| Kaggle test file | starts 30 days after train ends, spans 183 days |
| pandas memory, default dtypes | about 2.2 GB for the transaction table |

Fraud rate by 30-day block (block 6 is only 2 days):

| Days | Rows | Frauds | Fraud rate |
| --- | --- | --- | --- |
| 0-29 | 130,968 | 3,243 | 2.48% |
| 30-59 | 89,838 | 3,627 | 4.04% |
| 60-89 | 91,768 | 3,700 | 4.03% |
| 90-119 | 98,027 | 3,849 | 3.93% |
| 120-149 | 85,303 | 2,962 | 3.47% |
| 150-179 | 86,525 | 2,943 | 3.40% |
| 180-181 | 8,111 | 339 | 4.18% |

The first month has more transactions and a lower fraud rate than the rest. This is a sign of
seasonality or drift (**assumption**: a holiday period) and matters for the split.

## Label

`isFraud` (int, 0/1, no missing values).

**Organiser:** a transaction is fraud (1) when a chargeback is reported on the card; later
transactions linked to the same user account, email address or billing address are also labelled
fraud. A transaction is legitimate (0) if nothing is reported within 120 days.

Consequences:

- Labels are *confirmed-by-chargeback*, not *investigated fraud*. Unreported fraud is labelled 0.
- Labels cluster by entity (see "Entity proxies" below). Features that identify an entity can
  learn "this entity was fraudulent before".
- In a real deployment a label is known only after a chargeback (or 120 days). Our split needs a gap.

## Timestamp

`TransactionDT` (int, seconds). **Organiser:** a time delta from a hidden reference datetime, not a
real timestamp. Day number = `TransactionDT // 86400`. Hour of day = `(TransactionDT // 3600) % 24`
(**assumption**: the reference is midnight in some timezone, so hour is only relative).

## Field groups

| Field(s) | Type | Missing | Meaning | Tier |
| --- | --- | --- | --- | --- |
| `TransactionID` | int | 0% | Row key; join key to identity | Not a feature |
| `TransactionAmt` | float | 0% | Amount in USD (**organiser**). Min 0.25, median 68.77, max 31,937.39 | T0 |
| `ProductCD` | str, 5 values | 0% | Product code. W 74%, C 12%, R 6%, H 6%, S 2%. Fraud rate C 11.7%, W 2.0% | T0 |
| `card1`-`card6` | mixed | 0-1.5% | Masked card info: `card4` network (visa, mastercard, ...), `card6` type (debit, credit, ...); `card1` 13,553 values | T0 (and entity proxy) |
| `addr1`, `addr2` | float | 11.1% | Masked billing region and country (**organiser**) | T0 (and entity proxy) |
| `dist1`, `dist2` | float | 59.7%, 93.6% | Distances between addresses (**organiser**, exact meaning unknown) | T0 |
| `P_emaildomain`, `R_emaildomain` | str | 16.0%, 76.8% | Purchaser and recipient email domain | T0 |
| `M1`-`M9` | str (T/F, M0-M2) | 28.7-59.3% | Match checks, e.g. name on card vs address (**organiser**) | T0 |
| `C1`-`C14` | float | 0% | Counts, e.g. addresses linked to the card; meaning masked | Vendor (history) |
| `D1`-`D15` | float | 0.2-93.4% | Time deltas in days, e.g. since previous transaction | Vendor (history) |
| `V1`-`V339` | float | 0-86.1% (median 47.3%) | Vesta-engineered features: ranking, counting, entity relations | Vendor (history) |
| `id_01`-`id_38` | mixed | median 11.7% | Identity: network, browser, OS, device signals; masked | T0 (24% of rows only) |
| `DeviceType` | str | 2.4% of identity rows | desktop or mobile | T0 |
| `DeviceInfo` | str | | Device/OS string | T0 |

Tiers follow the plan: T0 = describes only the current transaction; Vendor = columns that already
encode history and are excluded from the T0 baseline.

## Entity proxies (verified, computed in time order over all 182 days)

There is no customer ID. Three candidate proxies:

| Proxy | Entities | Median rows per entity (per row) | Rows with at least 5 earlier tx | Frauds whose entity had an earlier fraud |
| --- | --- | --- | --- | --- |
| `card1` + `addr1` | 39,974 | 125 | 81.2% | 83.4% |
| `card1`-`card6` + `addr1` + `P_emaildomain` | 94,873 | 26 | 62.2% | 74.7% |
| `card1` + `addr1` + (day - `D1`) | 217,850 | 5 | 26.1% | 60.7% |

- The first two are too coarse to be one customer: a median of 125 transactions in 6 months for
  one card is implausible for e-commerce. They behave more like card-and-region segments.
- The third is a community heuristic (**assumption**: `D1` is days since the card was first seen,
  so day - `D1` is a fixed "start day" per card). It gives customer-sized groups but short histories.
- All three show strong label clustering, confirming the label-propagation risk.

## Known limitations

1. No true customer, account, merchant or device ID.
2. Most fields are masked; meanings are partly unknown.
3. `TransactionDT` is relative, so calendar effects (weekday, hour) are only approximate.
4. Identity data covers only 24.4% of rows, and its presence correlates with fraud.
5. Labels are chargeback-based and propagated by entity; they include label noise.
6. Licence: Kaggle competition rules, not an open licence. Data must not be redistributed.
   (Exact wording to be copied from the competition rules page.)
