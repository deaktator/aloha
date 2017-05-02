---
layout: docs
title: Documentation
---

# An In-Depth Walkthrough

There are a few basic concepts that carry through all aspects of Aloha.  Those are, namely, *features* and *semantics*. 


## Feature

Features are the basic unit when creating dataset and model specifications.  Features can be thought of as 4-tuples 
of the form:

>
> ( Specification, Default, Feature Domain, Feature Codomain )
>

1. The specification (or *spec*) is the actual statement, body or definition of what the feature does.
1. The default is the value that a feature produces when a value can't otherwise be produced.
   A default is not required when all parameters in the spec are required but is required if any of the parameters 
   in a spec are optional (more on this later).
1. Feature [domain](https://en.wikipedia.org/wiki/Domain_of_a_function).
1. Feature [codomain](https://en.wikipedia.org/wiki/Codomain).


### A Feature Example: BMI

An example of a feature is Body Mass Index (BMI) which is defined as 

>
>  BMI = 703 * (weight in lbs.) / (height in inches)<sup>2</sup>
>

Assuming we have a hierarchical datastructure that has a weight and height field, we could represent this as 

```scala
703 * ${weight} / pow(${height}, 2)
```

where the parameter values to be extracted are inside of the delimiters "`${`" and "`}`".  The rest of the feature is just 
normal [scala](http://scala-lang.org) code (with some import statements previously defined).  Notice this specification 
is very similar to how expressions with [parameter substitutions](http://www.tldp.org/LDP/abs/html/parameter-substitution.html) 
are defined in [bash scripts](http://www.tldp.org/LDP/abs/html/).  In fact, default values are provided in the same way
as in bash.  For instance,  if the `weight` parameter were optional and the particular value was missing, we can provide 
a default value as follows:

```scala
${weight:-200}
```

Which means, like in bash, that when weight is not declared or has been declared as null, the default of *200* will be 
used.  The difference here is that the default value has to type check with the original value.  If for instance 
`weight` were a 32-bit float, then the default couldn't be `200.5` (which is a 64-bit double), but would have to 
be `200.5f`.
  
So far, we've already run into a few snags.  If `weight` was optional and the value wasn't provided, then our expression 
if naively written, could produce [NPEs](http://docs.oracle.com/javase/8/docs/api/java/lang/NullPointerException.html).  
Aloha avoids these errors by treating optional values appropriately.  It ensures that optional parameters are 
represented by [`scala.Option`](http://scala-lang.org/api/current/#scala.Option) values.

> The key insight is that each feature can be thought of as an 
> [applicative](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html#applicatives)
> that operates over many parameters lifted into the Option context.  This way, features can be written without worrying
> about the boring details like null values, etc.

This insight, along with scala's [type inference](https://twitter.github.io/scala_school/type-basics.html#inference) 
abilities allows us to synthesize strongly typed code without having to specify parameter types.  We only need to know 
the input and output types and need to know how to generate the code that extracts parameter values from the input type 
(see plugins below).   Let's revisit our example to go into more detail.  So the example above can be decomposed in 
the two parameters: 

- height: required int
- weight: optional float

and the function of these two parameters. Let's say that we could write scala functions to extract the height and width
from the input type.  For simplicity, let's say we are working with a `Map[String, Any]`.  Then, our functions could 
be written like the following: 

```tut:silent
val weight = (x: Map[String, Any]) => x get { "weight" } collect { case x: Float => x }  // Optional value.
val height = (x: Map[String, Any]) => x("height").asInstanceOf[Int]                      // Required value.
```

Now let's say that Aloha has a generic way of combining the return values of two functions (representing parameters) 
using a third combining function.  We could easily write this as: 

```tut:silent
/**
 * @param u a function with domain X and codomain U 
 * @param v a function with domain X and codomain V 
 * @param f a function with domain U x V and codomain Y
 * @tparam X the domain of the returned function  
 * @tparam U the first intermediate type.
 * @tparam V the second intermediate type.
 * @tparam Y the codomain of the returned function 
 * @return a function that takes a value x &isin; X and return f(u(x), v(x))
 */
def functionWithTwoVariables[X, U, V, Y](u: X => U, v: X => V)(f: (U, V) => Y) = 
  (x: X) => f(u(x), v(x))
```

Then we could construct the BMI function easily from the definitions `weight` and `height` using the following: 

```tut:silent
// functions w and h are defined above.
// Type of bmi:   Map[String,Any] => Option[Double]
val bmi = functionWithTwoVariables(weight, height)((weightOpt, height) => {
  import math.pow
  weightOpt.map { weight => 
    703 * weight / pow(height, 2)
  } 
})
```

Notice that once we defined `weight` and `height` above, we didn't need to supply any type information.  We just 
needed to know that the function `weight` returned an optional value.  Also notice that the body is identical 
to how the feature looked in the original specification.

This begs the question, how is this possible?  The answer is the [semantics](https://en.wikipedia.org/wiki/Semantics) 
implementation.


## Semantics

Simply put, the semantics is the thing that gives meaning to the features.  It is the interpreter that takes a 
feature specification and a default value and turns this tuple into working functions that extract and process data.

For instance, in the BMI example used above, the semantics was responsible for turning the statement
 
```scala
703 * ${weight} / pow(${height}, 2)
```

into a function with domain `Map[String, Any]` and codomain `Option[Double]`.

The semantics has additional responsibilities including knowing about the domain of the features.  The API contract
for semantics is rather vague.  It's up to the individual implementations of semantics to define how pluggable it 
wants to be. 


### Compiled Semantics

Aloha currently has one main type of Semantics, namely 
[CompiledSemantics](aloha-core/scaladocs/index.html#com.eharmony.aloha.semantics.compiled.CompiledSemantics). CompiledSemantics 
is extremely powerful because it allows:

1. the use of any arbitrary scala code 
1. importing code defined inside *OR* outside the Aloha library

The CompiledSemantics is responsible for transforming the feature specification to a function but it delegates 
to a plugin for the synthesis of the code responsible for the parameter extraction.  This will become clear soon,
but before continuing, a brief note.

#### A Note on the Power of Imports

It is important to note that the ability to import is **extremely powerful**!  By allowing imports, Aloha can 
easily be extended outside the library.  Imports provide the power analogous to 
[user-defined functions](https://en.wikipedia.org/wiki/User-defined_function) (UDFs) in databases, but imports 
are actually much more powerful due to scala's facility for 
[implicit parameters](http://daily-scala.blogspot.com/2010/04/implicit-parameters.html).


>
> Imports can do more than just add language features. They can *change the meaning of the language*.
>


Imports are specified globally in CompiledSemantics, away from where the features are specified. This is by design 
so that the same specifications can have different meaning in different problem domains.  The consequence is that 
imports need to be used with caution, especially when referencing implicit language features.  The following example 
illustrates this.  

Imagine a third party developer wrote the following code 

```scala
package com.fictitious.company

object Ops {
  /**
   * @param v an explicit value
   * @param a an implicit value to add to v 
   * @param ring A type class representing an algebraic ring.
   * @tparam A the type of algebraic ring
   * @return ''v'' added to some value ''a'' in the implicit scope
   */
  def add[A](v: A)(implicit a: A, ring: Numeric[A]) = ring.plus(v, a)
}

object Implicits {
  implicit val two: Int = 2 
  implicit val four: Int = 4
}
```

Let’s assume a feature specification: `add(${some.value})` and the parameter 
`${some.value}` has an integer value, *1*. If the imports were: 

- `com.fictitious.company.Ops.add`
- `com.fictitious.company.Implicits.two`

Then the output of the feature would be *3* in this instance.  If we changed the imports to 

- `com.fictitious.company.Ops.add`
- `com.fictitious.company.Implicits.four`

then the **same feature specification** would yield a value of *5*.  While this can be extremely useful, it can also
be extremely dangerous.  It is imperative to understand the imports being imported into scope.


### Protobuf Plugin 

Currently there are a few main CompiledSemantics plugins.  The first is
[CompiledSemanticsProtoPlugin](/aloha/docs/api/index.html#com.eharmony.aloha.semantics.compiled.plugin.proto.CompiledSemanticsProtoPlugin) located
in the `com.eharmony.aloha.semantics.compiled.plugin.proto` package in `aloha-io-proto`. This
plugin allows for arbitrary kinds of [Protocol Buffer](https://developers.google.com/protocol-buffers/) instances 
that extend [GeneratedMessage](https://developers.google.com/protocol-buffers/docs/reference/java/com/google/protobuf/GeneratedMessage).

See [dependencies](aloha-core/dependencies.html) for the version of protocol buffers used in Aloha.  At the time of 
this writing, Aloha uses protocol buffers 2.4.1.

```tut:silent
// Scala Code: Constructing a compiled semantics protocol buffer plugin is easy.
import com.eharmony.aloha.semantics.compiled.plugin.proto.CompiledSemanticsProtoPlugin
import com.eharmony.aloha.test.proto.TestProtoBuffs.Abalone

val plugin = CompiledSemanticsProtoPlugin[Abalone]

// pass to compiled semantics ... (more on this later)
```

### Avro Plugin

Aloha Also supports Avro input and output.  The support for this in located in the `aloha-io-avro` module.

```tut:silent
import java.io.File
import org.apache.avro.{Protocol, Schema}
import org.apache.avro.generic.GenericRecord
import com.eharmony.aloha.semantics.compiled.plugin.avro.CompiledSemanticsAvroPlugin

// Get the Avro Schema.
val alohaRoot = new File(new File(".").getAbsolutePath.replaceFirst("/aloha/.*$", "/aloha/"))
val protocolFile = new File(alohaRoot, "aloha-io-avro/src/test/resources/avro/test.avpr")
val protocol = Protocol.parse(protocolFile)
val schema = protocol.getType("Test")

// Pass the Avro Schema to plugin.  Need to specify the type associated with the Schema.
val plugin = CompiledSemanticsAvroPlugin[GenericRecord](schema)

// pass to compiled semantics ... (more on this later)
```


#### Future structured input datatypes Aloha may support
 
One of the niceties of Aloha is that it allows models to accept structured input in a typesafe way.  Currently, this 
is limited to protocol buffers but we'd like to extend allow to integrate with other serialization protocols such as

* [Thrift](https://thrift.apache.org)


### CSV Plugin 

Another kind of CompiledSemantics plugin currently in use is the

[CompiledSemanticsCsvPlugin](/aloha/docs/api/index.html#com.eharmony.aloha.semantics.compiled.plugin.csv.CompiledSemanticsCsvPlugin) located
in the `com.eharmony.aloha.semantics.compiled.plugin.csv` package in `aloha-core`. This plugin
allows the user to specify [Delimiter-separated values](https://en.wikipedia.org/wiki/Delimiter-separated_values) 
with lots of different underlying input types.  This plugin takes string-based data and turns it into strongly typed 
data that can encode fields with types: 

- Booleans
- Enumerated Type Values (Categorical Variables)
- 32-bit and 64-bit integers
- 32-bit and 64-bit floating point numbers
- Strings
- Vectorized inputs
- Optional inputs

Specifying the plugin is easy.  It just requires providing a column name to column type map.  The types come from
[com.eharmony.aloha.semantics.compiled.plugin.csv.CsvTypes](aloha/docs/api/index.html#com.eharmony.aloha.semantics.compiled.plugin.csv.CsvTypes$).

```tut:silent
// Scala Code
import com.eharmony.aloha.semantics.compiled.plugin.csv.CompiledSemanticsCsvPlugin
import com.eharmony.aloha.semantics.compiled.plugin.csv.CsvTypes.{IntType, FloatOptionType}

val plugin = CompiledSemanticsCsvPlugin(
  "height" -> IntType,
  "weight" -> FloatOptionType
)
```

Notice that since [CompiledSemanticsCsvPlugin](/aloha/docs/api/index.html#com.eharmony.aloha.semantics.compiled.plugin.csv.CompiledSemanticsCsvPlugin) extends
[CompiledSemanticsPlugin](/aloha/docs/api/index.html#com.eharmony.aloha.semantics.compiled.CompiledSemanticsPlugin)\[[CsvLine](/aloha/docs/api/index.html#com.eharmony.aloha.semantics.compiled.plugin.csv.CsvLine)\],
the features produced by a [CompiledSemantics](/aloha/docs/api/index.html#com.eharmony.aloha.semantics.compiled.CompiledSemantics) instance
parameterized by an [CompiledSemanticsCsvPlugin](/aloha/docs/api/index.html#com.eharmony.aloha.semantics.compiled.plugin.csv.CompiledSemanticsCsvPlugin) operate
on instances of [CsvLine](/aloha/docs/api/index.html#com.eharmony.aloha.semantics.compiled.plugin.csv.CsvLine).  It
is therefore necessary to produce instances of CsvLine to make use of this plugin.  This is pretty easy to do though.

```tut:silent
import com.eharmony.aloha.semantics.compiled.plugin.csv.CsvLines

// Same as:
//   val tsvLinesWithDefaults = CsvLines(Map("height" -> 0, "weight" -> 1))
val tsvLines = CsvLines(
  Map(                     // 0-based field indices
    "height" -> 0, 
    "weight" -> 1
  ),
  Map.empty,               // enums: Map[String, Enum] 
  "\t",                    // fs (field separator) 
  ",",                     // ifs (intra-field separator, for vectorized inputs)
  (s: String) => s == "",  // missingData function 
  false,                   // errorOnOptMissingField 
  false                    // errorOnOptMissingEnum
)
```

Once in possession of a `CsvLines` instance, it's easy to generate `CsvLine` instances:

```tut:silent
import com.eharmony.aloha.semantics.compiled.plugin.csv.CsvLine

val csvLines = CsvLines(Map("height" -> 0, "weight" -> 1), fs = ",")

val oneCsvLine: CsvLine = csvLines("72, 175") // height = 72 inches, weight = 175 lbs.
assert(72 == oneCsvLine.i("height") && Some(175f) == oneCsvLine.of("weight"))

val manyCsvLinesUsingVarArgs: Vector[CsvLine] = csvLines("72, 175", "70, 180", "64, 110")
val csvLineIterator: Iterator[CsvLine] = csvLines(Iterator("72, 175", "70, 180", "64, 110"))
val csvLineVector: Seq[CsvLine] = csvLines(Seq("72, 175", "70, 180", "64, 110"))
val nonStrictVectorOfCsvLine = csvLines.nonStrict(Vector("72, 175", "70, 180", "64, 110"))
```

There are many ways to get one or multiple lines CSV lines.  These methods, found in
[CsvLines](aloha/docs/api/index.html#com.eharmony.aloha.semantics.compiled.plugin.csv.CsvLines),
are very general and
have [strict](https://en.wikipedia.org/wiki/Evaluation_strategy#Strict_evaluation) and
[non-strict](https://en.wikipedia.org/wiki/Evaluation_strategy#Non-strict_evaluation) versions.
These methods transform collections passed to them and should retain the collection shape
appropriately.


## Factory

Factories are important to create models: 

```tut:silent
// Scala Code
import com.eharmony.aloha.audit.impl.OptionAuditor
import com.eharmony.aloha.factory.ModelFactory
import com.eharmony.aloha.semantics.compiled.CompiledSemantics
import com.eharmony.aloha.semantics.compiled.compiler.TwitterEvalCompiler
import com.eharmony.aloha.semantics.compiled.plugin.csv.CompiledSemanticsCsvPlugin
import com.eharmony.aloha.semantics.compiled.plugin.csv.CsvTypes.{IntType, FloatOptionType}
import java.io.File
import scala.concurrent.ExecutionContext.Implicits.global

/** @return a cache directory if possible */
def cacheDir(): Option[File] = {
  val f = new File("/tmp/aloha/cache")
  if (f.exists)
    if (f.isDirectory)
      Option(f)
    else None
  else if (f.mkdirs)
    Option(f)
  else None
}

// vvvvv  Define the semantics  vvvvvvvvvvvvvvvvvvvvvv

// -----  Define the plugin  -------------------------
val plugin = CompiledSemanticsCsvPlugin(
  "profile.user_id" -> IntType
)
// -----  Define the plugin  -------------------------


// Imports allow extensions to the language.  These can be anything on
// the classpath, and can be defined outside of Aloha.  Importing BasicFunctions
// is not required but it's like Aloha's version of Predef to make things work
// better / more easily.
val imports = Seq("scala.math._", "com.eharmony.aloha.feature.BasicFunctions._")

// Compiled semantics is the most general.  It requires a compiler, a plugin
// and a sequence of imports.
val semantics = CompiledSemantics(TwitterEvalCompiler(classCacheDir = cacheDir()), plugin, imports)
// ^^^^^  Define the semantics  ^^^^^^^^^^^^^^^^^^^^^^

// Create an Auditor
val auditor = OptionAuditor[Double]()

// Create a model factory.  It requires a semantics to make sense of features
// and an auditor to report predictions in the proper container.
val factory = ModelFactory.defaultFactory(semantics, auditor)

val modelFile = {
  val alohaRoot = new File(new File(".").getAbsolutePath.replaceFirst("/aloha/.*$", "/aloha/"))
  new File(alohaRoot, "aloha-core/src/test/resources/fizzbuzz.json")
}

// type: scala.util.Try[com.eharmony.aloha.models.Model[CsvLine, Double]]
val modelAttempt = factory.fromFile(modelFile)
val model = modelAttempt.get
```

Once we have a model, we can create some input data and run the model.  So, based on this data:

```tut:silent
val csvLines = CsvLines(Map("profile.user_id" -> 0))
val lines = csvLines(10 to 15 map (i => i.toString))
```

we can use the model to predict.  Since the model is just a function, this is rather easy.

```tut
val predictions = lines map (x => model(x)) // or just 'lines map model'

val inputs = lines map (x => x.i("profile.user_id"))
(inputs zip predictions).flatMap { case (x, p) => p.map(y => (x, y)) }.mkString("\n")
```

For more information on creating semantics and factories, see the [Creating Semantics](/aloha/docs/creating_semantics.html) page.
