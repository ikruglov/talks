class: center, middle, inverse

# Sereal: a view from inside
### Ivan Kruglov

.footnote[
  created with [remark](http://github.com/gnab/remark)
]

---
## What is Sereal?

- Fast, compact, schema-less, binary serialization and deserialization oriented towards dynamic languages

- Initially written for Perl in perlxs (interface between C and Perl)

- Currently, there are implementations for Perl, Go, Python, objC, Ruby, Java

- Features:
    - support of shared references
    - support of weak references
    - support of aliases
    - support of perl regular expresions
    - space efficiency 
    - speed efficiency 
    - others

.small[
    https://github.com/Sereal/Sereal
]
.small[
    https://github.com/Sereal/Sereal/wiki/Sereal-Comparison-Graphs
]

---
## Sereal specification (cut)

.medium[
```
           Tag |  Hex | Follow
---------------+------+------------------------------------------
POS_0          | 0x00 | small positive integer - value in low 4 bits (identity)
POS_1          | 0x01 | 
NEG_16         | 0x10 | small negative integer - value in low 4 bits (k+32)
NEG_15         | 0x11 |
VARINT         | 0x20 | <VARINT> - Varint variable length integer
FLOAT          | 0x22 | <IEEE-FLOAT>
UNDEF          | 0x25 | None - Perl undef var; eg my $var= undef;
BINARY         | 0x26 | <LEN-VARINT> <BYTES> - binary/(latin1) string
REFN           | 0x28 | <ITEM-TAG> - ref to next item
REFP           | 0x29 | <OFFSET-VARINT> - ref to previous item stored at offset
HASH           | 0x2a | <COUNT-VARINT> [<KEY-TAG> <ITEM-TAG> ...] - count followed by key/value pairs
ARRAY          | 0x2b | <COUNT-VARINT> [<ITEM-TAG> ...] - count followed by items
OBJECT         | 0x2c | <STR-TAG> <ITEM-TAG> - class, object-item
ALIAS          | 0x2e | <OFFSET-VARINT> - alias to item defined at offset
COPY           | 0x2f | <OFFSET-VARINT> - copy of item defined at offset
REGEXP         | 0x31 | <PATTERN-STR-TAG> <MODIFIERS-STR-TAG>
FALSE          | 0x3a | false (PL_sv_no)
TRUE           | 0x3b | true  (PL_sv_yes)
ARRAYREF_0     | 0x40 | [<ITEM-TAG> ...] - count of items in low 4 bits (ARRAY must be refcnt=1)
ARRAYREF_1     | 0x41 |
ARRAYREF_2     | 0x42 |
HASHREF_0      | 0x50 | [<KEY-TAG> <ITEM-TAG> ...] - count in low 4 bits, key/value pairs (HASH must be refcnt=1)
HASHREF_1      | 0x51 |
HASHREF_2      | 0x52 |
SHORT_BINARY_0 | 0x60 | <BYTES> - binary/latin1 string, length encoded in low 5 bits of tag
SHORT_BINARY_1 | 0x61 |
SHORT_BINARY_2 | 0x62 |
```
]

.footnote[.small[
    https://github.com/Sereal/Sereal/blob/master/sereal_spec.pod
]]

---
## Example #1

```terminal
perl -MSereal -e 'print encode_sereal("foo")' | hexdump -C
3d f3 72 6c 03 00 63 66 6f 6f
```

--

```terminal
3d f3 72 6c  03        00          63 66 6f 6f

^magic-str   ^version  ^no-options ^SHORT_BINARY_3 + content(foo)
\-------------header-------------\ \------------body------------\
```

--

- magic string

 - v1, v2 - '=srl'

 - v3 - '=\xF3rl' ('=srl' with set higest bif of 's')

---
## Example #2

```terminal
perl -MSereal -e 'print encode_sereal({ foo => 10 })' | hexdump -C
3d f3 72 6c 03 00 51 63 66 6f 6f 0a
                 /                 \
 ----------------                   ------------------------
/                                                           \
51                63              66 6f 6f                 0a

^HASHREF_1   key: ^SHORT_BINARY_3 ^content(foo)     value: ^POS_10
```

---
## Example #3
```terminal
perl -MSereal -e 'print encode_sereal([(1) x 32])' | hexdump -C
3d f3 72 6c 03 00 28 2b 20 01 01 ... 01
                 /                     \
 ----------------                       ---------------------
/                                                            \
28     2b               20                        01 01 ... 01

^REFN  ^ARRAY   length: ^VARINT(32)  items x len: ^POS_1

```

---
## Sereal specification (cut)

.medium[
```
           Tag |  Hex | Follow
---------------+------+------------------------------------------
POS_0          | 0x00 | small positive integer - value in low 4 bits (identity)
POS_1          | 0x01 | 
NEG_16         | 0x10 | small negative integer - value in low 4 bits (k+32)
NEG_15         | 0x11 |
VARINT         | 0x20 | <VARINT> - Varint variable length integer
FLOAT          | 0x22 | <IEEE-FLOAT>
UNDEF          | 0x25 | None - Perl undef var; eg my $var= undef;
BINARY         | 0x26 | <LEN-VARINT> <BYTES> - binary/(latin1) string
REFN           | 0x28 | <ITEM-TAG> - ref to next item
*REFP           | 0x29 | <OFFSET-VARINT> - ref to previous item stored at offset
HASH           | 0x2a | <COUNT-VARINT> [<KEY-TAG> <ITEM-TAG> ...] - count followed by key/value pairs
ARRAY          | 0x2b | <COUNT-VARINT> [<ITEM-TAG> ...] - count followed by items
OBJECT         | 0x2c | <STR-TAG> <ITEM-TAG> - class, object-item
ALIAS          | 0x2e | <OFFSET-VARINT> - alias to item defined at offset
*COPY           | 0x2f | <OFFSET-VARINT> - copy of item defined at offset
REGEXP         | 0x31 | <PATTERN-STR-TAG> <MODIFIERS-STR-TAG>
FALSE          | 0x3a | false (PL_sv_no)
TRUE           | 0x3b | true  (PL_sv_yes)
ARRAYREF_0     | 0x40 | [<ITEM-TAG> ...] - count of items in low 4 bits (ARRAY must be refcnt=1)
ARRAYREF_1     | 0x41 |
ARRAYREF_2     | 0x42 |
HASHREF_0      | 0x50 | [<KEY-TAG> <ITEM-TAG> ...] - count in low 4 bits, key/value pairs (HASH must be refcnt=1)
HASHREF_1      | 0x51 |
HASHREF_2      | 0x52 |
SHORT_BINARY_0 | 0x60 | <BYTES> - binary/latin1 string, length encoded in low 5 bits of tag
SHORT_BINARY_1 | 0x61 |
SHORT_BINARY_2 | 0x62 |
```
]

.footnote[.small[
    https://github.com/Sereal/Sereal/blob/master/sereal_spec.pod
]]

---
## Example #4

.small[
```terminal
perl -MSereal -e '%hash; $hash{rec} = \%hash; print encode_sereal(\%hash)' | hexdump -C
```
]

```terminal
28    aa           01                63 72 65 63     29 02

^REFN ^HASH | fag  ^VARINT(1)   key: ^rec     value: ^REFP
                                                         |
      ^---------------------offset 2----------------------
```

--

- REFP targets are flagged

---
## Example #5

.small[
```terminal
perl -MSereal -e 'print encode_sereal([{ foo => "bar" }, { foo => "bar" }])' | hexdump -C
```
]

```terminal
arrayref
   1st hashref                      2nd hashref
            key         value                key   value
42 28 2a 01 63 66 6f 6f 63 62 61 72 28 2a 01 2f 05 63 62 61 72
                                             
            ^                                ^COPY
            ^                                    |
            ^------------offset 5-----------------   
```

--

- COPY targets are not flagged

--

- key deduplication is cheap in Perl
 - table of strings' addresses (not strings) in shared key storage is kept

---
## Example #6

.small[
```terminal
perl -MSereal -e 'print encode_sereal(["foobar", "foobar"], { dedupe_strings => 1 })' | hexdump -C
```
]

```terminal
arrayref
   item 1               item 2
42 66 66 6f 6f 62 61 72 2f 02

   ^                    ^COPY
   ^                        |
   ^--------offset 2---------
```

--
- dedupe_strings doesn't work for small strings (< 3)

---
## Serialization of objects

- OBJECT
 - class name and a tag which represents the objects data 
 - OBJECTV - is a shortcut to omit dumping class name

- OBJECT_FREEZE
 - call FREEZE method upon serialization
 - call THAW upon deserialization
 - OBJECTV_FREEZE - again just a shortcut

--
# 

.small[
```terminal
perl -MSereal -e 'print encode_sereal(bless({ foo => "bar" }, "obj"))' | hexdump -C
```
]

```terminal
OBJECT
   classname   hashref
2c 63 6f 62 6a 51 63 66 6f 6f 63 62 61 72
```

--
- class name deduplication is also cheap

---
## Compression and headers

- Compression
 - Snappy and Zlib
 - encoded in version tag
 - only body is compressed

- Sereal v2 adds headers
 - independent Sereal document
 - supposed to be small (i.e. meta 
 - never compressed

---
## A bit of internals of Sereal in perl

- highly optimized

- use custom hash tables

- use a lot of shortcuts

- use branch prediction hints

- cache-friendly

---
## Sereal vs other serialization libraries

[benchmark]

---
## Sereal vs other serialization libraries (recursion)
# 

.small[
```terminal
$ perl -MCBOR::XS -e '%hash; $hash{rec} = \%hash; print encode_cbor(\%hash)' | hexdump -C
cbor text or perl structure exceeds maximum nesting level (max_depth set too low?) at -e line 1.

perl -MJSON::XS -e '%hash; $hash{rec} = \%hash; print encode_json(\%hash)' | hexdump -C
json text or perl structure exceeds maximum nesting level (max_depth set too low?) at -e line 1.

perl -MData::MessagePack -e '%hash; $hash{rec} = \%hash; print Data::MessagePack->pack(\%hash)' | hexdump -C
perl structure exceeds maximum nesting level (max_depth set too low?) at -e line 1.

perl -MSereal -e '%hash; $hash{rec} = \%hash; print encode_sereal(\%hash)' | hexdump -C
3d f3 72 6c 03 00 28 aa  01 63 72 65 63 29 02
```
]

---
## Sereal vs other serialization libraries (regexp)
# 

.small[
```terminal
perl -MJSON::XS -e 'print encode_json(qr/123/)' | hexdump -C
encountered object '(?^:123)', but neither allow_blessed, convert_blessed nor allow_tags settings are enabled (or TO_JSON/FREEZE method missing) at -e line 1.

perl -MCBOR::XS -e 'print encode_cbor(qr/123/)' | hexdump -C
encountered object '(?^:123)', but no TO_CBOR or FREEZE methods available on it at -e line 1.

perl -MData::MessagePack -e 'print Data::MessagePack->pack(qr/123/)' | hexdump -C
encountered object '(?^:123)', Data::MessagePack doesn't allow the object at -e line 1.

perl -MSereal -e 'print encode_sereal(qr/123/)' | hexdump -C
3d f3 72 6c 03 00 2c 66 52 65 67 65 78 70 28 31 63 31 32 33 60
```
]

--

- no support of shared references
- no support of weak references
- no support of aliases
- no support of perl regular expresions

---
## Sereal in perl vs Sereal in other languages

- test dataset is one second of events (uncompressed)

```
        dec     enc
- perl  0.85s   0.40s
- go    1.86    0.85s
- ruby  2.4s    1.1s
```

---
## What's next in Sereal?

--

- Sereal::Merger

 - A, B, C => [ A, B, C ]
 
 - [A, B], [C , D] => [A, B, C, D]

 - { foo => bar }, { bar => foo } => { foo => bar, bar => foo }

--

- Filtering upon decoding

 - restore only part of encoded information

---
class: center, middle, inverse

# Thank you!
### https://github.com/Sereal/Sereal
### https://github.com/ikruglov

---

vim: ft=markdown
