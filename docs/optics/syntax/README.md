---
layout: docs
title: Syntax DSL
permalink: /docs/optics/syntax/
---

## Syntax DSL

In some cases the full power of Optics is not required and a familiar property-like syntax to work with values of immutable structures is desired.
To avoid boilerplate Arrow can generate this property-like dsl using `@optics` annotation.

```kotlin
package com.example.domain

@optics data class Street(val number: Int, val name: String)
@optics data class Address(val city: String, val street: Street)
@optics data class Company(val name: String, val address: Address)
@optics data class Employee(val name: String, val company: Company?)
```

The DSL will be generated in a syntax sub-package of your `data class` and can be used by invoking `setter()` on an instance. i.e. for a package `com.example.domain` the DSL will be generated in `com.example.domain.syntax`.

```kotlin
import com.example.domain.*
import com.example.domain.syntax.*

val john = Employee("John Doe", Company("Kategory", Address("Functional city", Street(42, "lambda street"))))

john.setter().company.address.street.name.modify(String::toUpperCase)
// Employee(name=John Doe, company=Company(name=Kategory, address=Address(city=Functional city, street=Street(number=42, name=LAMBDA STREET))))
```

Arrow can also generate a dsl for a `sealed class` which can be helpful to reduce boilerplate code, or improve readability.

```kotlin
package com.example.domain

@optics sealed class NetworkResult
@optics data class Success(val content: String): NetworkResult()
@optics sealed class NetworkError : NetworkResult()
@optics data class HttpError(val message: String): NetworkResult()
object TimeoutError: NetworkError()
```

Let's imagine we have a function `f` of type `(HttpError) -> HttpError` and we want to invoke it on the `NetworkResult`.

```kotlin
val networkResult: NetworkResult = HttpError("boom!")
val f: (String) -> String = String::toUpperCase

when (networkResult) {
  is HttpError -> networkResult.copy(f(networkResult.message))
  else -> networkResult
}
```

We can rewrite this code with our generated dsl.

```kotlin
networkResult.setter().networkError.httpError.message.modify(f)
// HttpError(message=BOOM!)
```

The DSL also has special support for [Each]({{ '/docs/optics/each' | relative_url }}) and [At]({{ '/docs/optics/at' | relative_url }}).

`Each` can be used to focus into a structure `S` and see all its foci `A`. Here we focus into all `Employee`s in the `Employees`.

```kotlin
@optics data class Employees(val employees: ListK<Employee>)
```

```kotlin
import arrow.data.*

val jane = Employee("Jane Doe", Company("Kategory", Address("Functional city", Street(42, "lambda street"))))
val employees = Employees(listOf(john, jane).k())

employees.setter().employees.every(ListK.each()).company.address.street.name.modify(String::capitalize)
// Employees(employees=ListK(list=[Employee(name=John Doe, company=Company(name=Kategory, address=Address(city=Functional city, street=Street(number=42, name=Lambda street)))), Employee(name=Jane Doe, company=Company(name=Kategory, address=Address(city=Functional city, street=Street(number=42, name=Lambda street))))]))
```

If you are in the scope of `Each` you don't need to specify the instance.

```kotlin
ListK.each<Employee>().run {
  employees.setter().employees.every().company.address.street.name.modify(String::capitalize)
}
// Employees(employees=ListK(list=[Employee(name=John Doe, company=Company(name=Kategory, address=Address(city=Functional city, street=Street(number=42, name=Lambda street)))), Employee(name=Jane Doe, company=Company(name=Kategory, address=Address(city=Functional city, street=Street(number=42, name=Lambda street))))]))
```

`At` can be used to focus in `A` at a given index `I` for a given structure `S`.

```kotlin
@optics data class Db(val content: MapK<Int, String>)
```

Here we focus into the value of a given key in `MapK`.

```kotlin
import arrow.optics.syntax.*

val db = Db(mapOf(
  1 to "one",
  2 to "two",
  3 to "three"
).k())

db.setter().content.at(MapK.at(), 2).some.modify(String::reversed)
// Db(content=MapK(map={1=one, 2=owt, 3=three}))
```

If you are in the scope of `At` you don't need to specify the instance.

```kotlin
MapK.at<Int, String>().run {
  db.setter().content.at(2).some.modify(String::reversed)
}
// Db(content=MapK(map={1=one, 2=owt, 3=three}))
```

## Syntax DSL vs Optics

In contrast to regular optics the DSL is bound to a value. Lets also generate `Lens` and `Optional` for our above domain so we can compare the difference.
In above example we want to apply a function `f` to the `Street::name` of the `Company`'s `Address` of an `Employee`.

We can compose an `Optic` that achieves the same goal.

```kotlin
import arrow.optics.Optional

val employeesStreetName: Optional<Employee, String> = employeeCompany() compose companyAddress() compose addressStreet() compose streetName()
```

The result is an `Optional` since `Employee::company` is nullable. The `Optional` can be used to apply a function `f` to any `Employee`.

```kotlin
employeesStreetName.modify(john, String::toUpperCase)
// Employee(name=John Doe, company=Company(name=Kategory, address=Address(city=Functional city, street=Street(number=42, name=LAMBDA STREET))))
```

So while Optics are more powerful, the syntax DSL can help you write elegant and concise code.
