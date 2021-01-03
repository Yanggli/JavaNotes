[toc]

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
Manual memory management: `C, C++`. Garbage-collected language: `Java`.
```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        element = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {...}

    public Object pop() {
        if (size == 0) {
            throw new EmptyStackException();
        }

        return elements[--size];
    }
}
```

Consider the code above, there is nothing wrong with the code itself. However, the program has a "memory leak". And will cause `OutOfMemoryError`, but it's rare. 
*Memory Leak*: if a stack grows and then shrinks, the objects that popped off the stack will not be garbage collected, even if the program using the stack has no more references to them. This is because stack maintain <u> *Obsolete references* </u> to those objects.
```java
// fixed version
public Object pop() {
    if (size == 0) {
        throw new EmptyStackException();
    }
    Object result = elements[--size];
    elements[size] = null;
    return result;
}
```

**Nulling out object references should be exception rather than the norm. Whenever a class manages its own memory, the programmer should be alert for memory alert. Another common source of memory leaks is caches.**

Then, through the below method, is the memory space used to create `str` be reclaimed? In fact, no, the list still holds a reference to `str`, so the memory space when `str` is created will not be reclaimed. For this kind of reference, we call it <u> *unconscious memory reference* </u>.
```java
// whether str is recycled?
List<String> list = new ArrayList<>();
String str = "java";
list.add(str);
str  = null;
```

## Item 8: Avoid finalizers and cleaners

### Finalizers

<u> *Finalizers* </u> is finalize() function that to release resources used by objects before they're removed from the memory. 
```java
// Called by the garbage collector on an object when garbage collection
// determines that there are no more references to the object.
public class Object{
    @Deprecated(since="9")
    protected void finalize() throws Throwable { }
}
```
对象可由两种状态，涉及到两类状态空间，一是终结状态空间 F = {unfinalized, finalizable, finalized}；二是可达状态空间 R = {reachable, finalizer-reachable, unreachable}。各状态含义如下：
*unfinalized: 新建对象会先进入此状态，GC并未准备执行其finalize方法，因为该对象是可达的
* finalizable: 表示GC可对该对象执行finalize方法，GC已检测到该对象不可达。正如前面所述，GC通过F-Queue队列和一专用线程完成finalize的执行
* finalized: 表示GC已经对该对象执行过finalize方法
* reachable: 表示GC Roots引用可达
* finalizer-reachable(f-reachable)：表示不是reachable，但可通过某个finalizable对象可达
* unreachable：对象不可通过上面两种途径可达
```java
public class GC {

    public static GC SAVE_HOOK = null;

    public static void main(String[] args) throws InterruptedException {
        // 新建对象，因为SAVE_HOOK指向这个对象，对象此时的状态是(reachable,unfinalized)
        SAVE_HOOK = new GC();
        //将SAVE_HOOK设置成null，此时刚才创建的对象就不可达了，因为没有句柄再指向它了，对象此时状态是(unreachable，unfinalized)
        SAVE_HOOK = null;
        //强制系统执行垃圾回收，系统发现刚才创建的对象处于unreachable状态，并检测到这个对象的类覆盖了finalize方法，因此把这个对象放入F-Queue队列，由低优先级线程执行它的finalize方法，此时对象的状态变成(unreachable, finalizable)或者是(finalizer-reachable,finalizable)
        System.gc();
        // sleep，目的是给低优先级线程从F-Queue队列取出对象并执行其finalize方法提供机会。在执行完对象的finalize方法中的super.finalize()时，对象的状态变成(unreachable,finalized)状态，但接下来在finalize方法中又执行了SAVE_HOOK = this;这句话，又有句柄指向这个对象了，对象又可达了。因此对象的状态又变成了(reachable, finalized)状态。
        Thread.sleep(500);
        // 对象处于(reachable,finalized)状态应该是合理的。对象的finalized方法被执行了，因此是finalized状态。又因为在finalize方法是执行了SAVE_HOOK=this这句话，本来是unreachable的对象，又变成reachable了。
        if (null != SAVE_HOOK) { //此时对象应该处于(reachable, finalized)状态 
            // 这句话会输出，注意对象由unreachable，经过finalize复活了。
            System.out.println("Yes , I am still alive");
        } else {
            System.out.println("No , I am dead");
        }
        // 再一次将SAVE_HOOK放空，此时刚才复活的对象，状态变成(unreachable,finalized)
        SAVE_HOOK = null;
        // 再一次强制系统回收垃圾，此时系统发现对象不可达，虽然覆盖了finalize方法，但已经执行过了，因此直接回收。
        System.gc();
        // 为系统回收垃圾提供机会
        Thread.sleep(500);
        if (null != SAVE_HOOK) {
            // 这句话不会输出，因为对象已经彻底消失了。
            System.out.println("Yes , I am still alive");
        } else {
            System.out.println("No , I am dead");
        }
    }

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("execute method finalize()");
        // 这句话让对象的状态由unreachable变成reachable，就是对象复活
        SAVE_HOOK = this;
    }
}
```

***

**Finalizers are unpredictable, often dangerous and generally unnecessary.** The Java 9 replacement for finalizers is <u> *Cleaners* </u>. **Cleaners are less dangerous than finalizers, but still unpredictable, slow, and generally unnecessary.**

In C++, destructors are the normal way to reclaim the resources associated with an object, a necessary counterpart to constructors. In Java, the garbage collector reclaims the storage associated with an object when it becomes unreachable, requiring no special effort on the part of the programmer.

**Disadvantage(Finalizers)**: 
* no guarantee they will be executed promptly(迅速的). It can take arbitrarily long between the time that an object becomes unreachable and the time its dinalizer or cleaner run.
* no guarantee that they will run at all. **Should not depend on a finalizer or cleaner to update persistent state.**
* uncaught exception thrown during finalization is ignored.
* server performance: increase running time. (cleaner is better.)
* finalizers open your class up to finalizer attacks.


