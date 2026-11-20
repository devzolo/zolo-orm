# Built-in codecs

Use native `decimal` and `bigdecimal` directly in models; their numeric codecs
are selected automatically. See [financial values](financial-values.md).

Import compatibility domains and other codecs from `orm`. Dates, UTC timestamps
and the legacy ExactDecimal wrapper use
canonical text by default; blobs use real binary SQL values. Exact decimals can
opt into server numeric storage with `@decimal(storage: .Decimal)` or
`@decimal(precision: p, scale: s)`; see [financial values](financial-values.md).

| Domain | Constructor | Codec | Storage |
| --- | --- | --- | --- |
| `Date` | `Date::parse("2024-02-29")` | `DateCodec` | `.Text` |
| `Timestamp` | `Timestamp::parse("2024-02-29T12:30:00.1Z")` | `TimestampCodec` | `.Text` |
| `decimal` | `19.99d` / `decimal::parse(text)` | Automatic | `.Decimal` (default) / `.Text` |
| `bigdecimal` | `19.99bd` / `bigdecimal::parse(text)` | Automatic | `.Decimal` (default) / `.Text` |
| `ExactDecimal` | `ExactDecimal::parse("9007199254740993.01")` | `DecimalCodec` | `.Text` |
| `Blob` | `Blob::from_bytes([0, 255, 128])` | `BlobCodec` | `.Binary` |

```zolo
use orm::{Model, Date, DateCodec, ExactDecimal, DecimalCodec, Blob, BlobCodec}

@table("receipts")
struct Receipt deriving Model {
  @key
  id: int,
  @codec(DateCodec)
  day: Date,
  @codec(DecimalCodec)
  amount: ExactDecimal,
  @codec(BlobCodec, storage: .Binary)
  attachment: Blob?,
}
```

The same codec handles creates, saves, named changes, filters and projections.
A nullable field preserves SQL NULL; an empty blob is a present value with zero
bytes. Query comparisons encode domain values before binding them. Aggregate
arithmetic and ordering on text-encoded domain fields are rejected: text ordering
does not define decimal numeric order. Opting into `.Decimal` enables exact
numeric comparisons, ordering, SUM/MIN/MAX and supported SQL arithmetic while
retaining the domain decoder.

## Text formats and validation

`Date` accepts Gregorian dates from year 0001 through 9999, including the
century leap-year rules. It stores exactly `YYYY-MM-DD`.

`Timestamp` accepts UTC text ending in `Z`, with no fraction or one through
nine fractional digits. It stores `YYYY-MM-DDTHH:MM:SS.NNNNNNNNNZ`. Offsets,
leap-second 60 and excess precision are rejected. The parser does not convert
timezones or round values.

`ExactDecimal` accepts an optional minus sign, digits, and an optional decimal point
followed by digits. Precision is limited by available memory, with no conversion
through floating point. Canonicalization removes leading integer zeros and
trailing fractional zeros: `"00012.3400"` becomes `"12.34"`, and
`"-0.00"` becomes `"0"`. Exponents, plus signs, whitespace, NaN and
infinity are rejected. This domain preserves numeric value, not a fixed monetary
scale; retain scale separately when presentation requires it.

Use `parse` for the text domains. It returns `Result<Domain, str>`, so invalid
input can be handled before constructing a model. The language also exposes the
low-level newtype constructor `.new`, which bypasses validation. Each encoder
revalidates its input: an invalid wrapper constructed that way panics during
encoding rather than persisting malformed data. Decoding corrupt stored text
returns the ORM's contextual field error.

## Binary values

`Blob` is the nominal wrapper exported by `std::database`. Its payload is
opaque `bytes`, not a string or base64 text. `Blob::from_bytes` accepts
integer values 0 through 255 and returns `Result<Blob, DbError>`. It copies
the input. `blob.bytes()` returns a fresh array; changing either array cannot
mutate the blob.

For direct SQL bindings, `blob.unwrap()` exposes the raw binary value.
`Database::bytes_from_array`, `Database::byte_array`, and
`Database::is_bytes` provide the lower-level constructor, copy and type check.
ASCII-only blobs remain binary, and arbitrary bytes including NUL and invalid
UTF-8 round-trip without conversion.

The database bridge supports binary values on the VM, browser VM, native and
LLVM paths. Database access still requires an enabled driver. The standalone
wasm-AOT backend rejects database host calls because it has no database host
adapter.

See [the executable example](../examples/builtin_codecs.zolo) for precision,
nullable blobs, projections, named changes, cursor scope matching and decode
errors.
