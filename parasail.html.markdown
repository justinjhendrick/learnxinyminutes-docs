---
language: ParaSail
contributors:
    - ["Justin Hendrick", "https://github.com/justinjhendrick"]
filename: learnparasail.psl
---

ParaSail is a work in progress language built for parallel and high integrity
software. It has both implicit and explicit parallelism
and compiler checked Pre/Post conditions

Not all features mentioned here are fully implemented. See the release notes
in the source on the [website](http://parasail-lang.org).

```
//  This is a comment

//  1. Basics

//  Functions
func Add(X : Univ_Integer; Y : Univ_Integer) -> Univ_Integer is
   return X + Y;
end func Add;
//  End of line semi-colons are optional
//  +, +=, -, -=, *, *=, /, /=
//  all do what you'd expect (/ is integer division)

//  If you find Univ_Integer to be too verbose you can import Short_Names
//  which defines aliases like Int for Univ_Integer and String for Univ_String
import PSL::Short_Names::*, *

func Greetings() is
   const S : String := "Hello, World!"
   Println(S)
end func Greetings
//  All declarations are 'const', 'var', or 'ref'
//  Assignment is :=, equality checks are ==, and != is not equals

func Boolean_Examples(B : Bool) is
   const And := B and #true           //  Parallel execution of operands
   const And_Then := B and then #true //  Short-Circuit
   const Or := B or #false            //  Parallel execution of operands
   const Or_Else := B or else #false  //  Short-Cirtuit
   const Xor := B xor #true
end func Boolean_Examples
//  Booleans are a special type of enumeration
//  All enumerations are preceded by a sharp '#'

func Fib(N : Int) {N >= 0} -> Int is
   if N <= 1 then
      return N
   else
      //  Left and right side of '+' are computed in Parallel here
      return Fib(N - 1) + Fib(N - 2)
   end if
end func Fib
//  '{N >= 0}' is a precondition to this function
//  Preconditions are built in to the language and checked by the compiler

//  ParaSail does not have mutable global variables
//  Instead, use 'var' parameters
func Increment_All(var Nums : Vector<Int>) is
   for each Elem of Nums concurrent loop
      Elem += 1
   end loop
end func Increment_All
//  The 'concurrent' keyword in the loop header tells the compiler that
//  iterations of the loop can happen in any order.
//  It will choose the most optimal number of threads to use.
//  Other options are 'forward' and 'reverse'.

func Sum_Of_Squares(N : Int) -> Int is
   //  The type of Sum is inferred
   var Sum := 0
   for I in 1 .. N forward loop
      Sum += I ** 2 //  ** is exponentiation
   end loop
end func Sum_Of_Squares

func Sum_Of(N : Int; Map : func (Int) -> Int) -> Int is
   return (for I in 1 .. N => <0> + Map(I))
end func Sum_Of
//  It has functional aspects as well
//  Here, we're taking an (Int) -> Int function as a parameter
//  and using the inherently parallel map-reduce.
//  Initial value is enclosed with angle brackets

func main(Args : Basic_Array<String>) is
   Greetings()     //  Hello, World!
   Println(Fib(5)) //  5
   //  Container Comprehension
   var Vec : Vector<Int> := [for I in 0 .. 10 {I mod 2 == 0} => I ** 2]
   //  Vec = [0, 4, 16, 36, 64, 100]
   Increment_All(Vec)
   //  Vec = [1, 5, 17, 37, 65, 101]
   //  '|' is an overloaded operator.
   //  It's usually used for concatenation or adding to a container
   Println("First: " | Vec[1] | ", Last: " | Vec[Length(Vec)]);
   //  Vectors are 1 indexed, 0 indexed ZVectors are also available
   
   Println(Sum_Of_Squares(3))
   
   //  Sum of fibs!
   Println(Sum_Of(10, Fib))
end func main

//  Preceding a type with 'optional' allows it to take the value 'null'
func Divide(A, B, C : Real) -> optional Real is
   //  Real is the floating point type
   const Epsilon := 1.0e-6;
   if B in -Epsilon .. Epsilon then
      return null
   elsif C in -Epsilon .. Epsilon then
      return null
   else
      return A / B + A / C
   end if
end func Divide

//  2. Modules
//  Modules are composed of an interface and a class
//  ParaSail has object orientation features

//  modules can be defined as 'concurrent'
//  which allows 'locked' and 'queued' parameters
concurrent interface Locked_Box<Content_Type is Assignable<>> is
   // Create a box with the given content
   func Create(C : optional Content_Type) -> Locked_Box;

   // Put something into the box
   func Put(locked var B : Locked_Box; C : Content_Type);

   // Get a copy of current content
   func Content(locked B : Locked_Box) -> optional Content_Type;

   // Remove current content, leaving it null
   func Remove(locked var B : Locked_Box) -> optional Content_Type;

   // Wait until content is non-null, then return it, leaving it null.
   func Get(queued var B : Locked_Box) -> Content_Type;
end interface Locked_Box;

concurrent class Locked_Box is
   var Content : optional Content_Type;
exports
   func Create(C : optional Content_Type) -> Locked_Box is
      return (Content => C);
   end func Create;

   func Put(locked var B : Locked_Box; C : Content_Type) is
      B.Content := C;
   end func Put;

   func Content(locked B : Locked_Box) -> optional Content_Type is
      return B.Content;
   end func Content;

   func Remove(locked var B : Locked_Box) -> Result : optional Content_Type is
      // '<==' is the move operator
      // It moves the right operand into the left operand,
      // leaving the right null.
      Result <== B.Content;
   end func Remove;

   func Get(queued var B : Locked_Box) -> Result : Content_Type is
      queued until B.Content not null then
      Result <== B.Content;
   end func Get;
end class Locked_Box;

func Use_Box(Seed : Univ_Integer) is
   var U_Box : Locked_Box<Univ_Integer> := Create(null);
   //  The type of 'Ran' can be left out because
   //  it is inferred from the return type of Random::Start
   var Ran := Random::Start(Seed);

   Println("Starting 100 pico-threads trying to put something in the box");
   Println(" or take something out.");
   for I in 1..100 concurrent loop
      if I < 30 then
         Println("Getting out " | Get(U_Box));
      else
         Println("Putting in " | I);
         U_Box.Put(I);

         //  The first parameter can be moved to the front with a dot
         //  X.Foo(Y) is equivalent to Foo(X, Y)
      end if;
   end loop;

   Println("And the winner is: " | Remove(U_Box));
   Println("And the box is now " | Content(U_Box));
end func Use_Box;

```

## Further reading
Read the [Blog](http://parasail-programming-language.blogspot.com/) and the
[website](http://parasail-lang.org). Feel free to ask
for help on the [Google Group](https://groups.google.com/forum/
#!forum/parasail-programming-language)
