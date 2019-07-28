# What Kotlin could learn from C++'s keyword `const`

Let's say, in Kotlin we have some simple 2D-vector class. It has two methods, one (`normalize`) which mutates the object, and one (`length`) which does not:

```kotlin
import kotlin.math.sqrt

class Vector(private var x: Double, private var y: Double) {
    fun length() = sqrt(x * x + y * y)
    fun normalize() {
        val l = length()
        x /= l
        y /= l
    }
}
```

We could now use it like that:

```kotlin
fun bar(v: Vector) {
    v.normalize()
}

fun baz(v: Vector) {
    println(v.length())
}

fun main() {
    val myVector = Vector(3.0, 4.0)
    bar(myVector)
    baz(myVector)
}
```

However, we might want to make sure `baz` can not mutate our object. It should only be allowed to call the non-mutating methods. To achieve this, we need to provide two different classes:

```kotlin
open class Vector(protected var x: Double, protected var y: Double) {
    fun length() = sqrt(x * x + y * y)
}

class MutableVector(x: Double, y: Double) : Vector(x, y) {
    fun normalize() {
        val l = super.length()
        x /= l
        y /= l
    }
}
```

Now we can adjust the rest of our code, such that the desired property of `baz` is satisfied:

```kotlin
fun bar(v: MutableVector) {
    v.normalize()
}

fun baz(v: Vector) {
    println(v.length())
}

fun main() {
    val myVector = MutableVector(3.0, 4.0)
    bar(myVector)
    baz(myVector)
}
```

[`MutableList`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-mutable-list/index.html) and [`List`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-list/index.html) in Kotlin's standard library follow a similar approach. Actually, Kotlin is just an example here. It's the same in other languages too.

So, let's translate this straight into C++:

```cpp
#include <cmath>
#include <iostream>

class vector {
public:
    vector(double x, double y): x_(x), y_(y) {}
    double length() const {
        return std::sqrt(x_ * x_ + y_ * y_);
    }
protected:
    double x_;
    double y_;
};

class mutable_vector: public vector {
public:
    mutable_vector(double x, double y): vector(x, y) {}
    void normalize() {
        const auto l = length();
        x_ /= l;
        y_ /= l;
    }
};

void bar(mutable_vector& v) {
    v.normalize();
}

void baz(vector& v) {
    std::cout << v.length() << std::endl;
}

int main() {
    mutable_vector my_vector(3.0, 4.0);
    bar(my_vector);
    baz(my_vector);
}
```

Looks fine. However, in a code review, this would raise a huge laugh since C++ provides a much better solution. Using the [`const`](https://en.cppreference.com/w/cpp/keyword/const) keyword, one can let the compiler do the tedious work of providing two different interfaces!

It looks as follows:

```cpp
#include <cmath>
#include <iostream>

class vector {
public:
    vector(double x, double y): x_(x), y_(y) {}
    double length() const {
        return std::sqrt(x_ * x_ + y_ * y_);
    }
    void normalize() {
        const auto l = length();
        x_ /= l;
        y_ /= l;
    }
private:
    double x_;
    double y_;
};

void bar(vector& v) {
    v.normalize();
}

void baz(const vector& v) {
    std::cout << v.length() << std::endl;
}

int main() {
    vector my_vector(3.0, 4.0);
    bar(my_vector);
    baz(my_vector);
}
```

By marking `vector::length` as const, but not `vector::normalize`, the compiler knows, which member functions can be called depending on the const qualification of an instance or a reference to one.

`baz` now takes a reference-to-const parameter, which produces the exact effect we wanted, i.e., trying to call `v.normalize()` in its body would result in a compile-time error. :tada: