---
title: "General Fixture"
date: 2023-06-10T21:23:48+03:00
draft: false
---

## tltr
Build Minimal.

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

Pseudocode
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

This clearly means the _Standard Fixture_ is not what you want. You build way too much for your test, and you are forced to nullify/revert some standard actions on the object.

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

In the xUnit community, use of a _Standard Fixture_ **simply to avoid designing** a _Minimal Fixture_ for each test is considered **undesirable** and has been given the name General Fixture.

## My conclusion

A commonly accepted practice is the use of _Implicit Setup_ in conjunction with **_Testcase Class per Fixture_**. This approach is suitable when only a few Test Methods share the same fixture design because they require the same setup. In such cases, utilizing a _Minimal Fixture_ can be advantageous to avoid the unnecessary overhead associated with creating objects that are only needed in other tests.

**_A Minimal Fixture_ focuses on using the smallest and simplest fixture possible for each test**. By keeping the fixture small and simple, tests become easier to understand compared to fixtures that include unnecessary or irrelevant information. The concept of a _Minimal Fixture_ plays a crucial role in achieving _Test as Documentation_. To determine if an object is necessary as part of the fixture, one can try removing it. If the test fails as a result, it indicates
that the object was likely necessary in some way.

I would start with a _Minimal Fixture_ in my tests. At the first iteration, I will set up _Testcase per Class_. Then, if it grows, multiple `Mockito.given().thenReturn()` (mock mock mock 🦆) and different stubbing occur in each test, I will **refactor test code base** into _Testcase per Fixture_.


![minimal-fixture.png](images/minimal-fixture.png)

How i'd have it, this is a pseudocode, ignoring assertions:
```java
package project.space.earth.feature1;

public class TestClass1 {
    
    @Test
    void test1() {
        Foo stub = Foo.builder().id().build(); //minimal required fields
        given(service.createFoo(any())).willReturn(stub); //minimal stub

        Foo inputFoo = newFooComposition(); //minimal fixture
        Foo actual = sut.insert(inputFoo); //exercise

        //...
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
        given(dependency.action(any(), any())).willReturn(new Object());//minimal stub

        Object object = new Object();
        Bad bar = Bar.toBuilder().attachObject(object).build(); //minimal fixture
        
        Foo existingFoo = new Foo(); //minimal fixture
        final Foo actual = sut.action(existingFoo, bar); //exercise
        
        //...
    }
    
}

```