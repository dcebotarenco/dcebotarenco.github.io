---
title: "General Fixture"
date: 2023-06-10T21:23:48+03:00
draft: false
---

My opinion based on references:

* **xUnit Test Patterns** book by _Gerard Meszaros_
* **Test Drive Development: By Example** by _Kent Beck_.
* **Growing Object-Oriented Software, Guided by Tests** by _Steve Freeman_, _Nat Pryce_
* [Object Mother](https://martinfowler.com/bliki/ObjectMother.html)
* [ObjectMother - Easing Test Object Creation in XP](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.18.4710&rep=rep1&type=pdf)

The post contains text snippets from books.

## Let's define 'fixture'

The test fixture is everything we need to have to set up in order to exercise the SUT. It includes at least an instance of the class whose method we are testing. We call everything we need in place to exercise the SUT the **test fixture**, and we call the part of the test logic that we execute to set it up the **fixture setup** phase of the test. The “test fixture”—or just “fixture”— means “the pre-conditions of the test”.

![fixture.png](images/fixture.png)

### Fixture types

Let me briefly introduce the other types of fixtures. \
From the persistence perspective:

* _Fresh Fixture_ - Each test constructs its own brand-new test fixture for its
  own **private** use.
* _Fresh Persistent Fixture_ - Each test persist the fixture but at the end tears it down
* _Shared Fixture_ - We reuse the same instance of the test fixture across many tests. It is a persisted fixture

From the design perspective:

* _Minimal Fixture_ - We use the smallest and simplest fixture possible for each test
* _Standard Fixture_ - We reuse the design of the text fixture across the many tests.

## The Test Smell

In my opinion, _General fixture_ test code smell can make developers life complicated and has a big impact on the maintenance of the code in the long run. The project will end-up in situation when the tests are wrote with a lot of straggle or even wrote for coverage and sonar. Let me try to explain why...

It is also known as _Standard Fixture_. It is related to many other smells and causes like:

* _Obscure Tests_ - It is difficult to understand the test at a glance
    * _Irrelevant Information_ - Often occurs in conjunction with _Hard-Coded Test Data_ or a _General Fixture_ but can also arise because we make visible all data the test needs to execute rather than focusing on the data the test needs to be understood.
    * _Mystery Guest_ - The test reader is not able to see the cause and effect between fixture and verification logic because part of it is done outside the _Test Method_. When either the fixture setup or the result verification part of a test depends on information that is not visible within the test and the test reader finds it difficult to understand the behavior that is being verified without first finding and inspecting the external information, we have a _Mystery Guest_ on our hands.
* _Fragile Fixture_ - When a _Standard Fixture_ is modified to accommodate a new test, several other tests fail.
* _Fragile Test_ - A test fails to compile or run when the SUT is changed in ways that do not affect the part the test is exercising.
* _Data Sensitivity_ - If the data changes, the tests may fail unless great effort has been expended to make them insensitive to the data being used.
* _Context Sensitivity_ - The behavior of the system may be affected by the state of things outside (e.g system clock)
* _Slow Tests_ - Tests are consistently slow because each test builds the same over-engineered fixture
* Many more..

![general-fixture.png](images/general-fixture.png)

I came across _General fixture_ in combination with the [Object Mother](https://martinfowler.com/bliki/ObjectMother.html) at a client.
Here is an obfuscated example of code, the `complete` method is real.

```java
package project.space.earth.feature1;

public class TestClass1 {
    @Test
    void test1() {
        Foo expected = FooMother.complete(); //general/standard fixture
        Foo inputFoo = completeFooWithoutIds(); //local fixture out of general/standard fixture
        given(service.createFoo(inputFoo)).willReturn(expected); //stub

        Foo actual = sut.insert(inputFoo); //exercise

        assertThat(actual).usingRecursiveComparison(recursiveComparisonConfiguration).isEqualTo(expected); //verify
    }

    private static Foo completeFooWithoutIds() {
        Bar bar = BarMother.complete().toBuilder().id(null).childId(null).build();
        Tee tee = TeeMother.completeBuilder().id(null).field1(null).bars(List.of(bar)).build();
        return FooMother.completeBuilder().id(null).field1(null).createdBy(null).createdDate(null).lastModifiedBy(null).lastModifiedDate(null).tee(tee).build();
    }
}

package project.space.in.another.part.of.solar.system.feature2;
public class TestClass2 {

    @Test
    void test2() {
        Object object = new Object();
        Object anObject = new Object();
        Foo existingFoo = FooMother.complete(); //general fixture
        Bad bar = BarMother.complete().toBuilder().objectList(List.of(object)).build(); //local fixture out of general/standard fixture
        given(dependency.action(any(), any())).willReturn(anObject);//stub

        final Foo actual = sut.action(existingFoo, bar); //exercise

        final Foo expected = existingFoo.toBuilder().object(object).build(); //local fixture out of general/standard fixture
        assertThat(actual).usingRecursiveComparison(recursiveComparisonConfiguration).isEqualTo(expected); //verify
    }
    
}

package project.space.next.to.jupiter.feature3;
public class TestClass3 {
    //the same situation
}

package project.space.somewhere.next.to.the.moon.feature4;
public class TestClass4 {
    //the same situation
}
```
The `complete()` method creates an object with all the fields populated as a standard fixture. It is used in 90% of the use cases that we have in our application. The next day, a bug is discovered in one of those use cases that is caused by the general fixture. We change the `complete()`, and we get out tests failing and other non-related 50 tests also failed. The `complete().toBuilder()` is available to override default values but then we have to learn and understand the object state to exclude in the current test.

I think you see the _Mystery Guest_ here, it's hard to understand what is being built. There are no clues about the object's properties state and as a reader you have to jump outside the test context and learn the mystery object.

```.id(null).field1(null).createdBy(null).createdDate(null).lastModifiedBy(null).lastModifiedDate(null)```

This clearly means the Standard Fixture_ is not what you want. You build way too much for your test, and you are forced to nullify/revert some standard actions on the object.

Let's stop here, even tho we can discuss a lot on the code above.

## Root causes & theory

### Fixture Strategy Management

It all comes from the test fixture management strategy that has a large impact on the execution time and robustness of the tests. The effects of picking the wrong strategy won’t be felt immediately because it takes at least a few hundred tests before the _Slow Tests_ smell becomes evident and probably several months of development before the _High Test Maintenance Cost_ smell starts to emerge.

### Symptoms

The symptom is that each of the **failed** test builds a larger fixture than it should be. Each failed test builds much more that it would appear to be necessary in that test. It is also hard to understand the relationship between the fixture, the SUT and the expected result. We try to create a standard fixture that solves all the current and future use uses of the application. The more diverse the needs of those tests, the more likely we will end up in a _General
Fixture_. In a sense, _Standard Fixture_ is the result of **Big Design Upfront** of the test fixture for a whole suite of tests. 

## Impact

This pattern results in a large fixture that grows over time, and it is difficult to understand. It is difficult to understand how each test uses the fixture. The complexity of the fixture violates the _Tests as Documentation_ goal. It can also cause a _Fragile Fixture_/_Fragile Test_ as **people continue to alter the fixture** so that it can handle new tests. It can also enable _Slow Tests_ because a larger fixture takes more time to build, especially if a file system of
a database gets in the scene.

#### Martin Fowler's vision quoted by Gerard

> When I was reviewing an early draft of this book with Series Editor Martin Fowler, he asked me, “Do people actually do this?” This question exemplifies the philosophical divide of fixture design. Coming from an agile background, Martin
> lets each test pull a fixture into existence. If several tests happen to need the same fixture, then it makes sense to factor it out into the setUp method and split the class into one **_Testcase Class per Fixture_***. It doesn’t even occur to Martin to design a Standard Fixture that all tests can use. So who uses them?

In the xUnit community, use of a _Standard Fixture_ simply to avoid designing a _Minimal Fixture_ for each test is considered **undesirable** and has been given the name General Fixture.

## Testcase Class per what? - Fixture Strategy Management

##### How do we organize our Test Methods onto Testcase Classes?

There are a couple of ways to organize tests. This will change over the life of our project. A method is part of a _Testcase Class_. Should we put all our _Test Methods_ onto a single _Testcase Class_ for the application? Or should we create a _Testcase Class_ for each _Test Method_? Of course, the right answer lies somewhere between these two extremes, and it will change over the life of our project.

* Testcase Class per Class
* Testcase Class per Feature
* Testcase Class per Fixture
* Testcase Super class
* Test Helper

### Testcase Class per Class
##### We put all the Test Methods for one SUT class onto a single Testcase Class
![TestcaseByClass.png](images/TestcaseByClass.png)
When we write our first few _Test Methods_, we can put them all onto a **single** _Testcase Class_. As the number of _Test Methods_ increases, we will likely want to split the _Testcase Class_ so that one _Testcase Class per Class_ is tested, which reduces the number of _Test Methods_ per class . As those Testcase Classes get too big, we usually split the classes further(do we?). In that case, we need to decide which Test Methods to include in each Testcase Class.

### Testcase Class per Feature
##### We group the Test Methods onto Testcase Classes based on which testable feature of the SUT they exercise.
![TestcaseByFeature.png](images/TestcaseByFeature.png)
As the number of _Test Methods_ increases, we must decide which _Testcase Class_ to assign each _Test Method_, impacting our ability to grasp the overall test structure. It also influences our fixture setup approach. Using a Testcase Class per Feature allows us to divide a large _Testcase Class_ into smaller ones systematically, without modifying the _Test Methods_.

This approach is suitable when we have a substantial number of Test Methods and want to emphasize the specification of each SUT feature. However, it doesn't simplify or enhance the understanding of individual _Test Methods_; only _Testcase Class per Fixture_ accomplishes that. Additionally, if each SUT feature only requires one or two tests, it's more practical to stick with a single _Testcase Class per Class_. Note that having a large number of features on a class is a
“smell” indicating the possibility that the class might have too many responsibilities.

### Testcase Class per Fixture
##### We organize Test Methods into Testcase Classes based on commonality of the test fixture.
![TestcaseByFixture.png](images/TestcaseByFixture.png)
An alternative perspective suggests grouping _Test Methods_ that share the same test fixture into one _Testcase Class per Fixture_. This allows for centralized fixture setup code in the setUp method, but may scatter test conditions across multiple _Testcase Classes_.

Organizing _Test Methods_ based on the required test fixture simplifies individual tests by utilizing Implicit Setup. The _Testcase Class per Fixture_ pattern is suitable when multiple _Test Methods_ need an identical fixture and emphasizes simplicity. However, if each test requires a unique fixture, this pattern becomes less practical, resulting in numerous single-test classes. In such cases, _Testcase Class per Feature_ or _Testcase Class per Class_ may be more appropriate.

An advantage of _Testcase Class per Fixture_ is its ability to easily identify if all operations from each starting state are being tested. This pattern aids in discovering _Missing Unit Tests_ prior to production, especially when viewing an outline or method browser in an IDE.

_Testcase Class per Fixture_ aligns with behavior-driven development and promotes concise test methods with a focus on a single assertion. Combined with descriptive test method names, this pattern facilitates tests serving as documentation. A side effect is an increased number of _Testcase Classes_, which can be grouped using nested folders, packages, or namespaces dedicated to these test classes.

### Testcase Super class
##### We group the Test Methods onto Testcase Classes based on which testable feature of the SUT they exercise.
![TestcaseSuperClass.png](images/TestcaseSuperClass.png)
When encountering the need for repeated logic in multiple tests, we can address it by utilizing Test Utility Methods. The question then arises: Where should these Test Utility Methods be placed?

One option is to use a _Testcase Superclass_ as a centralized location for _Test Utility Methods_. By subclassing this _Testcase Superclass_, we can reuse the utility methods across multiple _Testcase Classes_. This approach assumes that our programming language supports inheritance, there are no conflicting uses of inheritance, and the _Test Utility Method_ does not require access to specific types that are not visible from the _Testcase Superclass_. The same tells us _Kent Beck_ in his chapter "Fixture".
> Each new kind of fixture should be a new subclass of TestCase.

### Test Helper
##### We define a helper class to hold any Test Utility Methods we want to reuse in several tests.
![TestHelper.png](images/TestHelper.png)
We can use a **Test Helper** if we wish to share logic or variables between several **Testcase Classes** and cannot (or choose not to) find or define a **Testcase Superclass** from which we might otherwise subclass all tests that require this logic.

The decision between _Testcase Superclass_ and _Test Helper_ all comes down to *type visibility*. The client classes need to be able to see the _Test Utility Method_ and the _Test Utility Method_ needs to be able to see all the types and classes it depends on. When it doesn't depend on many types or when everything it depends on is visible from a single place, like a DTO in a package on the boundary of the application, the _Test Utility Method_ can be put into a common
_Testcase Superclass_ we define for that controller. If it depends on types/classes that cannot be seen from a single place that all the clients can see, it may be necessary to put it on a _Test Helper_ in the appropriate test package or subsystem.

#### Variation: Object Mother

The xUnit book mentions multiple variations of the _Test Helper_. But I'd like to pay attention a bit to the variation:
_Object Mother_.
In my opinion, this particular topic is tricky.
Gerard Meszaros says:
> The Object Mother pattern is simply an aggregate of several other patterns, each of which makes a small but
> significant contribution to making the test fixture easier to manage. The Object Mother consists of **one or more Test
Helpers** that provide **Creation Methods** and **Attachment Methods** , which our tests then use to create ready-to-use
> test fixture objects. Object Mothers often provide several **Creation Methods** that create instances of the same class,
> where each method results in a test object in a **different starting state**. Because there is no single, crisp
> definition of what someone means by “Object Mother,” it is advisable to refer to the individual patterns when referring
> to specific capabilities of the Object Mother.

Martin says (https://martinfowler.com/bliki/ObjectMother.html):
> The first move is to create fixture in the setup method of an xunit test - that way it can be reused in multiple
> tests. But the trouble with this is often you need similar data in multiple test classes. At this point it makes sense
> to have a factory object that can return standard fixtures.

Here "stardard fixture" contradicts the "reuse the design of the text fixture across the many tests" from xUnit. But then Martin acknowledges that _Object Mother_ will couple your code to the standard fixture.

> Object Mothers do have their faults. In particular there's a heavy coupling in that many tests will depend on the
> exact data in the mothers. As a result it's tricky should you want to change that standard data for any reason.

http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.18.4710&rep=rep1&type=pdf
> The purpose of the pattern is to generate business objects that resemble as closely as possible actual objects that
> will exist in production.

In my opinion, _Object Mother_ is the last resort of creation an object in a valid state. It should create the object in a minimal valid state for the test.
I see some disadvantages in _Object Mother_ pattern:

* Each time developers want a new different variation of data, a new factory method is created, making the Object Mother big and hard to learn.
* Sometimes developers use the creation methods to create on object for all use cases like `ObjectMother.complete()` or `ObjectMother.someObject()`. This return the object with all the fields populated, but it is unclear in what state objects are created without diving in the external class. Thus these methods end up being used across the code base.
* In an active development, the model changes over a day as well as _Mother Objects_. Thus, we would have to maintain additional classes and the correctness of the business objects created.
  As the original [paper](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.18.4710&rep=rep1&type=pdf) says, we would have
  potentially to sign off the real objects with the analysts, if we have this luxury.
    * > Because of this, on one of our projects, we actually had developers pairing with analysts in order to ensure
      that the objects being generated by the pattern were as close as possible to the real thing.
* Looking the [ObjectMother - Easing Test Object Creation in XP](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.18.4710&rep=rep1&type=pdf) authors work with plain POJOs, default constructors, setters and getter. While the process of creating an object can indeed be extracted in factory methods, attachment methods, in my opinion, and some pieces of code could be part of the constructor with required arguments, as well as attachment methods may be replaced with [rich domain model vs anemic model](https://martinfowler.com/bliki/AnemicDomainModel.html).
* An alternative to _Object Mother_ maybe be the **Test Data Builders** idea from **Growing Object-Oriented Software, Guided by Test**

### Choosing a Test Method Organization Strategy

Clearly, there is no single **best practice** we can always follow. The best practice is the one that is most appropriate for the particular circumstance.

_Testcase Class by Class_ is very popular and the default way of grouping tests. When we have a lot of _Mockito_ stubs in each test method, and it grows, and we tend to copy/paste the stubs in each test, any you cannot push them on the class level because you will start guessing which is used when, its clear that something is not good, we have multiple fixtures in one class, we have to take action.

_Testcase Class per Fixture_ is commonly used when we are writing unit tests for stateful objects and each method needs to be tested in each state of the object.

_Testcase Class per Feature_ is more appropriate when we are writing customer tests against a Service Facade; It enables us to keep all the tests for a customer-recognizable feature together. When each test needs a slightly different fixture, the right answer may be to select the _Testcase Class per Feature_ pattern.

## My conclusion

A commonly accepted practice is the use of _Implicit Setup_ in conjunction with **_Testcase Class per Fixture_**. This approach is suitable when only a few Test Methods share the same fixture design because they require the same setup. In such cases, utilizing a _Minimal Fixture_ can be advantageous to avoid the unnecessary overhead associated with creating objects that are only needed in other tests.

**_A Minimal Fixture_ focuses on using the smallest and simplest fixture possible for each test**. By keeping the fixture small and simple, tests become easier to understand compared to fixtures that include unnecessary or irrelevant information. The concept of a _Minimal Fixture_ plays a crucial role in achieving _Test as Documentation_. To determine if an object is necessary as part of the fixture, one can try removing it. If the test fails as a result, it indicates
that the object was likely necessary in some way.

I would start with a _Minimal Fixture_ in my tests. At the first iteration, I will set up _Testcase per Class_. Then, if it grows, multiple `Mockito.given().thenReturn()` (mock mock mock 🦆) and different stubbing occur in each test, I will **refactor test code base** into _Testcase per Fixture_.

Having the _Testcase per Fixture_, I will share common utility methods in an _Testcase Super class_ shared across the classes of that use case. If the use case has negative flows, I would move some common methods in the super class apply. 

I can share the _ObjectMother_ but not as a global static method. It should be either package private limited to the suite of tests. _ObjectMother_ should have proper method names as in [paper](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.18.4710&rep=rep1&type=pdf). You should be able to combine different objects with different attachment methods.

For object creation, I consider the constructors with required arguments. If the object state grows and differs, and the telescopic constructors emerge, then I will use builder pattern with internal validations in `build()`.

I would think twice if I have to extract it in a _Test Helper_ aka _Object Mother_. I rekon there is code duplication, "But there are different kinds of duplication". There is an idea from "Clean Architecture" from Uncle Bob I kinda like:
> Then there is false or accidental duplication. If two apparently duplicated sections of code evolve along different
> paths—if they change at different rates, and for different reasons—then they are not true duplicates. Now imagine two
> use cases that have very similar screen structures. The architects will likely be strongly tempted to share the code for
> that structure. But should they? Is that true duplication? Or it is accidental?

We are at the cross roads if we have to share the same data structure and state or not. We either might fall into a standard/general
fixture setup for 2 use cases, or we have code duplication in both use cases. Choose wisely..

How i'd have it:
```java
package project.space.earth.feature1;

public class TestClass1 {
    
    //Here we can use social tests, test containers if this is backed by database.
    @Test
    void test1() {
        Foo stub = Foo.builder().id().build();
        given(service.createFoo(any())).willReturn(stub); //stub

        Foo inputFoo = newFooComposition(); //minimal fixture
        Foo actual = sut.insert(inputFoo); //exercise

        //assertThat(actual).usingRecursiveComparison(recursiveComparisonConfiguration).isEqualTo(stub); //verify
        //In this line we assert multiple things:
        //1. Foo was passed correctly to service (behaviour assertion)
        //2. Returned Foo is has correct state (state assertion)
        //Will leave this for the next time to discuss when I am using behaviour or state assertion. It is not part of the fixture.
    }

    private static Foo newFooComposition() {
        Bar bar = Bar.builder().build();
        Tee tee = Tee.builder().build();
        return new Foo.builder().bar(bar).tee(tee).build();
    }
}

package project.space.in.another.part.of.solar.system.feature2;
public class TestClass2 {

    @Test
    void test2() {
        given(dependency.action(any(), any())).willReturn(new Object());//stub

        Object object = new Object();
        Bad bar = Bar.toBuilder().attachObject(object).build(); //minimal fixture
        
        Foo existingFoo = new Foo(); //minimal fixture
        final Foo actual = sut.action(existingFoo, bar); //exercise

        //assertThat(actual.getObject()).isEqualTo(object); //verify
        //Lets discuss assertions next time
    }
    
}

```