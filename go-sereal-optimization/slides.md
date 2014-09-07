class: center, middle, inverse

# Optimize Sereal
### September Go meetup, Amsterdam
### Ivan Kruglov

.footnote[
  created with [remark](http://github.com/gnab/remark)
]

---
## Overview

- What Sereal?
- Brief explanation of how Sereal works
- Dig into Go implementation
- Go though made optimizations

---
## Results

- Marshaller

```terminal
    // before
    sereal-go $ go run psrl.go -folder data/ > data_go.srl
    ...snip...
*    The call took 3.001318514s to run.

    // after
    sereal-go $ go run psrl.go -folder data/ -expsize 52428800 > data_go.srl
    ...snip...
*    The call took 708.295852ms to run.
```

.footnote[.small[
    https://github.com/ikruglov/junk/blob/master/sereal-go/psrl.go
]]

---
## Results

- Unmarshaller

```terminal
    // before
    sereal-go $ go run dec.go -file data_go.srl
    ...snip...
*    Deserialization took 5.542141726s to run.

    // after
    sereal-go $ go run dec.go -file data_go.srl
    ...snip...
*    Deserialization took 1.592267889s to run.
```

.footnote[.small[
    https://github.com/ikruglov/junk/blob/master/sereal-go/dec.go
]]

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

^magic-str   ^version  ^no-options ^SHORT_BINARY_3 + content
\-------------header-------------\ \---------body----------\
```

---
## Example #2

```terminal
perl -MSereal -e 'print encode_sereal({ foo => 10 })' | hexdump -C
3d f3 72 6c 03 00 28 2a 01 63 66 6f 6f 0a
                 /                       \
   --------------                         ------ 
  /                                             \
28    2a     01      63 66 6f 6f                0a

^REFN ^HASH  ^length ^SHORT_BINARY_3 + content  ^POS_10
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
## Example #3

.small[
```terminal
perl -MSereal -e 'print encode_sereal(["foobar", "foobar"], { dedupe_strings => 1 })' | hexdump -C
```
]

```terminal
3d f3 72 6c 03 00 42 66 66 6f 6f 62 61 72 2f 02
                 /                            /
  ---------------                            |
 /                                           |
42           66 66 6f 6f 62 61 72       2f 02
^ARRAYREF_2  ^SHORT_BINARY_6 + content  ^COPY + offset
                                                    |
             ^-----------------here------------------
```

---
## Where Booking.com uses Sereal

- For transfering data between various services

- Main example: event stream
    - any action can send message - event
    - a message contains various information
    - events are received and processed by set of clients
    - flow of uncompressed events is about 50MB per sec

# 

--

.center[.red[Test dataset - one second of events]]

---
## Go implementation

- initially written by Damian Gryski (https://github.com/dgryski)

--

    - this guy is crazy, he has 1-year-long-streak on github

--

- encoder a.k.a marshaller
```go
func Marshal(v interface{}) ([]byte, error)
```

- decoder a.k.a unmarshaller
```go
func Unmarshal(b []byte, v interface{}) error 
```

---
## Marshaller (before)

.medium[
```go
func (e *Encoder) encode(b []byte, rv reflect.Value) ([]byte, error) {
    switch rk := rv.Kind(); rk {

    case reflect.Bool:
        b = e.encodeBool(b, rv.Bool())

    case reflect.Int:
        b = e.encodeInt(b, rv.Int())

    case reflect.Uint:
        b = e.encodeInt(b, rv.Uint())

    case reflect.String:
        b = e.encodeString(b, rv.String())

    case reflect.Slice:
        b = e.encodeArray(b, rv)

    case reflect.Map:
        b = e.encodeMap(b, rv)

    case reflect.Struct:
        b = e.encodeStruct(b, rv)

    case reflect.Float32:
        b = e.encodeFloat(b, rv.Float())

    case reflect.Float64:
        b = e.encodeDouble(b, rv.Float())
    
    ...
    }
}
```
]

---
## Marshaller profile (before)

.small[
```go
(pprof) top30
Total: 3035 samples
     357  11.8%  11.8%      357  11.8% scanblock
     231   7.6%  19.4%      231   7.6% runtime.aeshashbody
     189   6.2%  25.6%      201   6.6% runtime.MSpan_Sweep
     167   5.5%  31.1%     2396  78.9% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encode
     158   5.2%  36.3%      602  19.8% runtime.mallocgc
     117   3.9%  40.2%      119   3.9% settype
     113   3.7%  43.9%      118   3.9% hash_next
     111   3.7%  47.5%      120   4.0% itab
      94   3.1%  50.6%       94   3.1% flushptrbuf
      94   3.1%  53.7%       94   3.1% runtime.memmove
      80   2.6%  56.4%       80   2.6% reflect.memmove
      78   2.6%  58.9%      109   3.6% reflect.valueInterface
      75   2.5%  61.4%      434  14.3% cnew
      68   2.2%  63.7%      287   9.5% runtime.mapaccess2_faststr
      66   2.2%  65.8%      107   3.5% hash_lookup
      66   2.2%  68.0%       66   2.2% markonly
      58   1.9%  69.9%       70   2.3% reflect.Value.Bytes
      54   1.8%  71.7%      581  19.1% reflect.Value.MapIndex
      52   1.7%  73.4%       52   1.7% runtime.gettype
      50   1.6%  75.1%     2394  78.9% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeMap
      50   1.6%  76.7%       50   1.6% reflect.Value.Len
      47   1.5%  78.3%       47   1.5% runtime.markscan
      43   1.4%  79.7%       62   2.0% github.com/Sereal/Sereal/Go/sereal.varint
      38   1.3%  80.9%       38   1.3% runtime.memclr
      37   1.2%  82.1%      385  12.7% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeString
      34   1.1%  83.3%       34   1.1% reflect.Value.Kind
      30   1.0%  84.3%      527  17.4% reflect.Value.MapKeys
      28   0.9%  85.2%       28   0.9% reflect.Value.Index
      25   0.8%  86.0%       25   0.8% runtime.memeqbody
      24   0.8%  86.8%       46   1.5% reflect.Value.Elem
```
]

--

.red[
86.8% - 1.2% - 1.4% - 1.6% - 5.5% = ~77%
]

---
class: center, middle 

## WTF is the library doing 3/4 of the time?
![](wtf.jpg)

---
## Marshaller profile (before), continued

- memory operations: ~30%

- reflect: ~14%

- hash-related things: ~13%

- others: ~15%

# 

.small[
 - [profile graph](encoder-graphs-before.svg)
]

---
## Guilty function

.medium[
```go
func (e *Encoder) encodeMap(by []byte, m reflect.Value) []byte {
    keys := m.MapKeys()

    l := len(keys)
    by = append(by, typeHASH)
    by = varint(by, uint(l))

    for _, k := range keys {
        by, _ = e.encode(by, k)
        v := m.MapIndex(k)
        by, _ = e.encode(by, v)
    }

    return by
}
```
]

---
## Optimization concept

- Reflection is quite expensive and generates tons of garbage

--

- Let's get rid of reflection

--

- But I can't

--

- Okay, let's not use it as much as possible

--

# 

- Solution: .red[create set of shortcuts for common cases]

---
## Marshaller (after)

.medium[
```go
func (e *Encoder) encode(b []byte, v interface{}) ([]byte, error) {
    switch value := v.(type) {

    case nil:
        b = append(b, typeUNDEF)
    case int:
        b = e.encodeInt(...)
    case float32:
        b = e.encodeFloat(...)
    case string:
        b = e.encodeString(...)
    case []uint8:
        b = e.encodeBytes(...)
    case []interface{}:
        b, err = e.encodeIntfArray(...)
    case map[string]interface{}:
        b, err = e.encodeStrMap(...)

    ...

    default:
        b, err = e.encodeViaReflection(b, reflect.ValueOf(value)) // i.e. old logic
    }
}
```
]

---
## Marshaller (after), continued

.medium[
```go
func (e *Encoder) encodeStrMap(by []byte, m map[string]interface{}) ([]byte, error) {
    by = append(by, typeHASH)
    by = varint(by, uint(len(m)))

    var err error
    for k, v := range m {
        by = e.encodeString(by, k, true, strTable)
        if by, err = e.encode(by, v); err != nil {
            return by, err
        }
    }

    return by, nil
}

func (e *Encoder) encodeMap(by []byte, m reflect.Value) []byte {
    keys := m.MapKeys()

    l := len(keys)
    by = append(by, typeHASH)
    by = varint(by, uint(l))

    for _, k := range keys {
        by, _ = e.encode(by, k)
        v := m.MapIndex(k)
        by, _ = e.encode(by, v)
    }

    return by
}
```
]

---
## Marshaller profile (after)

.small[
```go
(pprof) top30
Total: 3003 samples
     447  14.9%  14.9%      447  14.9% scanblock
     274   9.1%  24.0%      274   9.1% hash_next
     250   8.3%  32.3%     2203  73.4% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeStrMap
     224   7.5%  39.8%     2204  73.4% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encode
     214   7.1%  46.9%      214   7.1% runtime.aeshashbody
     200   6.7%  53.6%      200   6.7% runtime.memmove
     177   5.9%  59.5%      457  15.2% runtime.mapaccess2_faststr
     147   4.9%  64.4%     2204  73.4% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeIntfArray
     134   4.5%  68.8%      134   4.5% flushptrbuf
     122   4.1%  72.9%      122   4.1% runtime.slicecopy
     105   3.5%  76.4%      139   4.6% github.com/Sereal/Sereal/Go/sereal.varint
      82   2.7%  79.1%       82   2.7% markonly
      77   2.6%  81.7%      630  21.0% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeString
      73   2.4%  84.1%      119   4.0% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeInt
      73   2.4%  86.5%       74   2.5% runtime.MSpan_Sweep
      61   2.0%  88.6%       61   2.0% runtime.efacethash
      56   1.9%  90.4%       56   1.9% runtime.memeqbody
      53   1.8%  92.2%      209   7.0% runtime.assertE2T2
      52   1.7%  93.9%       52   1.7% runtime.gettype
      36   1.2%  95.1%       36   1.2% runtime.memclr
      26   0.9%  96.0%      226   7.5% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeBytes
      24   0.8%  96.8%      156   5.2% copyout
      18   0.6%  97.4%       18   0.6% runtime.MHeap_LookupMaybe
      14   0.5%  97.9%       21   0.7% hash_iter_init
      10   0.3%  98.2%      186   6.2% runtime.mapiternext
       7   0.2%  98.4%        7   0.2% runtime.fastrand1
       7   0.2%  98.7%        7   0.2% runtime.memcopy64
       7   0.2%  98.9%        7   0.2% runtime.memeq
       5   0.2%  99.1%        5   0.2% runtime.aeshashstr
       4   0.1%  99.2%        4   0.1% runtime.duffzero
```

- [profile graph](encoder-graphs-after.svg)

]

---
## Marshaller profile (after)

.small[
```go
(pprof) top30
Total: 3003 samples
*     447  14.9%  14.9%      447  14.9% scanblock
*     274   9.1%  24.0%      274   9.1% hash_next
     250   8.3%  32.3%     2203  73.4% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeStrMap
     224   7.5%  39.8%     2204  73.4% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encode
*     214   7.1%  46.9%      214   7.1% runtime.aeshashbody
*     200   6.7%  53.6%      200   6.7% runtime.memmove
*     177   5.9%  59.5%      457  15.2% runtime.mapaccess2_faststr
     147   4.9%  64.4%     2204  73.4% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeIntfArray
*     134   4.5%  68.8%      134   4.5% flushptrbuf
*     122   4.1%  72.9%      122   4.1% runtime.slicecopy
     105   3.5%  76.4%      139   4.6% github.com/Sereal/Sereal/Go/sereal.varint
      82   2.7%  79.1%       82   2.7% markonly
      77   2.6%  81.7%      630  21.0% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeString
      73   2.4%  84.1%      119   4.0% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeInt
      73   2.4%  86.5%       74   2.5% runtime.MSpan_Sweep
      61   2.0%  88.6%       61   2.0% runtime.efacethash
      56   1.9%  90.4%       56   1.9% runtime.memeqbody
      53   1.8%  92.2%      209   7.0% runtime.assertE2T2
      52   1.7%  93.9%       52   1.7% runtime.gettype
      36   1.2%  95.1%       36   1.2% runtime.memclr
      26   0.9%  96.0%      226   7.5% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeBytes
      24   0.8%  96.8%      156   5.2% copyout
      18   0.6%  97.4%       18   0.6% runtime.MHeap_LookupMaybe
      14   0.5%  97.9%       21   0.7% hash_iter_init
      10   0.3%  98.2%      186   6.2% runtime.mapiternext
       7   0.2%  98.4%        7   0.2% runtime.fastrand1
       7   0.2%  98.7%        7   0.2% runtime.memcopy64
       7   0.2%  98.9%        7   0.2% runtime.memeq
       5   0.2%  99.1%        5   0.2% runtime.aeshashstr
       4   0.1%  99.2%        4   0.1% runtime.duffzero
```

- [profile graph](encoder-graphs-after.svg)

]

---
## Marshaller profile (preallocated memory)

.small[
```go
(pprof) top30
Total: 3004 samples
     357  11.9%  11.9%      357  11.9% hash_next
     307  10.2%  22.1%     2604  86.7% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encode
     293   9.8%  31.9%     2599  86.5% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeStrMap
     285   9.5%  41.3%      285   9.5% runtime.aeshashbody
     195   6.5%  47.8%      569  18.9% runtime.mapaccess2_faststr
*     179   6.0%  53.8%      179   6.0% scanblock
     166   5.5%  59.3%     2604  86.7% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeIntfArray
     158   5.3%  64.6%      158   5.3% github.com/Sereal/Sereal/Go/sereal.varint
     156   5.2%  69.8%      156   5.2% runtime.slicecopy
*     117   3.9%  73.7%      117   3.9% runtime.memmove
     108   3.6%  77.3%      807  26.9% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeString
      82   2.7%  80.0%      282   9.4% runtime.assertE2T2
      77   2.6%  82.6%       77   2.6% runtime.memeqbody
      74   2.5%  85.0%      136   4.5% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeInt
      74   2.5%  87.5%       74   2.5% runtime.efacethash
      48   1.6%  89.1%       48   1.6% runtime.memclr
      38   1.3%  90.3%       38   1.3% markonly
      37   1.2%  91.6%       37   1.2% flushptrbuf
      36   1.2%  92.8%      200   6.7% copyout
      32   1.1%  93.8%       32   1.1% runtime.MSpan_Sweep
      29   1.0%  94.8%       86   2.9% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeBytes
      24   0.8%  95.6%       24   0.8% runtime.gettype
      23   0.8%  96.4%       28   0.9% hash_iter_init
      16   0.5%  96.9%      257   8.6% runtime.mapiternext
      13   0.4%  97.3%       13   0.4% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeDouble
      12   0.4%  97.7%       12   0.4% runtime.MHeap_LookupMaybe
      10   0.3%  98.1%       10   0.3% runtime.duffzero
      10   0.3%  98.4%       10   0.3% runtime.memeq
       9   0.3%  98.7%        9   0.3% runtime.aeshashstr
       9   0.3%  99.0%      153   5.1% runtime.mapiterinit
```

- [profile graph](encoder-graphs-after2.svg)

]

---
## Hash access optimization

.medium[
```go
    // var byt []byte
    str := string(byt)

    if copyOffs, ok := strTable[str]; ok {
        by = append(by, typeCOPY)
        by = varint(by, uint(copyOffs))
        return by
    } else {
        strTable[str] = currentOffset
    }
```
]

.footnote[.small[
    https://github.com/dgryski/trifles/tree/master/strtable
]]

--

# 

dgryski made attempt to create custom hash table with insert function:

.medium[
```go
func (t *Table) Insert(k []byte, val uint32) (uint32, bool) {}
```
]

--

# 

shows 10% of improvement during benchmarking,
 
but negligible boost on real data set

--

# 

.red[FAILED]

---
## Hash access optimization, continued

.medium[
```go
    // var byt []byte
    str := string(byt)

    if copyOffs, ok := strTable[str]; ok {
        by = append(by, typeCOPY)
        by = varint(by, uint(copyOffs))
        return by
    } else {
        strTable[str] = currentOffset
    }
```
]

--

In Go 1.3 this access pattern was optimized

--

.medium[
```go
    // var byt []byte
    if copyOffs, ok := strTable[string(str)]; ok {
        by = append(by, typeCOPY)
        by = varint(by, uint(copyOffs))
        return by
    } else {
        strTable[string(str)] = currentOffset
    }
```
]

---
## Marshaller profile (conclusion)

.small[
```go
(pprof) top10
Total: 3004 samples
     357  11.9%  11.9%      357  11.9% hash_next
     307  10.2%  22.1%     2604  86.7% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encode
     293   9.8%  31.9%     2599  86.5% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeStrMap
     285   9.5%  41.3%      285   9.5% runtime.aeshashbody
     195   6.5%  47.8%      569  18.9% runtime.mapaccess2_faststr
     179   6.0%  53.8%      179   6.0% scanblock
     166   5.5%  59.3%     2604  86.7% github.com/Sereal/Sereal/Go/sereal.(*Encoder).encodeIntfArray
     158   5.3%  64.6%      158   5.3% github.com/Sereal/Sereal/Go/sereal.varint
     156   5.2%  69.8%      156   5.2% runtime.slicecopy
     117   3.9%  73.7%      117   3.9% runtime.memmove
```
]

.medium[
- hash_next - loop over hash keys
- runtime.aeshashbody, runtime.mapaccess2_faststr - string deduplication
- scanblock - GC
- runtime.slicecopy, runtime.memmove - copy data
]

---
## Results once again

- Marshaller

.medium[
```terminal
    // old code
    sereal-go $ go run psrl.go -folder data/ > data_go.srl
    ...snip...
    The call took 3.001318514s to run.

    // new code
    sereal-go $ go run psrl.go -folder data/ > data_go.srl
    ...snip...
    The call took 1.199459481 to run.

    // new code with preallocated memory
    sereal-go $ go run psrl.go -folder data/ -expsize 52428800 > data_go.srl
    ...snip...
    The call took 708.295852ms to run.
```
]

---
## Unmarshaller

- Same concept

- Create shortcuts for common cases

- Avoid reflection, try using direct assigments

.medium[
```go
    var iface interface{}
    iface = "string"
```
faster then
```go
    var iface interface{}
    rv := reflect.ValueOf(iface)
    rv.Set(reflect.ValueOf("string"))
```
]

# 

.footnote[.small[
    https://github.com/ikruglov/junk/blob/master/go-bench-inf-vs-refl/benchmark_test.go
]]

---
## Results once again

- Unmarshaller

.medium[
```terminal
    // old code
    sereal-go $ go run dec.go -file data_go.srl
    ...snip...
    Deserialization took 5.542141726s to run.

    // new code
    sereal-go $ go run dec.go -file data_go.srl
    ...snip...
    Deserialization took 1.592267889s to run.
```
]

---
## Some magic ...

.medium[
```go
commit c562af4
Author: Ivan Kruglov <ivan.kruglov@booking.com>
Date:   2014-08-18 00:57:09 +0200

    Go decode: this tiny changes makes ????% of speed improvment

diff --git a/Go/sereal/decode.go b/Go/sereal/decode.go
index d0bf6c4..7972525 100644
--- a/Go/sereal/decode.go
+++ b/Go/sereal/decode.go
@@ -431,7 +431,6 @@ func (d *Decoder) decodeHash(by []byte, idx int, ln int, ptr *interface{}, isRef
                return 0, ErrCorrupt{errBadHashSize}
        }

*+       var key []byte
*+       var value interface{}
+
        for i := 0; i < ln; i++ {
*-               var key []byte
                key, idx, err = d.decodeStringish(by, idx)
                if err != nil {
                        return 0, err
                }

*-               var value interface{}
                idx, err = d.decode(by, idx, &value)
                if err != nil {
                        return 0, err

```
]

--

.center[.red[
    ~10% of speed improvement
]]

---
## ... which is not easily revealed by microbenchmarks

.medium[
```go
func BenchmarkDeclareInterfaceInside(b *testing.B) {
    hash := make(map[string]interface{}, b.N)

    for i := 0; i < b.N; i++ {
        var iface interface{}
        iface = i
        hash[strconv.Itoa(i)] = iface
    }
}

func BenchmarkDeclareInterfaceOutside(b *testing.B) {
    hash := make(map[string]interface{}, b.N)
    var iface interface{}

    for i := 0; i < b.N; i++ {
        iface = i
        hash[strconv.Itoa(i)] = iface
    }
}
```
]

.medium[
```terminal
$ go test -bench=.
testing: warning: no tests to run
PASS
BenchmarkDeclareInterfaceInside         5000000               288 ns/op
BenchmarkDeclareInterfaceOutside        10000000              290 ns/op
ok      github.com/ikruglov/junk/go-bench-loop-inf      5.035s
```
]

.footnote[.small[
    https://github.com/ikruglov/junk/blob/master/go-bench-loop-inf/benchmark_test.go
]]

---
## Snappy/zlib pure Go vs pure C

- Go is fast, but still 2-3x slower then C

- Consider using C libraries for CPU intensive oprations

--

.medium[
```terminal
$ go test -bench=.
testing: warning: no tests to run
PASS
BenchmarkCSnappy              10         126865617 ns/op
BenchmarkGoSnappy              5         386799299 ns/op
ok      github.com/ikruglov/junk/go-bench-snappy-go-c   3.744s
```
]

--

- Sereal Go library gives ability to chose between pure Go and pure C compression libraries
 - go install
 - go install -tags clibs


---
## Conslusions

- Go is great!

- Profile code

- Don't stop when no obvious bottlenecks (apply common sense)

- Simplify life for compiler

- Try avoid using reflection

# 

- Work on Sereal is not done yet, plenty of topics to improve
    - mostly optimized for our dataset

---
## Comparasion of serialization libraries

.medium[
```terminal
$ go test -bench=.

PASS
BenchmarkUgorjiMsgpackMarshal           200000              5366 ns/op
BenchmarkUgorjiMsgpackUnmarshal         500000              4784 ns/op
BenchmarkVmihailencoMsgpackMarshal      1000000             2460 ns/op
BenchmarkVmihailencoMsgpackUnmarshal    500000              3284 ns/op
BenchmarkJsonMarshal                    500000              4745 ns/op
BenchmarkJsonUnmarshal                  200000              8034 ns/op
BenchmarkBsonMarshal                    1000000             2915 ns/op
BenchmarkBsonUnmarshal                  500000              3655 ns/op
BenchmarkVitessBsonMarshal              1000000             1822 ns/op
BenchmarkVitessBsonUnmarshal            1000000             1051 ns/op
BenchmarkGobMarshal                     200000              9484 ns/op
BenchmarkGobUnmarshal                   50000               69573 ns/op
BenchmarkXdrMarshal                     500000              3618 ns/op
BenchmarkXdrUnmarshal                   1000000             2712 ns/op
BenchmarkUgorjiCodecMsgpackMarshal      500000              5184 ns/op
BenchmarkUgorjiCodecMsgpackUnmarshal    500000              5377 ns/op
BenchmarkUgorjiCodecBincMarshal         500000              6904 ns/op
BenchmarkUgorjiCodecBincUnmarshal       500000              6901 ns/op
*BenchmarkSerealMarshal                  500000              6502 ns/op
*BenchmarkSerealUnmarshal                500000              5386 ns/op
BenchmarkBinaryMarshal                  500000              3056 ns/op
BenchmarkBinaryUnmarshal                1000000             3006 ns/op
ok      github.com/alecthomas/go_serialization_benchmarks       53.865s
```
]

.footnote[.small[
    https://github.com/alecthomas/go_serialization_benchmarks
]]

---
class: center, middle, inverse

# Thank you!
### https://github.com/Sereal/Sereal
### https://github.com/ikruglov

---

vim: ft=markdown
