---
layout: post
title: "java函数式接口"
subtitle: "一些java函数式接口的研究和总结"
date: 2024-06-13 22:00:00
author: "Momoka7"
header-style: text
catalog: true
tags:
  - java
  - function
  - Lambda
---

Java 中有几个常见的函数式接口，这些接口通常位于`java.util.function`包中。函数式接口是指只有一个抽象方法的接口，通常用于`Lambda`表达式和`方法引用`。

# 基础使用

1. **`Function<T, R>`**: 接受一个输入参数，返回一个结果。**使用`apply`调用**。

   ```java
   import java.util.function.Function;

   public class FunctionExample {
       public static void main(String[] args) {
           Function<Integer, String> intToString = (i) -> "Number: " + i;
           String result = intToString.apply(10);
           System.out.println(result);  // 输出: Number: 10
       }
   }
   ```

2. **`Predicate<T>`**: 接受一个输入参数，返回一个布尔值。**使用`test`调用。**

   ```java
   import java.util.function.Predicate;

   public class PredicateExample {
       public static void main(String[] args) {
           Predicate<String> isLongerThan5 = (s) -> s.length() > 5;
           boolean result = isLongerThan5.test("Hello, World!");
           System.out.println(result);  // 输出: true
       }
   }
   ```

3. **`Consumer<T>`**: 接受一个输入参数，没有返回值。**使用`accept`调用**。

   ```java
   import java.util.function.Consumer;

   public class ConsumerExample {
       public static void main(String[] args) {
           Consumer<String> printUpperCase = (s) -> System.out.println(s.toUpperCase());
           printUpperCase.accept("hello");  // 输出: HELLO
       }
   }
   ```

4. **`Supplier<T>`**: 没有输入参数，返回一个结果。**使用`get`调用**。

   ```java
   import java.util.function.Supplier;

   public class SupplierExample {
       public static void main(String[] args) {
           Supplier<String> stringSupplier = () -> "Hello, Supplier!";
           String result = stringSupplier.get();
           System.out.println(result);  // 输出: Hello, Supplier!
       }
   }
   ```

5. **`BiFunction<T, U, R>`**: 接受两个输入参数，返回一个结果。**使用`apply`调用**

   ```java
   import java.util.function.BiFunction;

   public class BiFunctionExample {
       public static void main(String[] args) {
           BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
           int result = add.apply(3, 4);
           System.out.println(result);  // 输出: 7
       }
   }
   ```

6. **`UnaryOperator<T>`**: 接受一个输入参数，返回一个与输入类型相同的结果。**使用`apply`调用**

   ```java
   import java.util.function.UnaryOperator;

   public class UnaryOperatorExample {
       public static void main(String[] args) {
           UnaryOperator<Integer> square = (i) -> i * i;
           int result = square.apply(5);
           System.out.println(result);  // 输出: 25
       }
   }
   ```

7. **`BinaryOperator<T>`**: 接受两个相同类型的输入参数，返回一个与输入类型相同的结果。**使用`apply`调用**

   ```java
   import java.util.function.BinaryOperator;

   public class BinaryOperatorExample {
       public static void main(String[] args) {
           BinaryOperator<Integer> multiply = (a, b) -> a * b;
           int result = multiply.apply(3, 4);
           System.out.println(result);  // 输出: 12
       }
   }
   ```

这些函数式接口大大简化了代码的编写，特别是在使用 Lambda 表达式时，使代码更加简洁和易读。在实际开发中，这些接口被广泛用于各种场景，如集合操作、流处理等。

# andThen、compose

`andThen` 和 `compose` 是 Java 中 `Function` 接口的两个默认方法，它们用于函数组合。通过这两个方法，可以将多个函数组合成一个函数，从而实现更复杂的功能。

## `andThen` 方法

`andThen` 方法用于先执行当前函数，再执行`andThen`中的函数。当前函数的输出将作为`andThen`中函数的输入。

#### 语法

```java
default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
    Objects.requireNonNull(after);
    return (T t) -> after.apply(apply(t));
}
```

#### 示例

```java
import java.util.function.Function;

public class AndThenExample {
    public static void main(String[] args) {
        Function<Integer, Integer> multiplyBy2 = x -> x * 2;
        Function<Integer, String> convertToString = x -> "Result: " + x;

        // 先乘以2，然后转换成字符串
        Function<Integer, String> multiplyAndConvert = multiplyBy2.andThen(convertToString);
        String result = multiplyAndConvert.apply(5); // Result: 10

        //也可以直接串联调用
        String result = multiplyBy2.andThen(convertToString).apply(5);
        System.out.println(result); // 输出: Result: 10
    }
}
```

## `compose` 方法

`compose` 方法与 `andThen` 相反，先执行 `compose` 中的函数，再执行当前函数。`compose` 中函数的输出将作为当前函数的输入。

#### 语法

```java
default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
    Objects.requireNonNull(before);
    return (V v) -> apply(before.apply(v));
}
```

#### 示例

```java
import java.util.function.Function;

public class ComposeExample {
    public static void main(String[] args) {
        Function<String, Integer> parseInt = x -> Integer.parseInt(x);
        Function<Integer, Integer> multiplyBy2 = x -> x * 2;

        // 先将字符串转换为整数，然后乘以2
        Function<String, Integer> parseAndMultiply = multiplyBy2.compose(parseInt);
        int result = parseAndMultiply.apply("5"); // 10
        //也可以直接串联调用
        int result = multiplyBy2.compose(parseInt).apply("5");
        System.out.println(result); // 输出: 10
    }
}
```

## 区别与选择

- **执行顺序**：
  - `andThen`：先执行当前函数，再执行`andThen`中的函数。
  - `compose`：先执行`compose`中的函数，再执行当前函数。
- **使用场景**：
  - 使用 `andThen` 时，当前函数的返回类型应与下一个函数的输入类型匹配。
  - 使用 `compose` 时，前一个函数的返回类型应与当前函数的输入类型匹配。

通过合理地使用这两个方法，可以非常方便地将多个函数组合在一起，创建复杂的函数链。

# 柯里化（Currying）

柯里化（Currying）是一种将多参数函数转换为一系列单参数函数的技术

## 原始的两参数加法函数

首先，定义一个接受两个整数并返回它们之和的函数：

```java
import java.util.function.BiFunction;

public class CurryExample {
    public static void main(String[] args) {
        BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
        int result = add.apply(2, 3);
        System.out.println(result); // 输出: 5
    }
}
```

## 实现柯里化函数

将上述的两参数函数转换为一系列单参数函数：

```java
import java.util.function.Function;

public class CurryExample {
    public static void main(String[] args) {
        //该函数接受一个整数a，返回一个新的函数，该函数接受一个整数b并返回a + b的结果。
        Function<Integer, Function<Integer, Integer>> curriedAdd = a -> b -> a + b;

        // 使用柯里化函数
        Function<Integer, Integer> add2 = curriedAdd.apply(2);
        int result = add2.apply(3);

        System.out.println(result); // 输出: 5
    }
}

```

# 其他

## 泛型和通配符的一些细节

> ? 通配符类型

<? extends T> 表示类型的上界，表示?匹配的类型都是类型R的子类，包括R本身
<? super T> 表示类型下界（Java Core中叫超类型限定），表示?匹配的类型都是T的父类，包括T本身。

## 集合中使用泛型和通配符

如果想从集合中**读取数据而不写入数据**，可以使用 ? extends 通配符（集合相当于生产者）,如果想要向集合中**写数据而不读数据**，则使用 ? super 通配符（集合相当于消费者）,如果既要存又要读，则不能使用通配符。

### 例

```java
//生物
class Creature{}
//动物
class Animal extends Creature{}
//猫
class Cat extends Animal{}
//狗
class Dog extends Animal{}
```

#### List<? extends T>

```java
//最常见的用法，泛型的类型是确定的（即Animal类型），
//所以listB能添加Animal、Cat、Dog(因为猫和狗都是动物)。
List<Animal> listB = new ArrayList<>();
//listA的泛型的类型是不确定的（即用通配符?表示的），
//所以listA无法添加元素
//listA由于容纳的是Animal及其子类，所以listA能获取元素，
//并且获取到的元素的类型是Animal类型准没错(多态)
List<? extends Animal> listA = new ArrayList<>();
```

由于只能从List<? extends T>中获取元素，而不能向它添加元素，所以称之为生产者（只出不进）。

#### List<? super T>

```java
//listC能够添加元素
//List<? super Animal>中的Animal是确定的
//即若?表示的是Creature，那么Creature及其子类都可以添加
//上至Object，则说明可添加任意对象（即使和Animal没有继承关系）
//由于无法确定元素的返回值类型到底是啥，故只能获取到Object
List<? super Animal> listC = new ArrayList();
```

由于只能向List<? super T>添加元素，而不能从它里面获取元素，所以称之为消费者（只进不出）

## 形如Function＜? super T, ? extends V＞的理解

在`Function<T, R>`的`andThen`方法中，其函数定义为：

```java
default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
    Objects.requireNonNull(after);
    return (T t) -> after.apply(apply(t)); //R apply(T t);
}
```

这里`R`是`after`的入参类型（也即调用者Function的返回值类型），`V`是`after`的返回值类型。

对`Function<? super R, ? extends V>`的个人理解：

`after`会在`调用者Function`执行后，使用其返回值作为自己的输入，由于`R`是`调用者Function`的返回值，`after`要能使用其返回值，其入参一定要是`R`或其父类（从多态和接口的设计角度来看，入参一般会是更抽象一级的父类）。
