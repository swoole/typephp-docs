# Keyword Methods

Keyword Methods are method syntax built into the TypePHP compiler. They do not rely on whether the object actually declares a method with the same name; instead, the compiler directly recognizes them and generates the corresponding code.

Keyword methods are mainly used for type conversion and type continuation:

```php
$id = $value->toInt();
$name = $value->toString();
$user = $value->toObject(User::class);
$stream = $value->toStream();
```

Keyword methods can act on variables, array elements, properties, and other expressions:

```php
$data['id']->toInt();
$response->body->toString();
$repository->find($id)->toObject(User::class);
```

## 1. Built-in Keyword Methods

| Method | Return type | Parameters | Description |
|------|----------|------|------|
| `toInt()` | `int` | none | convert to integer |
| `toFloat()` | `float` | none | convert to floating-point number |
| `toString()` | `string` | none | convert to string |
| `toBool()` | `bool` | none | convert to boolean |
| `toArray()` | `array` | none | convert to array |
| `toAny()` | `mixed` | none | downgrade the static type to dynamic type |
| `toRef()` | reference | none | explicitly obtain a reference to a referable expression |
| `toStream()` | `stream` | none | convert to Stream type |
| `toObject()` | `object` | optional class name | convert to object or restore the concrete object type |
| `toBigInt()` | `BigInt` | none | convert to high-precision integer |
| `toBigFloat()` | `BigFloat` | none | convert to high-precision floating-point number |
| `toDecimal()` | `Decimal` | none | convert to high-precision decimal number |
| `toStdArray()` | `StdArray` | type, length | restore a fixed-length Std container |
| `toStdVector()` | `StdVector` | element type | restore a dynamic Std container |
| `toStdMap()` | `StdMap` | key and value types | restore a hash map |
| `toStdOrderedMap()` | `StdOrderedMap` | key and value types | restore an ordered map |

Built-in keyword methods have the highest priority, and user code cannot override them through object extensions or keyword extensions.

## 2. Basic Type Conversion

### 2.1 toInt

```php
$id = '123'->toInt();
var_dump($id); // int(123)

$enabled = true->toInt();
var_dump($enabled); // int(1)
```

Equivalent to converting the expression to TypePHP's native integer type. The return value can continue calling Int universal methods:

```php
$result = $value->toInt()->add(10);
```

### 2.2 toFloat

```php
$price = '12.50'->toFloat();
var_dump($price); // float(12.5)
```

Returns TypePHP's native floating-point type:

```php
$rounded = $value->toFloat()->round(2);
```

### 2.3 toString

```php
$text = $value->toString();
echo $text->trim()->upper();
```

`toString()` is often used to restore the string type from `mixed`, so that subsequent string universal methods can be statically resolved.

### 2.4 toBool

```php
$enabled = $config['enabled']->toBool();

if ($enabled) {
    echo 'enabled';
}
```

### 2.5 toArray

```php
$items = $value->toArray();
echo $items->count();
```

For ordinary values, `toArray()` performs an array conversion. For objects with a zero-argument `toArray()` method, the underlying conversion helper can call the object method to obtain the array result.

## 3. Dynamic Type Conversion

### 3.1 toAny

`toAny()` downgrades the current expression to the `mixed` / `any` dynamic type:

```php
$dynamic = $typedValue->toAny();
```

It is suitable for scenarios where you need to return to Zend dynamic semantics, for example:

- a parameter explicitly requires `mixed`
- you want to discard the current static type information
- subsequent operations must be decided at runtime
- you need to avoid continuing to use native numeric operation rules

`toAny()` takes no arguments:

```php
$value->toAny();       // correct
$value->toAny('arg');  // compile error
```

Once the type is downgraded, the compiler does not automatically restore the original concrete object type. Use `toObject(ClassName::class)` to re-continue when needed.

### 3.2 toRef

`toRef()` explicitly obtains a reference, equivalent to TypePHP's `std::ref()`:

```php
$callback($value->toRef());
```

It is mainly used in scenarios where the compiler cannot tell from a dynamic call whether a parameter is passed by reference:

```php
$callable($data->toRef());
```

`toRef()` can only be used on locatable lvalues, such as variables, array elements, and object properties:

```php
$value->toRef();
$array['key']->toRef();
$object->property->toRef();
```

Temporary computation results usually cannot obtain a reference:

```php
($a + $b)->toRef(); // not supported
```

`toRef()` takes no arguments.

For statically resolved native references, local aliases, and dynamic-call escape restrictions, see [Strongly Typed References](strong-references.md).

## 4. Object Type Continuation

### 4.1 Generic Object

When no class name is passed, `toObject()` returns a generic object with no concrete class information:

```php
$object = $value->toObject();
```

Such an object allows dynamic property and method access, but the compiler cannot resolve calls to the native methods of a concrete class.

### 4.2 Restoring the Concrete Object Type

Passing `ClassName::class` restores the concrete class information:

```php
$user = $value->toObject(User::class);

echo $user->getName();
```

A common scenario is obtaining an object from an array, a dynamic function, or a `mixed` return value:

```php
$user = $container['user']->toObject(User::class);
$user->save();
```

It can also be chained directly:

```php
echo $container['user']
    ->toObject(User::class)
    ->getName()
    ->upper();
```

Object conversion with a class name checks whether the actual object matches the target class or its inheritance relationship. A type error is thrown when the types do not match.

For code that must also run on Zend PHP, use the equivalent `std::object($value, ClassName::class)` form and provide a Zend-side polyfill.

For details on object type continuation, see [Object Type Conversion](object-type-conversion.md).

## 5. Stream Type Conversion

`toStream()` restores a dynamic resource to the TypePHP Stream type:

```php
$stream = $value->toStream();
$stream->write('hello');
$stream->seek(0);
echo $stream->getContents();
```

It is especially useful when obtaining a stream resource from an array or dynamic function:

```php
$streams[0]
    ->toStream()
    ->write($payload);
```

If the actual value is not a valid stream resource, the conversion or subsequent calls will produce an error.

## 6. High-Precision Type Conversion

### 6.1 toBigInt

```php
$number = $value->toBigInt();
$result = $number->mul(100)->add(1);
```

### 6.2 toDecimal

```php
$price = $value->toDecimal();
$total = $price->mul($quantity);
```

### 6.3 toBigFloat

```php
$number = $value->toBigFloat();
$result = $number->sqrt();
```

For conversion rules, precision, and limitations between high-precision types, see [Arbitrary Precision Math](math.md) and [Basic Type Conversion](type-convert.md).

## 7. Std Container Type Continuation

After a Std container crosses a dynamic function boundary through a Box resource, it loses its concrete template type. The `toStd*` methods are used to restore the container type, and they do not copy the underlying container.

### 7.1 StdVector

```php
$vector = $value->toStdVector(Type::Int);
$vector[] = 42;
```

### 7.2 StdArray

```php
$array = $value->toStdArray(Type::Float, 4);
$array[0] = 3.14;
```

### 7.3 StdMap

```php
$map = $value->toStdMap(
    Type::Int,
    Type::String
);
```

### 7.4 StdOrderedMap

```php
$map = $value->toStdOrderedMap(
    Type::String,
    User::class
);
```

The `toStd*` methods must be used for top-level variable assignment, and the target variable cannot be reassigned to another type. For the complete rules, see [Std Containers](std-containers.md).

## 8. Type Inference and Chaining

The return types of keyword methods are directly determined by the compiler:

```php
$value->toInt();               // int
$value->toString();            // string
$value->toArray();             // array
$value->toObject(User::class); // User
$value->toStream();            // stream
```

Therefore, you can safely continue calling methods of the corresponding type:

```php
$result = $value
    ->toString()
    ->trim()
    ->lower()
    ->length();
```

Objects, Streams, and high-precision types also support type continuation:

```php
$name = $value
    ->toObject(User::class)
    ->getName()
    ->trim();
```

## 9. Parameter Limitations

Unless otherwise specified, basic conversion methods take no arguments:

```php
$value->toInt();
$value->toString();
$value->toAny();
$value->toRef();
```

`toObject()` can accept one class name:

```php
$value->toObject(User::class);
```

The `toStd*` methods accept compile-time parameters such as element type, key type, and value type depending on the container kind.

Incorrect argument counts are reported at the compilation stage and are not deferred to runtime.

## 10. Lookup Priority

When TypePHP encounters a named method call, built-in keyword methods are processed first. For example:

```php
$value->toString();
```

Regardless of whether the receiver is currently `mixed`, an integer, a string, or an object, the compiler will first recognize it as a built-in conversion method.

Built-in keyword methods cannot be overridden by the following mechanisms:

- real object methods
- object extension methods
- universal type extension methods
- custom keyword extension methods
- `__call()`

This priority guarantees that type conversion semantics are always unambiguous.

## 11. Differences from Universal Methods

| Item | Keyword methods | Universal methods |
|--------|------------|----------|
| Main purpose | type conversion, type continuation | perform operations on a type |
| Receiver requirement | can act on any type | must match a concrete type |
| Lookup method | directly recognized by the compiler | looked up by receiver type |
| Examples | `toInt()`, `toObject()` | `trim()`, `count()`, `write()` |
| Return type | built into and determined by the compiler | determined by the method definition |

For example:

```php
$text = $value->toString(); // keyword method: restore the string type
$text = $text->trim();      // String universal method: process the string
```

## 12. Extended Keyword Methods

Besides built-in keyword methods, you can add project-specific methods for any value through class-level `MethodsFor`. This is an extension capability of the keyword method system, and does not belong to the built-in keyword methods themselves.

```php
#[MethodsFor('*')]
final class KeywordExtensions
{
    public static function inspect(mixed $value, string $label): mixed
    {
        echo $label, ': ';
        var_dump($value);
        return $value;
    }
}

$value->inspect('input');
```

Extended keyword methods must be `public static`, and the first parameter must be `mixed`. For their complete declaration, validation, and call rules, see [Extension Method Providers](methods-for.md).
