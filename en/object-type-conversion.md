# Object Type Conversion

Object type conversion is used to recover or declare an object's class information at compile time, so the compiler can generate more efficient and safer C++ code. It mainly involves three types of scenarios:

- `std::object($value, ClassName::class)` or `$value->toObject(ClassName::class)`
- Fallback checks for typed object variable assignment, function return values, and property writes

These scenarios are all related to "whether an object belongs to a certain class", but not all of them happen only at compile time.

## 1. Type Relationships

Object type determination uses PHP's `is-a` semantics, that is, the `instanceof` relationship:

- A `Child` object can be used as a `Base` object.
- A `Base` object cannot be used as a `Child` object.
- Interfaces and abstract classes are judged according to PHP's implementation/inheritance relationships.

At the runtime level, `php::toObject($value, ce)` is used to check the object type. This function requires `$value` to be an object and `value instanceof ce` to hold; otherwise it throws an exception.

## 2. Compile-Time Behavior

The following information is handled at compile time and does not rely on runtime determination.

### 2.1 Type Declaration and Type Inference

When the compiler can obtain the class name directly from the syntax, it records the variable's typed object information:

```php
$user = new User();
```

At this point `$user` is recorded as type `User`. When `$user->method()` is subsequently called, the compiler can directly resolve to `User::method()` and generate a native call.

### 2.2 Statically Provably Safe Assignment

If the right-hand value's type can be proven at compile time to be the left-hand value's type itself or a subclass, the assignment passes directly without inserting a runtime check:

```php
$base = new Base();
$base = new Child(); // Child is-a Base, compile-time safe
```

### 2.3 Statically Provably Wrong Assignment

If the compiler can determine at compile time that the object types are incompatible, it reports a compile-time error directly:

```php
$child = new Child();
$child = new Base(); // Base is not a Child, compile-time error
```

Assignment between unrelated classes also reports a compile-time error.

### 2.4 Type Continuation via `std::object()` or `toObject()`

Both forms tell the compiler at compile time to "treat this expression as the specified class from here on":

```php
$user = std::object($data['user'], User::class);
$user->getName(); // the compiler resolves it as type User
```

`std::object($value, User::class)` and `$value->toObject(User::class)` have the
same TypePHP semantics. Prefer `std::object()` when the source must also run on
Zend PHP, where an application can provide an ordinary compatibility method.

The class name argument here must be resolvable at compile time, such as a string literal, `ClassName::class`, `self::class`, or `parent::class`.

## 3. Runtime Behavior

The following cases must be checked at runtime, because the real object type cannot be fully determined at compile time.

### 3.1 Object Check for `std::object()` and `toObject()`

Although both forms continue type information at compile time, they still generate a runtime check:

```php
$user = std::object($value, User::class);
```

The generated core logic is equivalent to:

```cpp
php::toObject(value, ce_User)
```

At runtime it checks:

- `$value` must be an object.
- `$value instanceof User` must hold.

Therefore, an object of `AdminUser extends User` can pass the `User::class` check; an ordinary `stdClass` or an object of another unrelated class cannot.

### 3.2 Fallback Check for Typed Object Assignment

When the right-hand value is `mixed` / `any`, or its declared type is not precise enough, the compiler inserts a runtime check:

```php
$child = new Child();
$child = std::any($value);
```

The compiler cannot know the real class of `$value` at the static stage, so it checks `$value instanceof Child` at runtime.

If the return type of a function or method is a parent class, interface, or abstract class, that is also a case of insufficient precision:

```php
function make(): Base {
    return new Child();
}

$child = new Child();
$child = make(); // runtime check whether the return value is instanceof Child
```

If `make()` actually returns `Child`, the check passes; if it returns `Base`, the check fails.

### 3.3 Std Container Object Element Conversion

When a Std container stores object types, retrieving an element from the container and restoring it to an object type also goes through a runtime object check:

```php
$objects = std::vector(User::class);
$user = $objects[0];
```

When the container element type is a concrete class name, the compiler knows the target class, but the runtime still needs to ensure the retrieved value is an object of that class or a subclass.

## 4. No Exact Class Comparison

Object type conversion does not use the "exact class equality" rule and does not require `get_class($value) === ClassName::class`.

```php
class Base {}
class Child extends Base {}

$base = (new Child())->toObject(Base::class);  // valid
```

If exact class comparison is truly needed, explicitly use `get_class()` or `$value::class` in the business code. Object type conversion itself follows the `is-a` rule of the PHP type system.

## 5. Selection Recommendations

- When the static type is already accurate, there is no need to use `toObject(Class::class)`.
- After retrieving an object from an array, a dynamic return value, or a `mixed` / `any` value, use `std::object($value, ClassName::class)` or `toObject(ClassName::class)` when you need to recover class information.
- Prefer `std::object()` for code shared with Zend PHP; use the keyword method when TypePHP-only fluent syntax is desirable.
- When you want to keep the dynamic type, use `toAny()` or `std::any()` instead of recovering to a typed object.
- When a forced reference is needed, use `toRef()` or `std::ref()`; this is unrelated to object type conversion.
