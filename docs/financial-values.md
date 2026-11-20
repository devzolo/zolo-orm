# Decimal values from the language

[Documentation](README.md) · [Example](../examples/decimal_math.zolo) · [ERP persistence](erp-persistence.md)

The ORM uses Zolo's `decimal` and `bigdecimal` directly. Arithmetic, parsing and
rounding belong to the language; the ORM maps the values and validates column
precision/scale. This integration is source-only and has not been compiled or
executed in this batch.

## Declare the value you need

```zolo
use orm::Model

struct Invoice deriving Model {
  @key(generated: true)
  id: int,
  @decimal(precision: 18, scale: 2)
  amount: decimal,
  @decimal(precision: 60, scale: 8)
  exchange_total: bigdecimal?,
}
```

No codec import or wrapper construction is required. `decimal` defaults to
`DECIMAL(28, 2)` and `bigdecimal` to `DECIMAL(38, 2)`. Optional values preserve
SQL NULL. Creates, patches, predicates and projections retain the declared
native type.

A native `decimal` column cannot declare precision above 28: a database value
within a wider column could exceed the language value's range. Choose
`bigdecimal` for wider columns or calculations needing more precision.
Numeric precision is 1..1000, scale is 0..precision; individual database engines
can impose narrower bounds. Arbitrary precision in the application does not
make a fixed SQL column unlimited.

## Calculate with native operators

```zolo
let price: decimal = 19.99d
let subtotal = price * 3d
let tax = (subtotal * 0.075d).quantize(2, .HalfEven)?
let payable = subtotal + tax

let large: bigdecimal = 123456789012345678901234567890.01bd
let installment = large.divide(12bd, scale: 2, rounding: .HalfEven)?
```

These are the same language values used outside the ORM. Native fixed decimal
arithmetic retains its existing range/precision semantics; choose `bigdecimal`
before an intermediate calculation that needs more precision. The fallible
`divide` method rounds the exact quotient directly at the requested scale,
without relying on an intermediate float or default division precision.

`quantize` and `divide` accept `std::decimal::Rounding`:
TowardZero, AwayFromZero, Floor, Ceiling, HalfUp and HalfEven. HalfUp rounds
ties away from zero; HalfEven rounds ties to the even retained digit.
Fixed decimal output scales are 0..28; bigdecimal output scales are 0..1024.

Use `decimal::parse(text)` or `bigdecimal::parse(text)` for untrusted decimal
text. They return `Result<value, str>` and accept plain signed digits with an
optional decimal fraction, up to 4096 input characters. Exponents, whitespace,
NaN and infinity are rejected. Fixed decimal parsing rejects values that
cannot be represented exactly. `to_plain_string()` produces canonical decimal
text without scientific notation. Display formatting is a separate concern.

## Persist without implicit rounding

The generated codec converts native values to exact decimal text. PostgreSQL's
NUMERIC binding and decoder preserve the value without a float conversion.
The ORM checks the destination precision and scale before a write: if the value
would require rounding, quantize it explicitly first.

```zolo
let amount = input.quantize(2, .HalfEven)?
Invoice::changes(amount: amount).apply(db, invoice_id)?
```

Use `apply_if_version` for a versioned model. The current encoding contract
reports an invalid value with the same explicit encoding failure as other
codecs; parsing and quantization return Result before model construction.

SQLite refuses numeric-decimal model execution because its NUMERIC affinity
can convert decimal text to binary floating point. To persist exact values as
text, opt in explicitly:

```zolo
@decimal(storage: .Text)
amount: decimal,
```

The field still decodes to `decimal`. Text storage deliberately does not offer
SQL numeric ordering, arithmetic or aggregation. MySQL's exact scalar binding
does not imply complete ORM RETURNING/upsert/migration support.

## Numeric SQL expressions retain their type

```zolo
let tax = Invoice::query()
  .select(|invoice| invoice.amount * 0.075d).all(db)?
let total = Invoice::query()
  .select(|invoice| Sql::sum(invoice.amount)).first_or_error(db)?
let positive = Invoice::query().filter(|invoice| invoice.amount > 0.005d)
```

Query operands use exact encoding independently of the column's storage scale.
A scale-2 column does not require rounding a tax rate or a comparison threshold.
Writes still check the destination scale.

Numeric storage supports order/range queries, grouping, captured `+`, `-` and `*`, and
SUM/MIN/MAX. Results keep the native logical type, nullable when SQL can return
NULL. A decimal aggregate that exceeds `decimal`'s representable range reports a
decode error; use a bigdecimal model when such totals are expected.

Integer quantities can participate in arithmetic. Float operands and implicit
mixing of decimal/bigdecimal or different domain codecs are rejected. Standalone
decimal literal projections without a model field decoder are not captured.
Decimal division/AVG and computed decimal UPDATE remain excluded until capture
has an explicit SQL precision/rounding contract.

The [manual PostgreSQL fixture](../examples/postgres_erp.zolo) records exact
roundtrip, aggregate, tax-rate and locking expectations without connecting
automatically. It awaits compilation and disposable-server execution.

## Compatibility with existing ExactDecimal models

`ExactDecimal` and `DecimalCodec` remain available for existing models.
Their default is still exact textual storage. Arithmetic compatibility methods
now delegate to the native `bigdecimal` implementation; the ORM no longer has
its own decimal arithmetic engine.

```zolo
let old = ExactDecimal::parse("12.34")?
let native = old.to_decimal()?
let wide = old.to_bigdecimal()?
let compatible = ExactDecimal::from_bigdecimal(wide + 0.01bd)
```

`NativeDecimalCodec` and `BigDecimalCodec` are public for custom adapters, but
ordinary native fields select them automatically. New models should use the
language types directly.
