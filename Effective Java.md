# Creating and Destroying Objects

## Item 1: Consider static factory methods instead of constructors
Basically, the way to obtain an instance is to provide a public constructor. There is another technique that provide a <u>*public static factory method* </u> which is a static method that returns an instance of the class.

```java
public class PhoneNumber {
    private static final PhoneNumber COMMON = new PhoneNumber(123, 1234);

    private final int areaCode;
    private final int number;

    private PhoneNumber(int areaCode, int number) {
        checkArgument(areaCode > 100); // Guava (concerns: version incompatible)
        this.areaCode = areaCode;
        this.number = number;
    }

    public static PhoneNumber of(int areaCode, int number) {
        if (areaCode == 123 && number == 1234) {
            return COMMON;
        }
        return new PhoneNumber(areaCode, number);
    }

    public static void main(String[] args) {
        PhoneNumber common1 = PhoneNumber.of(123, 1234);
        PhoneNumber common2 = PhoneNumber.of(123, 1234);
        System.out.println(common1 == common2);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        PhoneNumber that = (PhoneNumber) o;
        return areaCode == that.areaCode && number == that.number;
    }

    @Override
    public int hashCode() {
        return Objects.hash(areaCode, number);
    }

    @Override
    public String toString() {
        return MoreObjects.toStringHelper(this)
                .add("areaCode: ", areaCode)
                .add("number: ", number)
                .toString();
    }
}
```

```java
package com.company;

public class Person {
    private String title;
    private final String name;
    private final String surname;
    private String prefix;

    private Person(String title, String name, String surname, String prefix) {
        this.title = title;
        this.name = name;
        this.surname = surname;
        this.prefix = prefix;
    }

    public static class Builder {
        private String title;
        private final String name;
        private final String surname;
        private String prefix;


        private Builder(String name, String surname) {
            this.name = name;
            this.surname = surname;
        }

        public static Builder of(String name, String surname) {
            return new Builder(name, surname);
        }

        public void setPrefix(String prefix) {
            this.prefix = prefix;
        }

        public Person build() {
            // check statement
            return new Person(title, name, surname, prefix);
        }
    }
}
```

**Advantages:**
* Static factory method has names.
  * With descriptive names, customer can better understand it.
  * A class can have only a single constructor with a given signature. By providing different lists of parameters, customer will use different constructor. But it's hard for customers to remember details. Because static factory method has names, so they do not have previous restriction.
* Static factory method are not required to create a new object each time when they're revoked.
  ```java
    // if this number is very common, will return common and not create a new instance every time
    public static PhoneNumber of(int areaCode, int number) {
        if (areaCode == 123 && number == 1234) {
            return COMMON;
        }
        return new PhoneNumber(areaCode, number);
    }

    public static void main(String[] args) {
        PhoneNumber common = PhoneNumber.of(123, 1234);
        PhoneNumber common2 = PhoneNumber.of(123, 1234);

        // compare them using == instead of equals.
        System.out.println(common == common2);
    }
  ```
* Static factory method can return an object of any subtype of their return type.
  * Hiding from the client (better encapsulation) of object creation
    ```java
    public class UserFactory {

        public static User newUser(UserEnum type){
            switch (type){
                case ADMIN: return new Admin();
                case STAFF: return new StaffMember();
                case CLIENT: return new Client();
                default:
                    throw new IllegalArgumentException("Unsupported user. You input: " + type);
            } 
        }

        public static void main(String[] args) {
            // client code - give me an admin object, 
            // don't care about the inner details of how it gets constructed
            User admin = UserFactory.newUser(ADMIN); 
        }
    }
    ```
  * Flexibility to swap out implementations without breaking client code
    ```java
    // static factory method
    public static List<String> getMyList(){
        return new ArrayList<>(); 
    }

    public class MyMuchBetterList<E> extends AbstractList<E> implements List<E> {
    // implementation
    }

    public static List<String> getMyList(){
        return new MyMuchBetterList<>(); // compiles and works, subtype of list
    }
    ```
* The class of returned object can vary from call to call as a function of the <u> input parameters </u>. Example: EnumSet. 
    ```java
    public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
        Enum<?>[] universe = getUniverse(elementType);
        if (universe == null)
            throw new ClassCastException(elementType + " not an enum");

        if (universe.length <= 64)
            return new RegularEnumSet<>(elementType, universe);
        else
            return new JumboEnumSet<>(elementType, universe);
    }
    ```
* The class of returned object need not exit when the class containing the method is written. This one is essential components in a service provider framework.
  * Service Interface
  * Provider Registration API
  * Service access API
  * Service provider interface

 **Disadvantages:**
 * Classes without public or protected constructors cannot be subclassed.
 * They are hard for programmers to find.
   * from, of, valueOf, create or newInstance, getType, newType and type

## Item 2: Consider a builder when faced with many constructor parameters
Static factories and constructors share a limitation: they do not scale well to large number of optional parameters.

Traditionally, programmers have used the <u> *telescoping constructor*</u> pattern. This constructor will force customer to pass required value. **Telescoping constructor works, but it is hard to write client code when there are many parameters, and harder still to read it.** And if customer accidentally reverses two such parameters, the compiler will not complain, but the program will misbehave at runtime.
```java
public class NutritionFacts {
    private final int servingSize; // ml
    private final int servings; // per container
    private final int calories; // per serving
    private final int fat; // g/serving
    private final int sodium; // mg/serving
    private final int carbohydrate; // g/serving

    public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories) {
        this(servingSize, servings, calories, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat) {
        this(servingSize, servings, calories, fat, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium) {
        this(servingSize, servings, calories, fat, servings, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium, int carbohydrate) {
        this.servingSize = servingSize;
        this.servings = servings;
        this.calories = calories;
        this.fat = fat;
        this.sodium = sodium;
        this.carbohydrate = carbohydrate;
    }
}
```

A second alternative when facing with many optional parameters is a constructor is <u> *JavaBeans* </u> pattern, in which you call a parameterless constructor to create the object and then call setter methods to set each required parameter and each optional parameter of interest. This patter has no disadvantage of the telescoping constructor pattern. But it is wordy.
JavaBeans has its own disadvantages. 
* Because construction is split across multiple calls, a JavaBean may be in an inconsistent state partway through its construction.
* JavaBeans pattern precludes the possibility of making a class immutable.
```java
public class NutritionFacts {
    // parameters initialized to default values
    private final int servingSize = -1; 
    private final int servings = -1; 
    private final int calories = 0; 
    private final int fat = 0; 
    private final int sodium = 0; 
    private final int carbohydrate = 0;

    public NutritionFacts(){}

    // settings
    public void setServingSize(int val) {
        servingSize = val;
    }

    public void setServings(int val) {
        servings = val;
    }

    public void setCalories(int val) {
        calories = val;
    }

    // ... ...
}
```

The third one is <u> *Builder* </u> pattern. The client calls a constructor with all of thr required parameters and gets a <u> builder object </u> .
```java
public class Person {
    private String name;
    private String age;

    public Person withName(String name) {
        this.name = name;
        return this;
    }

    public Person withAge(String age) {
        this.age = age;
        return this;
    }

    public Person build() {
        return new Person();
    }
    
    public static void main(String[] args) {
        Person p = new Person().withName("Test").build();
    }
}
```

Problem: lack of compile-time guarantee. `p.getAge().equals(18)` will throw NPE.
Solution: use @NonNull. When running it, it will check the parameter.

**TODO: Builder with hierarchies**

## Item 3: Enforce the singleton property with a private constructor or an enum type
A singleton is simply a class that is instantiated exactly once. Three properties:
* A private constructor
* A static field containing its only instance
* A static factory method for obtaining the instance
```java
public class Singleton {
     private static final Singleton singleton = null ;
     private Singleton() {  }
     public static Singleton getInstance() {
         if (singleton== null ) {
             singleton= new Singleton();
         }
         return singleton;
     }
}
```
While this is a common approach, it's important to note that it can be **problematic in multithreading scenarios**, which is the main reason for using Singletons. Simply put, it can result in more than one instance, breaking the pattern's core principle. Singleton may have multiple processes pass the condition check of (singleton== null) at the same time, so multiple instances are created, and memory leaks are likely.
->  mutually exclusive or synchronized.
```java
// version 1.1
public class Singleton
{
     private static Singleton singleton = null ;
     private Singleton() {  }
     public static Singleton getInstance() {
         if (singleton== null ) {
             synchronized (Singleton. class ) {
                 singleton= new Singleton();
             }
         }
         return singleton;
     }
}
```
The synchronized method will help us synchronize all threads. But we still initialize multiple instances.

```java
// version 1.2
public class Singleton
{
     private static Singleton singleton = null ;
     private Singleton()  {  }
     public static Singleton getInstance()  {
         synchronized (Singleton. class ) {
             if (singleton== null ) {
                singleton= new Singleton();
             }
          }
         return singleton;
     }
}
```
In this version, we sync all thread, and make sure the new operation will be triggered once. However, we sync reading as well (which is not needed).

```java
// version 1.4
public class Singleton
{
     private volatile static Singleton singleton = null ;
     private Singleton()  {    }
     public static Singleton getInstance()   {
         if (singleton== null )  {
             synchronized (Singleton. class ) {
                 if (singleton== null )  {
                     singleton= new Singleton();
                 }
             }
         }
         return singleton;
     }
}
```
```java
public enum Singleton{
    INSTANCE;
}
```

## Item 4: Enforce non-instantiability with a private constructor
You will want to write a class that is just a grouping of static methods and static fields. Such class do not need any constructor. Example: Arrays, Collections. This kind of utility classes were not designed to be initialized. 
1. Abstract the class - > not working
Because sub-class can be initialized.
2. make constructor private.

## Item 5: Prefer dependency injection to hardwiring resources
Many classes depends on one or more underlying resources. The approach below is satisfactory. **Static utility and singletons are inappropriate for classes whose behavior is parameterized by an underlying resources.** We want our class can handle multiple instances of a class.Each of the resources are designed by client. <u> *Dependency injection* </u> is the concept in which objects get other required objects from outside.
```java
public class SpellChecker{
    // The Lexicon class is a data structure of maintaining all unique words that appears in the given data set.
    private static final Lexicon dictionary = ...;
    private SpellChecker(){}
}
```
```java
public class SpellChecker{
    private static final Lexicon dictionary;
    private SpellChecker(Lexicon dictionary){}
}
```

Ideally Java classes should be as independent as possible from other Java classes. This increases the possibility of reusing these classes and to be able to test them independently from other classes. If the Java class creates an instance of another class via the new operator, it cannot be used (and tested) independently from this class and this is called a <u> *hard dependency* </u>. The following example shows a class which has no hard dependencies.
```java
import java.util.logging.Logger;

public class MyClass {

    private Logger logger;

    public MyClass(Logger logger) {
        this.logger = logger;
        // write an info log message
        logger.info("This is a log message.")
    }
}
```

We can use annotation to describe class dependencies. `@Inject, @Name`. The dependencies of injection is: 1) the constructor of the class (construction injection), 2) a field (field injection) and 3) the parameters of a method (method injection). <u> *Injection* </u> passes one object to another through a method or constructor.

**Tips:**
1. Do not use a singleton or static utility class to implement a class that depends on one or more underlying resources whose behavior affects that of the class.
2. Do not have the class create these resources directly. Instead, pass the resources or factories to create them into a constructor (or static factory or builder).

## Item 6: Avoid creating unnecessary objects
Reuse a single object is better than creating a new functionally equivalent object each time it is needed. An object can always be reused if it is immutable. Consider this statement: `String s = new String("test");` This statement creates a new String instance each time it is executed and none of the creation is necessary. The improved version is `String s = "test`. This one can be reused by any other code running in the same virtual machine that happens to contain the same string literal. 
> [Java Heap Space vs. Stack Memory](https://stackify.com/java-heap-vs-stack/#:~:text=The%20Heap%20Space%20contains%20all,blocks%20that%20contain%20their%20methods.)

You can always avoid creating unnecessary objects by using *static factory method*.

Some object creations are much more expensive than others. If you’re going to need such an “expensive object” repeatedly, it may be advisable to cache it for reuse. The problem with this implementation is that it relies on the String.matches method. **While String.matches is the easiest way to check if a string matches a regular expression, it’s not suitable for repeated use in performance-critical situations.** The problem is that it internally creates a Pattern instance for the regular expression and uses it only once, after which it becomes eligible for garbage collection. Creating a Pattern instance is expensive because it requires compiling the regular expression into a finite state machine.
```java
// Performance can be greatly improved!
static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})" + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
```
The way to improve it is: compile the regular expression into a Pattern instance (which is immutable) as part of class initialization, cache it,and reuse the same instance for every invocation of the isRomanNumeral method.
```java
// Reusing expensive object for improved performance
public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile("^(?=.)M*(C[MD]|D?C{0,3})" + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
    static boolean isRomanNumeral(String s) {
        return ROMAN.matcher(s).matches();
    }
}
```

## Item 7: Eliminate obsolete object references
