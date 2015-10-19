# Derive4J: Java 8 annotation processor for deriving algebraic data types constructors, pattern matching and more!

[ ![Download](https://api.bintray.com/packages/derive4j/derive4j/derive4j/images/download.svg) ](https://bintray.com/derive4j/derive4j/derive4j/_latestVersion)

## Table of contents
-  [Example: a 'Visitor' for HTTP Request](#example-a-visitor-for-http-request)
    - [Constructors](#constructors)  
    - [equals, hashCode, toString?](#equals-hashcode-tostring)  
    - [Pattern matching synthax](#pattern-matching-synthax)
    - [Accessors (getters)](#accessors-getters)
    - [Functional setters ('withers')](#functional-setters-withers)
    - [First class lazyness](#first-class-lazyness)
    - [Flavours](#flavours)
    - [Optics (functional lenses)](#optics-functional-lenses)
- [Updating deeply nested immutable datastructure](#updating-deeply-nested-immutable-datastructure)
- [Popular use-case: domain specific languages](#popular-use-case-domain-specific-languages)
- [Use it in your project](#use-it-in-your-project)
- [Contact](#contact)

**Caution**: if you are not familiar with Algebraic Data Types or the "visitor pattern" then you should learn a bit about them before further reading of this page:
- http://eng.wealthfront.com/2015/02/pattern-matching-in-java-with-visitor.html
- https://en.wikipedia.org/wiki/Algebraic_data_type
- https://en.wikipedia.org/wiki/Tagged_union
- http://tomasp.net/blog/types-and-math.aspx/
- http://fsharpforfunandprofit.com/posts/type-size-and-design/
- https://codewords.recurse.com/issues/three/algebra-and-calculus-of-algebraic-data-types
- http://chris-taylor.github.io/blog/2013/02/10/the-algebra-of-algebraic-data-types/

This project has a special dedicace to Tony Morris' blog post [Debut with a catamorphism] (http://blog.tmorris.net/posts/debut-with-a-catamorphism/index.html).

So what can this project do for us, poor functional programmer stuck with a legacy language called Java?
A good deal of what is available for free in better languages like haskell: pattern matching, laziness...
An example being worth a thousand words...

# Example: a 'Visitor' for HTTP Request

Let's say we want to modelize an HTTP request. For the sake of the example let's say that an http request can either be
- a GET on a given ```path```
- a DELETE on a given ```path```
- a POST of a content ```body``` on a given ```path```
- a PUT of a content ```body``` on a given ```path```

and nothing else!

You could then use the [corrected visitor pattern](http://logji.blogspot.ch/2012/02/correcting-visitor-pattern.html) and write the following class in Java:

```java
package org.derive4j.exemple;

/** A data type to modelize an http request. */
@Data
public abstract class Request {

  /** the Request 'visitor' interface, R being the return type
   *  used by the 'accept' method : */
  interface Cases<R> {
    // A request can either be a 'GET' (of a path):
    R GET(String path);
    // or a 'DELETE' (of a path):
    R DELETE(String path);
    // or a 'PUT' (on a path, with a body):
    R PUT(String path, String body);
    // or a 'POST' (on a path, with a body):
    R POST(String path, String body);
    // and nothing else!
  }

  // the 'accept' method of the visitor pattern:
  public abstract <R> R match(Cases<X> cases);

  /**
   * Alternatively and equivalently to the visitor pattern above, if you prefer a more FP style,
   * you can define a catamorphism instead. (see examples)
   * (most useful for standard data type like Option, Either, List...)
   */
}
```
## Constructors

Without Derive4J, you would have to create subclass of ```Request``` for all four cases. That is, write at the minimum something like:
```java
  public static Request GET(String path) {
    return new Request() {
      @Override
      public <X> X match(Cases<X> cases) {
        return cases.GET(path);
      }
    };}
```
for each cases.
But thanks to the ```@Data``` annotation, Derive4j will do that for you! That is, it will generate a ```Requests``` class (the name is configurable, the class is generated by default in ```target/generated-sources/annotations``` when using maven) with four static factory methods (what we call '*constructors*' in FP):
```java
  public static Request GET(String path) {...}
  public static Request DELETE(String path) {...}
  public static Request PUT(String path, String body) {...}
  public static Request POST(String path, String body) {...}
```

## equals, hashCode, toString?
Derive4J philosophy is to be as safe and consistent as possible. That is why Object.{equals, hashCode, toString} are not implemented by generated classes by default. Nonetheless, as a concession to legacy, it is possible to force Derive4J to implement them, by declaring them abstract. Eg by adding the following in your annotated class:
```java
  @Override
  public abstract int hashCode();
  @Override
  public abstract boolean equals(Object obj);
  @Override
  public abstract String toString();
```
The safer solution would be to never use those methods and use 'type classes' instead, eg. [Equal](https://github.com/functionaljava/functionaljava/blob/master/core/src/main/java/fj/Equal.java), [Hash](https://github.com/functionaljava/functionaljava/blob/master/core/src/main/java/fj/Hash.java) and [Show](https://github.com/functionaljava/functionaljava/blob/master/core/src/main/java/fj/Show.java).
The project [Derive4J for Functiona Java](https://github.com/derive4j/derive4j-fj) aims at generating them automatically.

## Pattern matching synthax
Now let's say that you want a function that return the body size of a ```Request```. Without Derive4J you would write something like:
```java
  static final Function<Request, Integer> getBodySize = request -> 
      request.match(new Cases<Integer>() {
        public Integer GET(String path) {
          return 0;
        }
        public Integer DELETE(String path) {
          return 0;
        }
        public Integer PUT(String path, String body) {
          return body.length();
        }
        public Integer POST(String path, String body) {
          return body.length();
        }
      });
```
With Derive4J you can do that a lot less verbosely, thanks to a generated fluent [structural pattern matching](http://www.deadcoderising.com/pattern-matching-syntax-comparison-in-scala-haskell-ml/) synthax! And it does exhaustivity checks! (you must handle all cases). The above can be rewritten into:
```java
static final Function<Request, Integer> getBodySize = Requests.match()
      .GET(path          -> 0)
      .DELETE(path       -> 0)
      .PUT((path, body)  -> body.length())
      .POST((path, body) -> body.length())
```
or even (because you don't care of GET and DELETE cases):
```java
static final Function<Request, Integer> getBodySize = Requests.match()
      .PUT((path, body)  -> body.length())
      .POST((path, body) -> body.length())
      .otherwise(()      -> 0)
```

## Accessors (getters)
Now, patterning mathing every time you want to inspect an instance of ```Request``` is a bit tedious. For this reason Derive4J generate 'getter' static methods for all fields. For the ```path``` and ```body``` fields, Derive4J will generate the following methods in the ```Requests``` class:
```java
  public static String getPath(Request request){
    return Requests.match()
        .GET(path          -> path)
        .DELETE(path       -> path)
        .PUT((path, body)  -> path)
        .POST((path, body) -> path)
        .apply(request);
  }
  // return an Optional because the body is not present in the GET and DELETE cases:
  static Optional<String> getBody(Request request){
    return Requests.match()
        .PUT((path, body)  -> Optional.of(body))
        .POST((path, body) -> Optional.of(body))
        .otherwise(()      -> Optional.empty())
        .apply(request);
  }
```
(Actually the generated code is equivalent but more efficient)

Using the generated ```getBody``` methods we can rewrite our ```getBodySize``` function into:
```java
static final Function<Request, Integer> getBodySize = request ->
      Requests.getBody(request)
              .map(String::length)
              .orElse(0);
```

## Functional setters ('withers')
The most painful part of immutable data srtuctures (like the one generated by Derive4J) is updating them. Scala case classes have ```copy``` methods for that. Derive4J generate similar modifier and setter methods in the ```Requests``` class:
```java
  public static Function<Request, Request> setPath(String newPath){
    return Requests.match()
            .GET(path          -> Requests.GET(newPath))
            .DELETE(path       -> Requests.DELETE(newPath))
            .PUT((path, body)  -> Requests.PUT(newPath, body))
            .POST((path, body) -> Requests.POST(newPath, body)));
  }
  public static Function<Request, Request> modPath(Function<String, String> pathMapper){
    return Requests.match()
            .GET(path          -> Requests.GET(pathMapper.apply(path)))
            .DELETE(path       -> Requests.DELETE(pathMapper.apply(path)))
            .PUT((path, body)  -> Requests.PUT(pathMapper.apply(path), body))
            .POST((path, body) -> Requests.POST(pathMapper.apply(path), body)));
  }
  public static Function<Request, Request> setBody(String newBody){
    return Requests.match()
            .GET(path          -> Requests.GET(path))    // identity function for GET
            .DELETE(path       -> Requests.DELETE(path)) // and DELETE cases.
            .PUT((path, body)  -> Requests.PUT(path, newBody))
            .POST((path, body) -> Requests.POST(path, newBody)));
  }
  ...
```
By returning a function, modifiers and setters allow for a lightweight syntax when updating deeply nested immutables datastructures.

## First class lazyness
Language like haskell provide lazyness by default, which simplify a lot of algorithms. In traditional Java you would have to declare a method argument as ```Supplier<Request>``` (and do memoization) to emulate lazyiness. With Derive4J that is no more necessary as it generate a lazy constructor that gave you transparent lazy evaluation for all consumer for your datatype:
```java
  // the requestExpression will be lazy-evaluated on the first call
  // to the 'match' method of the returned Request instance:
  public static Request lazy(Supplier<Request> requestExpression) {
    ...
  }
```
## Flavours
In the example above, we have use the default ```JDK``` flavour. Also available are ```FJ``` ([Functional Java](https://github.com/functionaljava/)) and ```Fugue``` ([Fugue](https://bitbucket.org/atlassian/fugue)) flavours. When using those alternative flavours, Derive4J will use eg. the specific ```Option``` implementations from those projects instead of the jdk ```Optional``` class.

## Optics (functional lenses)
If you are not familiar with optics, have a look at [Monocle](https://github.com/julien-truffaut/Monocle) (for scala, but [Functional Java](https://github.com/functionaljava/functionaljava/) provides similar abstraction).

Using Derive4J generated code, defining optics is a breeze (you need to use the ```FJ``` flavour by specifying ```@Data(flavour = Flavour.FJ)```:
```java
  /**
   * Lenses: optics focused on a field present for all datatype contructors
   * (getter cannot 'failed'):
   */
  public static final Lens<Request, String> _path = lens(
      Requests::getPath,
      Requests::setPath);
  /**
   * Optional: optics focused on a field that may not be present for all contructors
   * (getter return an 'Option'):
   */
  public static final Optional<Request, String> _body = optional(
      Requests::getBody,
      Requests::setBody);
  /**
   * Prism: optics focused on a specific constructor:
   */
  public static final Prism<Request, String> _GET = prism(
      // Getter function
      Requests.match()
          .GET(fj.data.Option::some)
          .otherwise(Option::none),
      // Reverse Get function (aka constructor)
      Requests::GET);

  // If there is more than one field, we use a tuple as the prism target:
  public static final Prism<Request, P2<String, String>> _POST = prism(
      // Getter:
      Requests.match()
          .POST((path, body) -> some(p(path, body)))
          .otherwise(Option::none),
      // reverse get (construct a POST request given a P2<String, String>):
      p2 -> Requests.POST(p2._1(), p2._2()));
}
```
# Updating deeply nested immutable datastructure
Let's say you want to modelize a CRM. Each client is a ```Person``` which can be contacted either by email, telephone or postal mail. With Derive4J you could write the following:
```java
import org.derive4j.*;
import java.util.function.BiFunction;

@Data
public abstract class Address {
  public abstract <R> R match(@FieldNames({"number", "street"}) 
  			      BiFunction<Integer, String, R> Address);
}
```
```java
import org.derive4j.Data;

@Data
public abstract class Contact {
    interface Cases<R> {
      R byEmail(String email);
      R byPhone(String phoneNumber);
      R byMail(Address postalAddress);
    }
    public abstract <R> R match(Cases<R> cases);
}
```
```java
import org.derive4j.*;
import java.util.function.BiFunction;

@Data
public abstract class Person {
  public abstract <R> R match(@FieldNames({"name", "contact"})
                              BiFunction<String, Contact, R> Person);
}
```
But now we have a problem: All client have been imported from a legacy database with a off by one error for the street number! We must create a function that increments each person street number (if it exists) by one. And without modifying the original datastructure (because it is immutable).
With Derive4J, writing such a function is trivial:
```java
import java.util.Optional;
import java.util.function.Function;

import static org.derive4j.exemple.Addresss.Address;
import static org.derive4j.exemple.Addresss.getNumber;
import static org.derive4j.exemple.Addresss.modNumber;
import static org.derive4j.exemple.Contacts.getPostalAddress;
import static org.derive4j.exemple.Contacts.modPostalAddress;
import static org.derive4j.exemple.Persons.Person;
import static org.derive4j.exemple.Persons.getContact;
import static org.derive4j.exemple.Persons.modContact;

  public static void main(String[] args) {

    Person joe = Person("Joe", Contacts.byMail(Address(10, "Main St")));

    Function<Person, Person> incrementStreetNumber = modContact(
    						       modPostalAddress(
    						         modNumber(number -> number + 1)));
    
    // newP is a copy of p with the street number incremented:
    Person correctedJoe = incrementStreetNumber.apply(joe);

    Optional<Integer> newStreetNumber = getPostalAddress(getContact(correctedJoe))
        .map(postalAddress -> getNumber(postalAddress));

    System.out.println(newStreetNumber); // print "Optional[11]" !!
  }
```


# Popular use-case: domain specific languages
Algebraic data types are particulary well fitted for creating DSLs. Like a calculator for arthmetic expressions:
```java
import java.util.function.Function;
import org.derive4j.Data;
import static org.derive4j.exemple.Expressions.*;

@Data
public abstract class Expression {

	interface Cases<R> {
		R Const(Integer value);
		R Add(Expression left, Expression right);
		R Mult(Expression left, Expression right);
		R Neg(Expression expr);
	}
	
	public abstract <R> R match(Cases<R> cases);

	private static Function<Expression, Integer> eval = Expressions
		.match()
			.Const(value        -> value)
			.Add((left, right)  -> eval(left) + eval(right))
			.Mult((left, right) -> eval(left) * eval(right))
			.Neg(expr           -> -eval(expr));

	public static Integer eval(Expression expression) {
		return eval.apply(expression);
	}

	public static void main(String[] args) {
		Expression expr = Add(Const(1), Mult(Const(2), Mult(Const(3), Const(3))));
		System.out.println(eval(expr)); // (1+(2*(3*3))) = 19
	}
}
```

# Use it in your project
Derive4J is available via Jcenter. It should be declared as a compile-time only dependency (not needed at runtime).
## Maven:
```xml
<repositories>
  <repository>
    <id>bintray</id>
    <url>http://jcenter.bintray.com</url>
  </repository>
</repositories>
...
<dependency>
  <groupId>org.derive4j</groupId>
  <artifactId>derive4j</artifactId>
  <version>0.3</version>
  <optional>true</optional>
</dependency>
```
## Gradle
```
compile(group: 'org.derive4j', name: 'derive4j', version: '0.3', ext: 'jar')
```
## Contact
jb@giraudeau.info, [@jb9i](https://twitter.com/jb9i) or use the project github issues.
