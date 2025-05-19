# Data Types

Vortex wraps data in logical types, a so called `DType`, these types denote what the underlying data represents but separate from how the data is stored physically. The supported types are `Null`, `Bool`, `Primitive`, `UTF8`, `Binary`, `Struct`, `List` and `Extension` , the types are implemented as an enum of which the values may contain additional tuples with information.

```rust
pub enum DType {
    Null,
    Bool(Nullability),
    Primitive(PType, Nullability),
    Decimal(DecimalDType, Nullability),
    Utf8(Nullability),
    Binary(Nullability),
    Struct(Arc<StructDType>, Nullability),
    List(Arc<DType>, Nullability),
    Extension(Arc<ExtDType>),
}
```

### Nullability

All types except `Null`  support nullability by including the `Nullability` enum into the `DType`.

```rust
pub enum Nullability {
    #[default]
    NonNullable,
    Nullable,
}
```

### Primitive

The `Primitive` data type contains fixed-width types that cannot be reduced into other types. Supported `PType` variants are `I8`, `I16`, `I32`, `I64`, `U8`, `U16`, `U32`, `U64`, `F16`, `F32`, `F64`. It is of note that `F16` does not support SIMD instructions, which may lead to degraded performance.

{% hint style="warning" %}
Test the performance of F16.
{% endhint %}

### Decimal

The `Decimal` data type represents real numbers exactly. It is made up of a precision, which tracks the number of significant digits, and a scale, which shifts the digits relative to the decimal point. A max scale and precision is given of 76 for both. This comes down to a theoretical unscaled range of $$[-(10^{76}-1), (10^{76}-1)]$$ which requires $$\bigl\lceil 76\cdot\log_2(10)\bigr\rceil \approx 253 \text{bits}$$.

{% hint style="warning" %}
Encoding currently checks for `required_bit_width`  which depends on usize, check if DecimalDTypes can also be constructed without this constraint. usize = 32 -> p = 9, usize = 64 -> p = 19.
{% endhint %}

```rust
pub struct DecimalDType {
    precision: u8,
    // Negative: shift left, enlarging decimal.
    // Positive: shift right, finer fractions.
    scale: i8,
}
```

### UTF8

The `UTF8` data type represents variable length UTF-8 encoded strings.

### Binary

The `Binary` data type represents variable length bytes.

### Struct

The Struct data type represents an ordered collection of named fields, where each field can have its own `DType`. Named fields are constructed as `FieldNames` , which is defined as `Arc<[Arc<str>]>` .&#x20;

{% code fullWidth="false" %}
```rust
pub struct StructDType {
    names: FieldNames,
    dtypes: Arc<[FieldDType]>,
}
```
{% endcode %}

Since a struct can be of arbitrary complexity, support for lazy loading fields is implemented within `FieldDType` and `FieldDTypeInner` which distinguishes between `Owned` and `View` fields. Owned fields are heap-allocated whereas view fields reference a Flatbuffer representation and underlying data is only fetched when the related field is accessed.

```rust
pub struct FieldDType {
    inner: FieldDTypeInner,
}

enum FieldDTypeInner {
    Owned(DType),
    View(ViewedDType),
}
```

### List

The `List` data type represents a variable length list which contains data of a singular `DType` type.

### Extension

The `Extension` data type provides a mechanism for the introduction of custom data types. It allows a user to express semantic meaning on top of available `DType` values.

{% hint style="warning" %}
What are the performance implications of using Extension over the raw DType equivalent? And is there not a case to be made for support? What happens if a reader has no notion of a certain ExtDType?
{% endhint %}

```rust
struct ExtDType {
    id: ExtID,
    storage_dtype: Arc<DType>,
    metadata: Option<ExtMetadata>,
}
```
