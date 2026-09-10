The TypePHP compiler provides a syntax for calling object methods on native types, called Universal Methods (`Universal Methods`). Universal Methods are a zero-cost abstraction design that works only at the compilation stage, with no runtime overhead.


## 1. What Are Universal Methods

Universal Methods (`Universal Methods`) allow you to call methods directly on variables of PHP native types, just as you would on objects. The compiler translates these method calls at compile time into the corresponding C functions or C++ method calls, **generating zero-overhead native code**.

```php
// calling methods directly on strings
$s = "hello world";
echo $s->length();          // → strlen($s) → 11
echo $s->upper();           // → strtoupper($s) → "HELLO WORLD"
echo $s->substr(0, 5);      // → substr($s, 0, 5) → "hello"

// calling methods directly on arrays
$arr = [1, 3, 5, 7, 9];
echo $arr->count();         // → count($arr) → 5
echo $arr->contains(3);     // → in_array(3, $arr) → true
$arr->push(11);             // → array_push($arr, 11)

// calling methods directly on integers
$a = 100;
$a->add(50);                // → a += 50 → 150
echo $a->toString();        // → "150"

// calling methods directly on high-precision types
$big = std::bigInt("12345678901234567890");
echo $big->mul(2)->toString();  // → "24691357802469135780"
```

> **Key idea**: All universal method calls are fully resolved into direct function calls at compile time. There is no vtable lookup, reflection, or runtime type checking — the generated C++ code is exactly equivalent to hand-written C function calls.

---

## 2. Design Principles

### 2.1 Compile-Time Resolution

Each universal method call goes through the following steps at compile time:

1. **Type inference**: the compiler infers the type of the receiver (Int / Float / String / Array / Stream / BigInt / Decimal / BigFloat / Var)
2. **Method lookup**: look up the method definition in that type's method table
3. **Argument validation**: check that the number of arguments is within the `min_args` ~ `max_args` range
4. **Code generation**: generate the corresponding C/C++ call code based on the handler type

---

## 3. Overview of Supported Types

| Type | Number of methods | Main categories |
|------|---------|---------|
| **Int** | 26 | arithmetic, math functions, type conversion |
| **Float** | 26 | arithmetic, math functions, trigonometric functions, type conversion |
| **Bool** | 2 | type conversion |
| **String** | 70+ | string operations, search, encoding, hash, multibyte, serialization |
| **Array** | 50+ | CRUD, sorting, iteration, set operations, serialization |
| **Stream** | 30+ | read/write, positioning, locking, socket, filters |
| **BigInt** | 24 | arithmetic, comparison, conversion, GCD, bitwise, shift, divmod, powmod, sqrt |
| **Decimal** | 18 | arithmetic, comparison, conversion, pow, divmod, powmod, sqrt, floor, ceil, round |
| **BigFloat** | 10 | arithmetic, comparison, conversion |

> **Type conversion methods**: `toInt()` / `toFloat()` / `toString()` / `toBool()` / `toArray()` / `toStream()` / `toBigInt()` / `toBigFloat()` / `toDecimal()` / `toObject()` / `toStd*` and other keyword methods do not belong to Universal Methods. They are first-class citizens built into the compiler. See [Type Conversion](type-convert.md).

---

## 4. Int Methods

### 4.1 Arithmetic Operations (returning Int)

```php
$a = 100;

$a->add(50);        // $a + 50  → returns 150
$a->sub(30);        // $a - 30  → returns 70
$a->mul(2);         // $a * 2   → returns 200
$a->div(4);         // $a / 4   → returns 25
$a->mod(7);         // $a % 7   → returns 2

$a->inc();          // $a + 1  → returns 101
$a->dec();          // $a - 1  → returns 99
```

```php
$a = 100;
$b = $a->add(50);  // $a is still 100, $b is 150 (returns a new value, original unchanged)
```

### 4.2 Math Functions

```php
$a = -100;

$a->abs();          // abs($a)         → 100
$a->ceil();         // ceil($a)        → -100.0 (float)
$a->floor();        // floor($a)       → -100.0 (float)
$a->round();        // round($a)       → -100.0 (float)
$a->sqrt();         // sqrt($a)        → NaN (negative number has no square root)
$a->pow(3);         // $a ** 3         → -1000000
$a->log();          // log($a)         → NaN
$a->log10();        // log10($a)       → NaN
$a->exp();          // exp($a)         → 3.72e-44 (float)

$a->max(50);        // max($a, 50)     → 50
$a->min(50);        // min($a, 50)     → -100
```

> **Note**: `ceil`, `floor`, and `round` return the **Float type** (PHP standard behavior). `pow` returns the **Var type** (because exponentiation may overflow).

### 4.3 Trigonometric Functions

```php
$a = 0;

$a->sin();          // sin(0)     → 0.0
$a->cos();          // cos(0)     → 1.0
$a->tan();          // tan(0)     → 0.0
$a->asin();         // asin(0)    → 0.0
$a->acos();         // acos(1)    → 0.0
$a->atan();         // atan(0)    → 0.0
$a->atan2(1);       // atan2(0,1) → 0.0
$a->deg2rad();      // deg2rad(0) → 0.0
$a->rad2deg();      // rad2deg(0) → 0.0
```

### 4.4 Type Conversion

```php
$a = 42;

$a->toFloat();      // (float) $a  → 42.0
$a->toString();     // (string) $a → "42"
$a->toBool();       // (bool) $a   → true
```

### 4.5 Complete Method List

| Method | Parameters | Return type | Description |
|------|------|---------|------|
| `add($x)` | 1 | Int | addition |
| `sub($x)` | 1 | Int | subtraction |
| `mul($x)` | 1 | Int | multiplication |
| `div($x)` | 1 | Int | division |
| `mod($x)` | 1 | Int | modulo |
| `inc()` | 0 | Int | increment (returns $a + 1) |
| `dec()` | 0 | Int | decrement (returns $a - 1) |
| `abs()` | 0 | Int | absolute value |
| `ceil()` | 0 | Float | round up |
| `floor()` | 0 | Float | round down |
| `round()` | 0-2 | Float | round to nearest |
| `sqrt()` | 0 | Float | square root |
| `pow($x)` | 1 | Var | exponentiation |
| `log()` | 0-1 | Float | natural logarithm |
| `log10()` | 0 | Float | base-10 logarithm |
| `exp()` | 0 | Float | exponential of e |
| `sin()` | 0 | Float | sine |
| `cos()` | 0 | Float | cosine |
| `tan()` | 0 | Float | tangent |
| `asin()` | 0 | Float | arcsine |
| `acos()` | 0 | Float | arccosine |
| `atan()` | 0 | Float | arctangent |
| `atan2($x)` | 1 | Float | two-argument arctangent |
| `deg2rad()` | 0 | Float | degrees to radians |
| `rad2deg()` | 0 | Float | radians to degrees |
| `max($x)` | 1 | Int/Float | maximum value |
| `min($x)` | 1 | Int/Float | minimum value |
| `toFloat()` | 0 | Float | to float |
| `toString()` | 0 | String | to string |
| `toBool()` | 0 | Bool | to bool |

---

## 5. Float Methods

Float's method set is almost identical to Int's, but the return type is Float (`ceil`/`floor`/`round`/trigonometric functions, etc. remain Float).

```php
$f = 3.14;

$f->add(1.0);       // $f + 1.0  → returns 4.14
$f->sub(1.0);       // $f - 1.0  → returns 2.14
$f->mul(2.0);       // $f * 2.0  → returns 6.28
$f->div(2.0);       // $f / 2.0  → returns 1.57

$f->abs();          // abs(3.14) → 3.14
$f->sqrt();         // sqrt(3.14) → 1.772...
$f->sin();          // sin(3.14)  → 0.00159...
$f->round(2);       // round(3.14, 2) → 3.14

// Float-specific conversions
$f->toInt();        // (int) $f   → 3
$f->toString();     // (string) $f → "3.14"
$f->toBool();       // (bool) $f  → true
```

The main differences between Float and Int methods:
- Int has `mod($x)`, Float does not (floating-point modulo is meaningless)
- Arithmetic methods operate directly on C++ `double`, with performance consistent with hand-written C code

---

## 6. Bool Methods

The Bool type has only two conversion methods:

```php
$b = true;

$b->toInt();        // (int) $b   → 1
$b->toString();     // (string) $b → "1"
```

---

## 7. String Methods

String has the richest method set, covering most string operation needs in daily development.

### 7.1 Basic Operations

```php
$s = "hello world";

$s->length();           // strlen($s)         → 11
$s->isEmpty();          // empty($s)          → false
$s->upper();            // strtoupper($s)     → "HELLO WORLD"
$s->lower();            // strtolower($s)     → "hello world"
$s->upperFirst();       // ucfirst($s)        → "Hello world"
$s->lowerFirst();       // lcfirst($s)        → "hello world"
$s->upperWords();       // ucwords($s)        → "Hello World"

$s->trim();             // trim($s)           → "hello world"
$s->lTrim();            // ltrim($s)          → "hello world"
$s->rTrim();            // rtrim($s)          → "hello world"
$s->trim(" \t\n\r");    // trim($s, chars)
```

### 7.2 Search and Comparison

```php
$s = "hello world";

// checks
$s->startsWith("hello");    // str_starts_with(...)    → true
$s->endsWith("world");      // str_ends_with(...)      → true
$s->contains("lo wo");      // str_contains(...)       → true
$s->compare("hello");       // strcmp(...)             → >0
$s->iCompare("HELLO");      // strcasecmp(...)         → 0
$s->isNumeric();            // is_numeric(...)         → false

// position lookup
$s->indexOf("world");       // strpos($s, "world")     → 6
$s->lastIndexOf("o");       // strrpos($s, "o")        → 7
$s->iIndexOf("WORLD");      // stripos($s, "WORLD")    → 6
$s->iLastIndexOf("O");      // strripos($s, "O")       → 7

// content lookup
$s->find("world");          // strstr($s, "world")     → "world"
$s->iFind("WORLD");         // stristr($s, "WORLD")    → "world"
$s->lastCharIndexOf("o");   // strrchr($s, "o")        → "orld"
```

### 7.3 Substring and Replacement

```php
$s = "hello world";

// substring
$s->substr(0, 5);           // substr($s, 0, 5)        → "hello"
$s->substr(6);              // substr($s, 6)           → "world"

// counting
$s->substrCount("l");       // substr_count($s, "l")   → 3
$s->wordCount();            // str_word_count($s)      → 2

// replacement
$s->replace("hello", "hi");        // str_replace("hello", "hi", $s)
$s->iReplace("HELLO", "hi");       // str_ireplace(...)
$s->substrReplace("hi", 0, 5);     // substr_replace($s, "hi", 0, 5)
$s->stripTags();                   // strip_tags($s)
$s->stripTags("<br><p>");          // strip_tags($s, tags)
```

### 7.4 Splitting and Joining

```php
$s = "hello world";

// splitting
$words = $s->split(" ");            // explode(" ", $s) → ["hello", "world"]
$words->count();                    // 2

// joining (operates on an array)
$words->join(", ");                 // implode(", ", $words) → "hello, world"

// repetition
$s->repeat(3);                      // str_repeat($s, 3) → "hello worldhello worldhello world"

// padding
$s->pad(20, "-");                   // str_pad($s, 20, "-")
```

### 7.5 Encoding and Escaping

```php
$s = "hello world & <test>";

$s->htmlEntityEncode();             // htmlentities($s)
$s->htmlEntityDecode();             // html_entity_decode($s)
$s->htmlSpecialCharsEncode();       // htmlspecialchars($s)
$s->htmlSpecialCharsDecode();       // htmlspecialchars_decode($s)

$s->urlEncode();                    // urlencode($s)       → "hello+world+%26+%3Ctest%3E"
$s->urlDecode();                    // urldecode($s)
$s->rawUrlEncode();                 // rawurlencode($s)
$s->rawUrlDecode();                 // rawurldecode($s)

$s->addSlashes();                   // addslashes($s)
$s->stripSlashes();                 // stripslashes($s)
$s->addCSlashes("A..z");            // addcslashes($s, "A..z")
$s->stripCSlashes();                // stripcslashes($s)

$s->base64Encode();                 // base64_encode($s)
$s->base64Decode();                 // base64_decode($s)
```

### 7.6 Hash and Checksum

```php
$s = "hello";

$s->md5();          // md5($s)       → "5d41402abc4b2a76b9719d911017c592"
$s->sha1();         // sha1($s)      → "aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d"
$s->crc32();        // crc32($s)     → 907060870
$s->hash("sha256"); // hash("sha256", $s) → "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"
$s->hashCode();     // C++ std::hash → hash value (int)
```

### 7.7 Regular Expression Matching

```php
$s = "hello world";

// match(pattern) → returns the match array
$result = $s->match("/hello/");

// matchAll(pattern) → returns all match results
$results = $s->matchAll("/[a-z]+/");
```

### 7.8 Serialization

```php
$s = '{"name":"John","age":30}';

$data = $s->jsonDecode();           // json_decode($s, true)  → ["name" => "John", ...]
$obj = $s->jsonDecodeToObject();    // json_decode($s)        → stdClass

$s = 'a:3:{i:0;s:3:"foo";i:1;s:3:"bar";i:2;s:3:"baz";}';
$arr = $s->unserialize();           // unserialize($s)        → ["foo", "bar", "baz"]
```

### 7.9 Multibyte Strings (mbstring)

Methods with the `mb` prefix correspond to PHP's `mb_*` function family:

```php
$s = "你好世界";

$s->mbLength();                     // mb_strlen($s)          → 4
$s->mbUpper();                      // mb_strtoupper($s)
$s->mbLower();                      // mb_strtolower($s)
$s->mbSubstr(0, 2);                 // mb_substr($s, 0, 2)    → "你好"
$s->mbIndexOf("世界");              // mb_strpos($s, "世界")  → 2
$s->mbFind("世");                   // mb_strstr($s, "世")
$s->mbDetectEncoding();             // mb_detect_encoding($s)
$s->mbConvertEncoding("UTF-8");     // mb_convert_encoding($s, "UTF-8")
$s->mbConvertCase(MB_CASE_TITLE);   // mb_convert_case($s, MB_CASE_TITLE)
$s->mbTrim();                       // mb_trim($s)
$s->mbLTrim();                      // mb_ltrim($s)
$s->mbRTrim();                      // mb_rtrim($s)
```

### 7.10 C++ Native Methods

The following methods directly call the `C++` member functions of `phpx::Variant`, with no corresponding `PHP` function:

```php
$s = "hello";

$s->equals("hello");        // C++ String.equals() —— value comparison
```

---

## 8. Array Methods

Array methods are divided into **read-only methods** and **mutating methods**. Mutating methods directly modify the original array variable.

### 8.1 Basic Information

```php
$arr = [1, 3, 5, 7, 9];

$arr->count();          // count($arr)         → 5
$arr->isEmpty();        // empty($arr)         → false
$arr->isList();         // array_is_list($arr) → true
```

### 8.2 CRUD

```php
$arr = [1, 2, 3];

// mutating methods (modify the original array)
$arr->push(4);              // array_push($arr, 4)          → [1,2,3,4]
$arr->push(5, 6, 7);        // supports multiple arguments
$arr->pop();                // array_pop($arr)              → returns 7
$arr->shift();              // array_shift($arr)            → returns 1
$arr->unshift(0);           // array_unshift($arr, 0)       → [0,...]
$arr->set(0, 100);          // C++ Array.set(0, 100)        → [100,...]
$arr->del(0);               // C++ Array.del(0)             → delete index 0
$arr->clean();              // C++ Array.clean()            → []

// read-only methods
$arr = ['a' => 1, 'b' => 2, 'c' => 3];
$arr->get('a');             // C++ Array.get('a')           → 1
$arr->keyExists('a');       // array_key_exists('a', $arr)  → true
$arr->contains(2);          // in_array(2, $arr)            → true
$arr->search(2);            // array_search(2, $arr)        → "b"
```

### 8.3 Iteration and Aggregation

```php
$arr = [1, 3, 5, 7, 9];

$arr->sum();                // array_sum($arr)       → 25
$arr->product();            // array_product($arr)   → 945
$arr->all(fn($v) => ...);   // array_all($arr, fn)
$arr->any(fn($v) => ...);   // array_any($arr, fn)

$arr->map(fn($v) => $v * 2);// array_map(fn, $arr)   → [2,6,10,14,18]
$arr->reduce(fn($c, $v) => $c + $v, 0);  // array_reduce($arr, fn, 0)
$arr->filter(fn($v) => $v > 5);          // array_filter($arr, fn)
$arr->walk(fn(&$v) => $v *= 2);          // array_walk($arr, fn)
```

### 8.4 Sorting

```php
$arr = [3, 1, 4, 1, 5, 9];

$arr->sort();               // sort($arr)            → [1,1,3,4,5,9]
$arr->sortDesc();           // rsort($arr)           → [9,5,4,3,1,1]
$arr->keySort();            // ksort($arr)           → sort by key
$arr->valueSort();          // asort($arr)           → sort by value (keys preserved)
```

All sorting methods **modify the original array**.

### 8.5 Set Operations

```php
$a = [1, 2, 3, 4, 5];
$b = [4, 5, 6, 7, 8];

$a->diff($b);               // array_diff($a, $b)   → [1,2,3]
$a->intersect($b);          // array_intersect(...)  → [4,5]
$a->merge($b);              // array_merge($a, $b)   → [1,2,3,4,5,4,5,6,7,8]
$a->unique();               // array_unique($a)      → [1,2,3,4,5]
$a->flip();                 // array_flip($a)        → {1:0, 2:1, 3:2, ...}
$a->reverse();              // array_reverse($a)     → [5,4,3,2,1]
$a->replace($b);            // array_replace($a, $b)
$a->values();               // array_values($a)      → reindex
$a->combine($keys);         // array_combine($keys, $a)
$a->fillKeys($value);       // array_fill_keys($a, $value)
```

> Methods such as `diff`, `intersect`, `merge`, and `replace` support **variadic arguments**: `$a->merge($b, $c, $d)` can merge multiple arrays at once.

### 8.6 Extraction and Slicing

```php
$arr = ['a' => 1, 'b' => 2, 'c' => 3, 'd' => 4, 'e' => 5];

$arr->keys();               // array_keys($arr)         → ['a','b','c','d','e']
$arr->slice(1, 3);          // array_slice($arr, 1, 3) → ['b'=>2, 'c'=>3, 'd'=>4]
$arr->chunk(2);             // array_chunk($arr, 2)    → [[1,2],[3,4],[5]]
$arr->column('name');       // array_column($arr, 'name')
$arr->splice(1, 3, [6,7]);  // array_splice($arr, 1, 3, [6,7]) —— modifies the original array
$arr->rand(2);              // array_rand($arr, 2)

$arr->keyFirst();           // array_key_first($arr) → "a"
$arr->keyLast();            // array_key_last($arr)  → "e"
$arr->find(fn($v) => $v > 3);  // array_find($arr, fn)
```

### 8.7 String-Related

```php
$arr = ["hello", "world"];

$arr->join(", ");           // implode(", ", $arr) → "hello, world"
$arr->replaceStr("hello", "hi");    // str_replace("hello", "hi", $arr)
$arr->iReplaceStr("HELLO", "hi");   // str_ireplace(...)
```

### 8.8 Serialization

```php
$arr = ["name" => "John", "age" => 30];

$arr->serialize();          // serialize($arr)      → "a:2:{...}"
$arr->marshal();            // serialize($arr) —— alias
$arr->jsonEncode();         // json_encode($arr)    → '{"name":"John","age":30}'
```

### 8.9 Type Conversion

```php
$arr = [1, 2, 3];

$arr->toInt();              // (int) non-empty array → 1
$arr->toFloat();            // (float) non-empty array → 1.0
$arr->toBool();             // (bool) non-empty array → true
$arr->toString();           // → "Array"
```

### 8.10 Complete Method Category Table

| Category | Methods |
|------|------|
| Basic information | `count`, `isEmpty`, `isList` |
| CRUD | `push`, `pop`, `shift`, `unshift`, `set`, `get`, `del`, `clean`, `keyExists`, `contains`, `search` |
| Iteration & aggregation | `sum`, `product`, `all`, `any`, `map`, `reduce`, `filter`, `walk` |
| Sorting | `sort`, `sortDesc`, `keySort`, `valueSort` |
| Set operations | `diff`, `diffAssoc`, `diffKey`, `intersect`, `intersectAssoc`, `merge`, `unique`, `flip`, `reverse`, `replace`, `values`, `combine`, `fillKeys` |
| Extraction & slicing | `keys`, `slice`, `chunk`, `column`, `splice`, `rand`, `keyFirst`, `keyLast`, `find` |
| String | `join`, `replaceStr`, `iReplaceStr`, `countValues`, `pad` |
| Serialization | `serialize`, `marshal`, `jsonEncode` |
| Type conversion | `toInt`, `toFloat`, `toBool`, `toString` |

---

## 9. Stream Methods

The Stream type represents a file handle or network connection. It is obtained through functions such as `fopen()`.

### 9.1 Read/Write

```php
$fp = fopen("test.txt", "w+");

$fp->write("hello world\n");      // fwrite($fp, "hello world\n") → bytes written
$fp->write("more data", 4);       // fwrite($fp, "more data", 4)  → writes only 4 bytes

$fp->seek(0);                     // fseek($fp, 0)   → back to the beginning
$content = $fp->read(1024);       // fread($fp, 1024)
$content = $fp->getContents();    // stream_get_contents($fp)

$char = $fp->getChar();           // fgetc($fp)      → reads one character
$line = $fp->getLine();           // fgets($fp)      → reads one line
$line = $fp->getLine(1024);       // fgets($fp, 1024)
$line = $fp->getRecord(1024, "\n"); // stream_get_line($fp, 1024, "\n")
```

### 9.2 Metadata and Status

```php
$fp->tell();                // ftell($fp)           → current position
$fp->eof();                 // feof($fp)            → whether at EOF
$fp->stat();                // fstat($fp)           → file status array
$fp->getMetaData();         // stream_get_meta_data($fp)
$fp->isLocal();             // stream_is_local($fp)
$fp->isTTY();               // stream_isatty($fp)
```

### 9.3 Control Operations

```php
$fp->truncate(0);           // ftruncate($fp, 0)    → truncate file
$fp->sync();                // fsync($fp)           → sync to disk
$fp->dataSync();            // fdatasync($fp)       → sync data
$fp->close();               // fclose($fp)          → close

// locks
$fp->lock(LOCK_EX);         // flock($fp, LOCK_EX)
$fp->lock(LOCK_SH);         // flock($fp, LOCK_SH)
$fp->lock(LOCK_UN);         // flock($fp, LOCK_UN)

// buffering settings
$fp->setBlocking(true);     // stream_set_blocking($fp, true)
$fp->setChunkSize(8192);    // stream_set_chunk_size($fp, 8192)
$fp->setReadBuffer(8192);   // stream_set_read_buffer($fp, 8192)
$fp->setWriteBuffer(8192);  // stream_set_write_buffer($fp, 8192)
$fp->setTimeout(30);        // stream_set_timeout($fp, 30)
$fp->supportsLock();        // stream_supports_lock($fp)
```

### 9.4 Socket Operations

```php
// server side
$server = stream_socket_server("tcp://0.0.0.0:8080");
$client = $server->accept();             // stream_socket_accept($server)
$client->accept(30);                     // 30-second timeout

// information
$client->getSocketName(true);            // stream_socket_get_name —— remote address
$server->getSocketName(false);           // stream_socket_get_name —— local address

// data
$client->sendTo("hello", 0, $addr);      // stream_socket_sendto(...)
$client->recvFrom(1024);                 // stream_socket_recvfrom(...)
$client->recvFrom(1024, 0, $addr);       // with address

// control
$client->enableCrypto(true);             // stream_socket_enable_crypto → enable TLS
$client->shutdown(STREAM_SHUT_RDWR);     // stream_socket_shutdown(...)

// filters
$fp->appendFilter("string.toupper");     // stream_filter_append(...)
$fp->prependFilter("string.tolower");    // stream_filter_prepend(...)
```

### 9.5 Stream Copy

```php
$src = fopen("source.txt", "r");
$dst = fopen("dest.txt", "w");

$src->copy($dst);                    // stream_copy_to_stream($src, $dst)
$src->copy($dst, 4096);              // specify buffer size
```

---

## 10. Big* High-Precision Type Methods

### 10.1 BigInt Methods

```php
$a = std::bigInt("12345678901234567890");

// arithmetic (all return a new BigInt, original unchanged)
$b = $a->add(1);        // $a + 1
$c = $a->sub(1);        // $a - 1
$d = $a->mul(2);        // $a * 2
$e = $a->div(10);       // $a / 10
$f = $a->mod(1000000);  // $a % 1000000
$g = $a->pow(3);        // $a ** 3

// unary methods
$h = $a->neg();         // -$a
$i = $a->abs();         // abs($a)

// special methods
$j = $a->gcd(15);       // gcd($a, 15)
$k = $a->divmod(3);     // quotient and remainder: returns [$q, $r]
$l = $a->powmod(5, 97); // modular exponentiation: ($a ** 5) % 97
$m = $a->sqrt();        // square root (truncated to integer)

// bitwise operation methods
$n = $a->bitAnd(0xFF);   // bitwise AND: $a & 0xFF
$o = $a->bitOr(0xFF);    // bitwise OR: $a | 0xFF
$p = $a->bitXor(0xFF);   // bitwise XOR: $a ^ 0xFF
$q = $a->bitNot();       // bitwise NOT: ~$a
$r = $a->testBit(3);         // test whether bit 3 is 1
$s = $a->popCount();         // number of 1 bits in binary
$t = $a->bitShiftLeft(3);    // left shift: $a << 3
$u = $a->bitShiftRight(2);   // right shift: $a >> 2

// comparison
$cmp = $a->cmp(100);    // -1/0/1

// type conversion
$a->toString();         // → "12345678901234567890"
$a->toInt();            // → int (may truncate)
$a->toFloat();          // → float (may lose precision)
```

| Method | Parameters | Return | Description |
|------|------|------|------|
| `add($x)` | 1 | BigInt | addition |
| `sub($x)` | 1 | BigInt | subtraction |
| `mul($x)` | 1 | BigInt | multiplication |
| `div($x)` | 1 | BigInt | integer division |
| `mod($x)` | 1 | BigInt | modulo |
| `pow($x)` | 1 | BigInt | exponentiation |
| `neg()` | 0 | BigInt | negation |
| `abs()` | 0 | BigInt | absolute value |
| `gcd($x)` | 1 | BigInt | greatest common divisor |
| `divmod($x)` | 1 | Array | quotient and remainder |
| `powmod($exp, $mod)` | 2 | BigInt | modular exponentiation |
| `sqrt()` | 0 | BigInt | square root (truncated) |
| `bitAnd($x)` | 1 | BigInt | bitwise AND |
| `bitOr($x)` | 1 | BigInt | bitwise OR |
| `bitXor($x)` | 1 | BigInt | bitwise XOR |
| `bitNot()` | 0 | BigInt | bitwise NOT |
| `testBit($index)` | 1 | Int | test whether a bit is 1 |
| `popCount()` | 0 | Int | number of 1 bits in binary |
| `bitShiftLeft($n)` | 1 | BigInt | left shift `$a << $n` |
| `bitShiftRight($n)` | 1 | BigInt | right shift `$a >> $n` |
| `cmp($x)` | 1 | Int | comparison |
| `toString()` | 0 | String | to string |
| `toInt()` | 0 | Int | to integer |
| `toFloat()` | 0 | Float | to float |

### 10.2 Decimal Methods

```php
$d = std::decimal("123.456");

$d->add(std::decimal("50.25"));   // addition
$d->sub(std::decimal("50.25"));   // subtraction
$d->mul(2);                       // multiplication
$d->div(3);                       // division
$d->mod(std::decimal("5.0"));     // modulo
$d->pow(2);                       // exponentiation: $d ** 2
$d->neg();                        // negation
$d->abs();                        // absolute value
$d->divmod(std::decimal("10"));   // quotient and remainder: returns [$q, $r]
$d->powmod(3, std::decimal("100")); // modular exponentiation: ($d**3) % 100
$d->sqrt();                       // square root
$d->floor();                      // round down
$d->ceil();                       // round up
$d->round();                      // round to integer
$d->round(2);                     // keep 2 decimal places
$d->cmp(std::decimal("100"));     // comparison
$d->toString();                   // → "123.456"
$d->toInt();                      // → 123
$d->toFloat();                    // → 123.456 (double)
```

### 10.3 BigFloat Methods

```php
$bf = std::bigFloat(3.14159265);

$bf->add(1.0);          // addition
$bf->sub(1.0);          // subtraction
$bf->mul(2.0);          // multiplication
$bf->div(2.0);          // division
$bf->neg();             // negation
$bf->abs();             // absolute value
$bf->cmp(3.0);          // comparison → >0
$bf->toString();        // to string
$bf->toInt();           // → 3
$bf->toFloat();         // → 3.14159265
```

> **Immutability**: All methods of Big* types **return a new value** and do not modify the original variable. Int, Float, String, and other methods are likewise immutable. Only Array's mutating methods modify the original array. See [Section 12](#12-mutating-methods-vs-immutable-methods).

---

## 11. Method Chaining

The return types of universal methods are known at compile time, so they can be chained directly:

```php
// string chaining
$result = "  Hello World!  "
    ->trim()
    ->lower()
    ->substr(0, 5)
    ->upper();
echo $result;  // "HELLO"

// Int chaining
$result = 100
    ->add(50)     // 150
    ->mul(3)      // 450
    ->sub(100)    // 350
    ->toString();
echo $result;  // "350"

// BigInt chaining (immutable, returns new value)
$result = std::bigInt("100")
    ->add(std::bigInt(50))
    ->mul(std::bigInt(3))
    ->toString();
echo $result;  // "450"

// cross-type chaining
$sum = "123456789012345678901234567890"
    ->length();     // String.length() → Int
echo $sum;  // 30

// chaining + final conversion
$result = std::bigInt("99999999999999999999")
    ->add(std::bigInt(1))
    ->toString();
echo "100000000000000000000 = " . $result;
```

> **Note**: Intermediate values in chaining are passed through return values, and the original variable never changes. Since methods of all types (except Array's mutating methods) do not modify the original value, each step of the chain operates on the return value of the previous step.

---

## 12. Mutating Methods vs Immutable Methods

Universal methods of different types behave differently in terms of mutability:

### 12.1 Mutating Methods (modify the original value)

The mutating methods of **Array** may **modify the original array**:

```php
$arr = [1, 2, 3];

$arr->push(4);      // modifies $arr → [1,2,3,4]
$arr->pop();        // modifies $arr → [1,2,3]
$arr->sort();       // modifies $arr → [1,2,3]
$arr->set(0, 100);  // modifies $arr → [100,2,3]
$arr->clean();      // modifies $arr → []
```

### 12.2 Immutable Methods (return a new value)

Except for Array's mutating methods, methods of all types are immutable; they only return a new value and **do not modify the original value**.

**Int and Float**:

```php
$a = 100;
$b = $a->add(50);
echo $a;   // 100 (original unchanged)
echo $b;   // 150
```

All methods of **BigInt, Decimal, and BigFloat** **do not modify the original value**, returning a newly created object:

```php
$a = std::bigInt(100);
$b = $a->add(50);   // $a is still 100, $b is 150

$a = std::bigInt(100);
$a->add(50);        // the return value is discarded! $a is still 100
```

All methods of **String** also return a new value, leaving the original string unchanged:

```php
$s = "hello";
$upper = $s->upper();  // $s is still "hello", $upper is "HELLO"
```

### 12.3 Summary of Mutating Methods

| Handler type | Affected type | Examples |
|-------------|---------|------|
| `direct_method_mutate` |  Array | `append`, `set`, `del`, `clean` |
| `php_fn_ref` | Array | `push`, `pop`, `shift`, `unshift`, `sort`, `sortDesc`, `splice`, `walk` |

---

## 13. Method Lookup for Var Types

When the variable's type is `Var` (generic PHP type), the compiler looks up in the following priority order:

1. **Built-in keyword methods** (the `to*` series, defined by `KEYWORD_METHOD_MAP`)
2. **Keyword extension methods** (`MethodsFor('*')`)
3. **Any-type extension methods** (`MethodsFor(Type::Any)`)
4. **Dynamic call** (falls back to ZendVM method calls)

```php
// the type of $x is Var (from function return values, array extraction, etc.)
$x = some_func_returning_var();

// the compiler searches in order:
// 1. first looks for length in the String method table → found! → strlen($x)
echo $x->length();

// 2. first looks for contains in the String method table → found! → str_contains($x, ...)
echo $x->contains("test");
```

> **Note**: If multiple types have a method with the same name, the first matched type takes priority. For example, `toInt` exists in multiple types — the lookup stops at the first matching type.

---

## 14. Type Extension Methods

TypePHP uses class-level `MethodsFor` to declare type extension methods. The provider's target is specified by the type symbol of the root namespace `Type`, and the `public static` methods in the class become extension methods of that type. The root namespace `Type` is different from the compiler-internal `TypePHP\Type`.

The following example is in the global namespace. If the code is in another namespace, write `#[\MethodsFor(\Type::Int)]`, or first execute `use \MethodsFor; use \Type;` and then use the same short names. Attributes, target types, and `ClassName::class` all follow PHP's standard name resolution rules and support the `use ... as ...` alias.

### 14.1 Int Extension

```php
#[MethodsFor(Type::Int)]
final class IntExtensions
{
    public static function isPrime(int $value): bool
    {
        if ($value < 2) return false;
        for ($i = 2; $i * $i <= $value; $i++) {
            if ($value % $i === 0) return false;
        }
        return true;
    }
}

$number->isPrime();
```

The first parameter is the receiver; the arguments provided in the method call correspond starting from the second parameter:

```php
#[MethodsFor(Type::Int)]
final class IntExtensions
{
    public static function between(int $value, int $min, int $max): bool
    {
        return $value >= $min && $value <= $max;
    }
}

var_dump($number->between(10, 100));
```

### 14.2 String Extension

```php
#[MethodsFor(Type::String)]
final class StringExtensions
{
    public static function surround(
        string $value,
        string $left = '[',
        string $right = ']'
    ): string {
        return $left . $value . $right;
    }
}

echo 'hello'->surround();       // [hello]
echo 'hello'->surround('<', '>'); // <hello>
```

### 14.3 Array Extension

```php
#[MethodsFor(Type::Array)]
final class ArrayExtensions
{
    public static function firstOrNull(array $items): mixed
    {
        return $items[0] ?? null;
    }
}

$first = $items->firstOrNull();
```

### 14.4 Stream Extension

```php
#[MethodsFor(Type::Stream)]
final class StreamExtensions
{
    public static function readChunk(stream $stream, int $length): string
    {
        return fread($stream, $length);
    }
}

$chunk = $stream->readChunk(4096);
```

### 14.5 Supported Provider Targets

```php
Type::Int
Type::Float
Type::Bool
Type::BigInt
Type::BigFloat
Type::Decimal

Type::String
Type::Array
Type::Object
Type::Stream
Type::Box
```

Provider methods must be `public static`, and the first parameter's type must match the provider target. private/protected methods are not registered and can be used as internal helpers. The method name is used directly as the public extension method name, without any naming-style conversion.

### 14.6 Return Types and Chaining

The return types of extension methods participate in subsequent type inference:

```php
$result = $number
    ->between(10, 100) // bool
    ->toString();

$text = $items
    ->firstOrNull()
    ->toString()
    ->trim();
```

The same target type and method name can only be registered once; duplicate providers will report an error at compile time.

## 15. Keyword Extension Methods

When a method needs to apply to any receiver, use `'*'` as the provider target and declare the first parameter as `any`:

```php
#[MethodsFor('*')]
final class KeywordExtensions
{
    public static function inspect(
        mixed $value,
        string $label = 'value'
    ): mixed {
        echo $label, ': ';
        var_dump($value);
        return $value;
    }

    public static function typeName(mixed $value): string
    {
        return get_debug_type($value);
    }
}
```

Keyword extensions can be used on different types:

```php
$number->inspect('number');
$array->inspect('array');
$object->inspect('object');

echo $value->typeName();
```

Default parameters and variadic parameters follow the rules of ordinary static methods:

```php
#[MethodsFor('*')]
final class ComparisonExtensions
{
    public static function isOneOf(mixed $value, mixed ...$choices): bool
    {
        return in_array($value, $choices, true);
    }
}

$status->isOneOf('pending', 'running', 'finished');
```

`Type::Any` only matches receivers whose static type is `any`; `'*'` is the keyword wildcard target that applies to all types. Built-in keyword methods (such as `toInt()`, `toString()`, and `toObject()`) cannot be overridden by extensions. For the complete declaration and validation rules of providers, see [Extension Method Providers](methods-for.md).

## 16. Complete Examples

### 16.1 String Processing Pipeline

```php
<?php
declare(strict_types=1);

function main(): void {
    $raw = "  <h1>Hello World!</h1>  \n";

    $processed = $raw
        ->trim()                        // strip leading/trailing whitespace
        ->stripTags()                   // remove HTML tags
        ->lower()                       // convert to lowercase
        ->upperWords();                 // capitalize first letters

    echo "original: " . $raw->jsonEncode() . "\n";
    echo "processed: " . $processed . "\n";
    echo "length: " . $processed->length() . "\n";
    echo "is numeric: " . (int)$processed->isNumeric() . "\n";
}
?>
```

### 16.2 Array Data Processing

```php
<?php
declare(strict_types=1);

function main(): void {
    $data = [5, 2, 8, 1, 9, 3, 7];

    echo "original: " . $data->join(", ") . "\n";

    // sorting
    $data->sort();
    echo "sorted: " . $data->join(", ") . "\n";

    // statistics
    echo "count: " . $data->count() . "\n";
    echo "sum: " . $data->sum() . "\n";
    echo "minimum: " . $data->get(0) . "\n";
    echo "maximum: " . $data->get($data->count() - 1) . "\n";
    echo "contains 5: " . (int)$data->contains(5) . "\n";

    // filtering and mapping
    $even = $data->filter(function($v) { return $v % 2 == 0; });
    echo "even: " . $even->values()->join(", ") . "\n";

    $doubled = $data->map(function($v) { return $v * 2; });
    echo "doubled: " . $doubled->join(", ") . "\n";

    // serialization
    echo "JSON: " . $data->jsonEncode() . "\n";
}
?>
```

### 16.3 High-Precision Computation and Chaining

```php
<?php
declare(strict_types=1);

function main(): void {
    // large integer factorial (using compound assignment, more concise)
    $n = 50;
    $result = std::bigInt(1);
    for ($i = 2; $i <= $n; $i++) {
        $result *= $i;
    }
    $digits = strlen($result->toString());
    echo "{$n}! has {$digits} digits\n";

    // method chain: BigInt arithmetic + conversion
    $big = std::bigInt(1000);
    $val = $big->mul(3)->add(200)->sub(50)->div(10)->toString();
    echo "1000 * 3 + 200 - 50 / 10 = {$val}\n";

    // Decimal financial calculation
    $price = std::decimal("19.99");
    $qty = 5;
    $taxRate = std::decimal("0.08");
    $total = $price * $qty * ($taxRate->add(std::decimal(1)));
    echo "total: " . $total->toString() . "\n";

    // BigFloat scientific calculation
    $pi = std::bigFloat("3.14159265358979323846");
    $area = $pi * 100;
    echo "circle area: " . $area->toString() . "\n";
    echo "rounded: " . $area->toInt() . "\n";
}
?>
```

### 16.4 File Processing

```php
<?php
declare(strict_types=1);

function main(): void {
    // write to file
    $fp = fopen("data.txt", "w+");
    $fp->write("Line 1\n");
    $fp->write("Line 2\n");
    $fp->write("Line 3\n");

    // back to the beginning and read
    $fp->seek(0);
    while (!$fp->eof()) {
        $line = $fp->getLine();
        if ($line !== false) {
            echo $line->trim();
            echo " (length: " . $line->trim()->length() . ")\n";
        }
    }

    $fp->close();
}
?>
```
