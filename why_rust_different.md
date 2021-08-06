as a new language,  rust gains lots of attractions, there're also some C/C++
projects begin to migrate to it(librsvg/curl/tor). Recently the most interesting
project is trying to bring rust to Linux kernel development,
what makes it interesting does not only because it's driven by Google
(It fund rust development in linux kernel, and using rust in their
home-developed fuchsia OS), but also that rust may become first alternative
programming language in kernel development, other than C.

Previously kernel development always written in C, there're some attempts
to bring C++ in, but never success. Although C++ can be seen as
C superset (from some ones' perspective), 
Linus personally hate it, he expressed many times "C++ is horrible language".
If followed the long [bring rust to kernel](https://lore.kernel.org/ksummit/CANiq72kF7AbiJCTHca4A0CxDDJU90j89uh80S3pDqDt7-jthOg@mail.gmail.com/)
mail list, we can find Linus still holds his opinion.

This mail list starts at 2021-06-25 (There're already some rounds of
discussion before), Linus keeps silent until 2021-07-07,
after Bart Van Assche brings up topic 'why C++ not in kernel'
 
 ```
 ...
 As a sidenote, I'm surprised that C++ is not supported for Linux kernel
code since C++ supports multiple mechanisms that are useful in kernel
code, e.g. RAII, lambda functions, range-based for loops, std::span<>
and std::string_view<>. Lambda functions combined with std::function<>
allow to implement type-safe callbacks. Implementing type-safe callbacks
in C without macro trickery is not possible.
```

and Linus came up with:
```
You'd have to get rid of some of the complete garbage from C++ for it
to be usable.

One of the trivial ones is "new" - not only is it a horribly stupid
namespace violation, but it depends on exception handling that isn't
viable for the kernel, so it's a namespace violation that has no
upsides, only downsides.

Could we fix it with some kind of "-Dnew=New" trickery? Yes, but
considering all the other issues, it's just not worth the pain. C++ is
simply not a good language. It doesn't fix any of the fundamental
issues in C (ie no actual safety), and instead it introduces a lot of
new problems due to bad designs.
```


Userspace(none-kernel) developers may be surprised by Linus' point,
most language (such as gc based java/c# or C++) design the library/runtime
to work "magically" when dealing with memory allocation(from
kernel developer's POV) . For example in C++

```C++
std::string temp;
temp.push_back("I_am_very_long_string");
```
there is no explicit allocation call , but implicitly allocate dynamic heap 
memory under the hood. Writing such code in userspace is no big deal, it just
said "I don't care about how memory came from,just give me 1MB memory, 
or I will throw Out-Of-Memory exception". 
most userspace programs don't care about it and just crash.  
this makes sense for the most scenario since they're designed to only work 
under "normal" conditions(say, with enough memory to finish the work), 
if such condition not satisfied anymore, there's very little it can do to recover.
Suppose that another leaking program consumed all the memory, leads your program OOM,
kill the leaking program is the only way to recover. But to do this, you still need memory 
allocation (to list and find leaking one through OS API,
thus may OOM again and not guaranteed to complete)

But this is not the case for kernel. Firstly there's no simple 
'just gives me 1MB memory' like malloc/new, 
kernel need to allocate different memory with different flags depends on 
current running context, and secondly if such allocation 
failed, you can not simply throw OOM exception and crash, not only because 
exception is treated as 'valueless' in kernel, but also because 
a sensible kernel should never crash even under rare OOM case.

Back to the mail list, Bart Van Assche shows some C++ code that override
new operator globally (I didn't post them here) to resolve the problem,
and Linus replied:


```
The point is, C++ really has some fundamental problems. Yes, you can
work around them, but it doesn't change the fact that it doesn't
actually *fix* any of the issues that make C problematic.

For example, do you go as far as to disallow classes because member
functions are horrible garbage? Maybe newer versions of C++ fixed it,
but it used to be the case that you couldn't sanely even split a
member function into multiple functions to make it easier to read,
because every single helper function that worked on that class then
had to be declared in the class definition.

Which makes simple things like just re-organizing code to be legible a
huge pain.

At the same time, C++ offer no real new type or runtime safety, and
makes the problem space just bigger. It forces you to use _more_
casts, which then just make for more problems when it turns out the
casts were incorrect and hid the real problem.

So no. We're not switching to a new language that causes pain and
offers no actual upsides.

At least the argument is that Rust _fixes_ some of the C safety
issues. C++ would not.
```

That's interesting argument, since Bart Van Assche suggested 

```
RAII, range-based for loops, std::span<> and std::string_view<>.
Lambda functions combined with std::function<>
```

can improve kernel development, but Linus rejected this ideal.


Miguel Ojeda (who posted 'support rust in kernel' patches) posted a sample 
that how easily C++ can introduce memory bugs without trigger any compiler
warning, even with modern standard.

```
The issue is that, even if people liked C++ a lot, there is little
point in using C++ once Rust is an option.

Even if you discuss "modern C++" (i.e. post-C++11, and even
post-C++17), there is really no comparison.

For instance, you mentioned `std::span` from the very latest C++20
standard; let's build one:

    std::span<int> f() {
        int a[] = { foo() };
        std::span<int> s(a);
        return s;
    }

Now anybody that accesses the returned `std::span` has just introduced
UB. From a quick test, neither Clang nor GCC warn about it. Even if
they end up detecting such a simple case, it is impossible to do so in
the general case.

Yes, it is a stupid mistake, we should not do that, and the usual
arguments. But the point is UB is still as easy as has always been to
introduce in both C and C++. In Rust, that mistake does not happen.
```

and followed :

```
The thing is, claims such as "C++ is as safe as Rust, you just need to
use modern C++ properly!" are still way too common online, and many
developers are unfamiliar with Rust, thus I feel we need to be crystal
clear (in a thread about Rust support) that it is a strict improvement
over C++, and not a small one at that.
```

Argues over different language are almost FUD topic, especially consider 
that C++ already being largely used by industry.Since it's compatible with C
and accepts buggy logic C code thus lead same bug, some practise rules
and new standards/libraries still being needed to make code less error-prone
on memory bugs, but one of rust's selling point is 'memory safety'( the span
example shown is about undefined behavior, but most problem undefined behavior
led in C/C++ are just memory bugs, thus I refer them as memory bugs), 
to most of our knowledge, memory safe language are almost same meaning to 
GC-based high level language, How can rust, being same low-level as C/C++ 
(to be used in kernel), while still can avoid such memory bugs ?
I think it's primary question of many readers.

Before answer that question, we may firstly find what's the issue that 
```
   makes C problematic
``` 
short answser is just memory safety, C is so bare-metal that 
it almost didn't forbid any incorrect 'semantic' code, for kernel development,
it makes low-level access easier, but also too relaxed to code correctly,
even for the most intelligent people who use it for years. Such bugs can be 
found by grep the kernel log, I grep it from 2.6.12-rc2(2005-Apr 1da177e4c3f)
 to about 5.13-rc6 (2021-Jun 6b00bc639) by

```bash
git log --grep=leak --grep=error --all-match --oneline | wc -l
```
got 4626 result, you may also grep for "double free" (2234) or
"use after free" (9987) to know how many memory safety bugs introduced 
and fixed in linux kernel. Some of these bugs are logic related, so
rewritten logic in other languages won't help, but lack of memory safety
in C is the cause of most of them, as a side note,
[Microsoft said](https://msrc-blog.microsoft.com/2019/07/16/a-proactive-approach-to-more-secure-code/)
```
~70% of the vulnerabilities Microsoft assigns a CVE each year continue to be memory safety issues
```
and analysis on libcurl shows:
```
19 of the last 22 vulnerabilities (since 2018) have C-induced memory unsafety as a cause.
```

These catalog of bugs share some common property: they're so "trivial",
even not being kernel developer, we can easily understand why these piece of 
code is incorrect (after reading the fix commit), and fix is very short
(such as add missing deallocate) and they're also so easy to happen,
and hard to be discovered without depth knowledge of logic/control flow,
If simply look around the code, nobody knows such control flow trigger some
bugs before reading the fix commit.

Here I pick a simple leak fix as example

```
619fee9eb13 

 net: fec: fix the potential memory leak in fec_enet_init()

    If the memory allocated for cbd_base is failed, it should
    free the memory allocated for the queues, otherwise it causes
    memory leak.

    And if the memory allocated for the queues is failed, it can
    return error directly.

```

Let's pick the important part out to see what happened:

code before fix:

```
        fec_enet_alloc_queue(ndev);
...
        cbd_base = dmam_alloc_coherent(&fep->pdev->dev, bd_size, &bd_dma,
                                       GFP_KERNEL);
        if (!cbd_base) {
               return -ENOMEM;
        }

```

code after fix:
```
        ret = fec_enet_alloc_queue(ndev);
        if (ret)
                return ret;

...

        cbd_base = dmam_alloc_coherent(&fep->pdev->dev, bd_size, &bd_dma,
                                       GFP_KERNEL);
        if (!cbd_base) {
                ret = -ENOMEM;
                goto free_queue_mem;
        }
...
free_queue_mem:
       fec_enet_free_queue(ndev);
       return ret;

```

Seems easy to understand, right ? Two allocations happened,
but forget to deallocate first one if the second failed.
But suppose similar logic need to allocate many times, remember which
variable needs to be deallocated became harder,
cleanup control flow also became hard to maintain,
it must be exactly same order as allocation
(otherwise can't be shared by jumped from different failure point),
developer needs to be very careful to keep them
in sync when modify logic, break this rule lead to leak/double free bugs.

So does C++ have improvement in this area?  Personally I think the answer is 
yes (Linus maybe not agree with it !)
RAII (as mentioned by Bart Van Assche), is C++ idom to address leak alike bugs,
put resource acquire/release in constructor/destructor since they'll be
run only once during object lifetime, such object auto manage the resource 
(usually as a stack variable which lifetime bind to syntax scope).
Similar mechanism is also borrowed by many other languages,
for example java 1.7 try-with-resource, C# 8 using, go defer function,
and also rust. Even [C commitee is looking into adding defer function to 
implement RAII like idom](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2542.pdf)
From my personal view, before rust , C++ approach is better than Java/C# or 
go defer(also including proposed C defer), I list my reasons here:

Java/C# forces developer to make resource type inherit particular interface 
to cooperate with the syntax, but since cleanup function are just 
public method exposed by type, developer are free to call these functions 
at their will, that force cleanup function to always bookmark 
"resource already cleanup flag" in logic to avoid double-free alike bugs.


Defer function (go or C) have semantic confusions : 
defer function may capture some variable for later cleanup, but what values
do the captured variable hold? Do they capture values at define point ,
or use most-written value ? [go prefer previous](https://tour.golang.org/flowcontrol/12)
so 
```go
    package main
    import "fmt"
    func main() {
         var a = "abc";
         defer fmt.Println("clean resource",a, "while running defer");
         a = "xyz";
         fmt.Println("resource is", a);
    }

```
will print 

```
resource is xyz
clean resource abc while running defer
```
lead a mismatch cleanup. Such policy implicit that
the variable which hold resource can't be changed to point to new resource after
defer function defined(otherwise you're cleaning incorrect resource),
or you have to define new defer function everytime when new resource is 
created and assigned.


C purposed defer function also have such confusions:

```
The results from 387 responses show a 2:1 preference for the value being read at the time the
deferred statements are executed (66.9%) rather than when the defer statement encountered (33.1%). One interesting thing to note is that of respondents who gave a rationale for their vote,
respondents known to have a strong C++ background seemed to gravitate towards using the
value when the defer statement is encountered, which suggests there may be some interesting
implicit bias.
A general solution to this problem is the introduction of lambdas to the C language [13] with both
copy and reference semantics. A more specific solution is to use a second defer statement
syntax which supplies an identifier list of variables to capture by value.

```
I think it's bad idea to bring 'reference capture in lambda' to C, will discuss later.

C++ RAII makes acquired resource as member,
so it has no confusion during cleanup, and more important, such abstraction
is straightforward, resource availability is exactly same as wrapper type 
lifetime, no longer need to statically bind to syntax scope (as others), you
can transfer it across scope boundary and manage it based on dynamic control
flow.

For example

``` C++
struct ComplexResource{
   .....
};

std::unique_ptr<ComplexResource> return_resource(){
   unique_ptr<ComplexResource> ret(new ComplexResource());
   .....
   return ret;
}

void use_resource(){
   unique_ptr<ComplexResource> resouce = return_resource();
   ...//use resource
   ...//at scope end,resource auto cleanup itself because unique_ptr cleanup its holding ComplexResource pointer
   ...//and ComplexResource auto cleanup its holding resource by ComplexResource destructor
}

```
use unique_ptr makes ComplexResource lifetime outlive scope where it constructed, 
while still hold auto management property.

but C++ RAII also have weakness, mainly due to semantic of C++ variable
and assignment. For example if ComplexResource is written as 
```
struct ComplexResource{
   void *resource;
   ComplexResource(){
    resource = allocate();
   }

   ~ComplexResource(){
    deallocate(resource);
   }
};
```
then
```
unique_ptr<ComplexResource> resouce = return_resource();
ComplexResource temp = *source
```
lead a double free, since C++ default to copy variable during assignment
and compiler generate a member-to-member copy logic by default for you,
both variable will try to clean the same resource at scope end.
So If you created a RAII wrapped type, also need to forbid such shadow copy
(by delete default copy constructor, or define your own sensible logic)

To avoid such unexpected copy while still allow some assignment to work,
C++ introduces move semantic, it allows type to declare
move constructor(unique_ptr do), thus makes return unique_ptr or assign
between them behaves as 'transfer' instead of copy.

These new semantics also makes C++ rules more complex, and introduces more
holes. C++ developers must heard about the rule of three, now it became five,
Semantic inconsistency between them leads to nasty bugs. Move semantic also
makes object behaves like 'destroyed before destructor', even destructor not
run yet, its internal state already being invalidate due to moved out,
that behavior doesn't match RAII very well.


Even understood these tricky rules, C++ still have other holes, For example
we expect constructor/destructor to run only once, but we can call it
just like normal function , as many times as we want, for example

```
{
ComplexResource resource; // resource init
resource::~ComplexResource(); // calling destructor already cleanup resource
} // scope exit, destructor runs again lead a double-free like bug
```
C++ 'placement new' requires manually calling destructor a must, 
but compiler can't tell where did the object pointer come from (placement new?
default new?, or stack based?), thus mismatched destruct can happen and leads
leak/double-free alike bugs.


smart_pointer has similar problem:
```
unique_ptr<ComplexResource> p1(new ComplexResource());
delete p1.get();
```

some C++ developer may argue that these examples are just picked to break
on purpose, which violates best practise, but unfortunately such practise
is not enforced by compiler, If worked with some C++ libraries,
you will find they don't apply same style, some are header based so you're
able to using exposed object as stack variable, some exposed raw pointer
in API to avoid ABI problem, some expose smart_ptr in API to enforce RAII,
you quickly get lost on what to do with them.

C++ language rules are complex,Even experienced developer have to
read hundreds pages of C++ spec
to understand what rules is applied in which condition
(as comparison, go language described as 'boring',
but here boring also means easy to understood).
It makes harder to understand these rules to use 'fashion solution'
than resolving bugs in boring way,  I think this must be one important reason
that kernel developers dislike C++, kernel developers need to control
semantic detail, and prefer simple, easy to understand language.

Now let's see how rust improve this.Rust clarify value semantic,
only POD (in C++ term) type (and compound of POD) can be copied, copy is 
just byte copy, such type can't have destructor.
Non-POD type (any type that has destructor is Non-POD) can't be copied, assign
of on such type always using 'move semantic'.
That means a type can be moveable or copyable, but not both.

Such clarification makes implementation much simpler: Copy just needs a 
'mark declaration' because implementation are always mem copy, and none-POD
type don't need to implement move constructor at all,
because copy is forbidden, move just moved whole variable (including all members),
no need to consider consistency between constructor/copy/move. 
because RAII is always non-POD, shadow copy impossible to happen,
and no wrapper(unique_ptr for example) needed (because variable itself just 
transfer ownership), much less hidden magic happened behind.


It also makes assignment semantic much simpler:
whether if value is copy/move only depends on type itself.
As comparison, C++ calling variant not only depends on
type (maybe multi overloaded, with different cv qualifier),
but also depends on variable assignment context (rvalue? xvalue? or prvalue?
sorry I really don't understand what xvalue/prvalue means). 


More importantly, move semantic in rust are 'truely moved'
instead of C++ 'semantic moved', consider following C++ move:

``` C++
void use_resource_transfered(unique_ptr<ComplexResource> ptr){
  //use ptr
}

void test(){
   unique_ptr<ComplexResource> res = return_resource();
   use_resource_transfered(std::move(res)); // or res.reset()
   use_resource_transfered(std::move(res)); //boom
}
```

although res already moved to use_resource_transfered(or reset),
but the 'moved from' var still accessible even the resource it hold
already invalid(semantically), but in rust the 'moved out' variable itself
became inaccessible:

```rust
fn use_resource_transfered(res: ComplexResource){
  //use resource
}

void test(){
   ComplexResource res = return_resource();
   use_resource_transfered(res); // or drop(res)  res already moved here
   use_resource_transfered(res); // or drop(res) rust tells you res can't be used anymore!
}
```

Before move happened, object ownership belongs to test function syntax scope,
but use_resource_transferred/drop moved the argument, so its ownership now
transferred to use_resource_transferred/drop syntax scope, test function
syntax scope don't own its ownership anymore, any future usage on
res object in test function is invalid. Since object moved into 
use_resource_transferred, res destructor will run at use_resource_transferred
scope end, instead of test function scope end. That surprised me at first,
but after understood the move semantic of rust, it became straightforward,
in rust no more 'moved out' variable left(as C++ move constructor do),
object are always transferred as whole,
so in rust you always think object has 'unique existence lifetime' across whole
program,it constructs/destructs exactly once during lifetime
(no matter how many times you transfer it between different variables), 
instead of 'bind to some variable and then transfer member into new variable'
as C++ move semantic (move multi times in C++ involve multi times
move-construct, destruct process). 

Consider RAII idom, we want resource availability bind to RAII object
lifetime, when ownership transfer happened (underline resource moved
to new RAII object, or being manually destroyed earlier), RAII object itself
should also end its lifetime, but if it still not exit syntax scope yet,
accessing trigger use-after-free bugs. Human quickly failed to track
these accessible-but-invalid states, that leads bad behavior at runtime.
But in rust compiler precisely knows how object
ownership transferred between variables, so it can prevent any invalid access
at **compile time**, completely omit use-after-free bugs.

Such analysis can't be applied to C++, since copy/move semantic is controlled
by developer, compiler can't verify its correctness semantically even for the
ones that compiler generated by default. Even these semantic are correctly
implemented, the 'moved out' left variable still leaves a hole.

Please note that rust do have a 'Clone' concepts to duplicate non-copyable
object, but it have completely different behavior to C++ copy constructor.
Firstly clone in rust are just normal function, have nothing to do with
builtin assignment keyword(you can naming it as 'dup' or anything you like).
Secondly when you clone something as

```rust
let cloned = a.clone();
```

Nothing special happened here, you just need to return a new object from
clone (or your own dup function), and assignment is still a move, it moves
the object function returned to cloned.


Another important feature of rust is 'reference', it also brings from C++, 
but we'll see how rust improved by closing the bug hole.

C++ reference can be viewed as a syntax sugar of pointer,
but appears like a 'value'. To makes it behaves more alike value,
C++ requires reference must be initialized to point to some 'value'.
Due to its pointer nature, compiler can not verify if it really point
to valid value, nor can it assume such reference still valid during usage
(just like raw pointer), for example
```C++
int *p = nullptr;
int &r = *p;  
```

Compiles, (although this piece of code still be 'correct' even at runtime
until read/write r).Valid reference can also be 'invalidated' 
by pointee earlier destroy , for example

```C++
std::string init;
std::string &ref = init;
{
   std::string scoped;
   ref = scoped;
}
ref = "abc" // boom
```
Such dangling reference are same bugs as dangling pointer, but due to its sugar
syntax, debugging may be more difficult (calling a member function on
a 'value', but found this pointer garbage). 

C++ also don't forbid you to destroy an object through its reference (just like
destroy any variable through a pointer)

```C++
ComplexResource value;
ComplexResource &ref = value;
ref::~ComplexResource();
```

How rust improve this ? Rust model reference perfectly with ownership/lifetime.
Reference some variable implicitly the reference itself don't own
the ownership of value, so it 'borrowed' ownership from
original value (instead of own the ownership like move) temperaly, original
variable still owns the ownership, rust assume original variable must have longer
lifetime than reference, so dangling reference is impossible

```rust
let a: &i32;
{
 let b:i32 = 0;
 a=&b;  <--------- rust tells you b live too short, already destroy at scope end
}
let c = a+1;   <------a referenced b and used here so it's an error
```
As discussed before, lifetime of every variable in rust are precisely tracked,
so rust also prevent accident dangling reference to any variable being moved:

```
let a = "abc".to_owned(); 
let ref_a = &a; 
let move_a = a;       // a already moved to move_a
print!("{:?}",ref_a); // error! ref_a still reference a but a itself already moved
~
```
Since reference don't own the ownership, you can also not able to destroy 
underline object through reference, because drop function in rust declare its argument
as 'moved' ownership, passing reference simply no satisfy the ownership
requirements. How brilliant! It has high level semantic of why such
code is incorrect, but rules to forbid bad code still based simple 
ownership/lifetime concepts!

Another important concept of rust reference is 'mutability',
one variable can not have two mutable reference, nor can have mutable and
immutable reference at same time , it sounds like a read/write lock, 
but actually is a compile time rule (without runtime overhead). 
it may sounds like a simple "won't change value without notification" improve,
but actually much more important than that. 


Again Let's take another C++ example to show important this rule is:
```
std::vector<int> vec = {1,2,3};
for(auto v: vec){
  vec.push_back(v); // ---> boom 
}
```
(You may also try this in other languages, java/C# report runtime exception
due to collect modified during iterate, python3/js loop runs for ever until
OOM. go range expr snapshot copy of vec so it worked).
But such logic won't compile in rust:
```
let mut vec = vec![1,2,3];
for e in &vec{  //-------> e here is immutable reference to vec 
  vec.push(*e); //-------> but push requires a mutable reference so conflict!
}
```

In this code iterator already referenced the vec(for loop do this syntax sugar), 
rust knows in for loop scope, vec shouldn't be modified since it being already
referenced by iterator. So vec.push is a compile time error
because it requires vec itself as 'mutable reference' (just like 
C++ non-const member function). 
Such restriction does not only apply to simple scenario, it even works
in chained nested logic, for example
```
let a: &i32;       

let vec = vec![1,2,3];
let it = vec.iter();
let a = it.last().unwrap();   // a is immutable reference to vec last element, but through iter, not directly with vec
vec.push(4);                  // vec want to do mutable change, thus its iter will be invalid, and reference to the iterator also being invalid
let b = a;                    // using invalid reference(such relationship anlysised through two indirect reference) , build error
```


rust also do some context analysis than simply look for variable scope,
if reorder last line assignment before vec.push, thus mutable usage
won't invalid the immutable reference, rust accepts it.

Since these reference mutability tracked through whole control flow,
you won't got unexpected invalid reference forever , no matter 
how hard you tried to:). 

When I learn these concepts first time, the only voice came to my mind is just
        OMG
Even with high level language as java/C#, the best it can do is just
throws an exception to tell you iterator invalid at runtime, but rust
knows it at **compile time**! There's no need to worry about any iterator
invalidation problem in rust! As if it built, iterator always valid,
if build failed, you must break something!


And with rust ability to track ownership/reference mutability, working with
third party libraries makes much easier, since every API exposed object
have well-defined lifetime and mutability, you don't need to care about
whether forgot to release it,or incorrectly deallocate its internal memory,
nor passing already invalid reference to API.

I also suggest interested developers to try rust lambda functions, rust models
lambda with ownership/mutability concepts in mind, thus you enjoy same level
of protection as normal variable,  while it builds, you can assume variable 
reference usage in lambda or lambda itself movement won't introduce any 
dangling reference /leak/ use-after-free bugs (interested developers can try
themselves to see if you can fool rust to accept your codes with such bugs),
such experience makes huge differences to C++ lambda and C callbacks,
which requires you to analysis carefully about the object availability at 
lambda construct/callback time, such as
```C++
{
   std::function<int()> lambda;
   {
      unique_ptr<int> temp(new int(3));
      lambda = [&](){ return *temp;};
   }
   lambda () // boom
}
```
That's the reason why I think it's bad idea to bring
'reference capture in lambda' to C. Without restrict object
ownership/lifetime track (this rule breaks C compatibility),
capture by reference create variables seems like a (owned) value,
with implicitly backed by pointer, will make use-after-free alike
bugs easier to introduce.




As a C++ developer,I'm impressed by rust ownership and reference mutability
concepts, for years development, you must have heard some 'best practice' such as
'who allocate, who deallocate' to keep object lifetime clearly bind to
object/scope, and makes program structure tree liked, thus child object/scope
can safely reference parent node ones', but question is in practice 
with grown of codebase scale, these rules are so easily to break without notice,
maybe due to code introduced by
inexperienced developer, or misunderstood of code structure, or just
by mistake during refactor. That leads nasty bugs silently and seems unavoidable,
But in rust, break these practice (which almost implicitly leads some sort of
problematic lifetime/mutability semantic) simply lead a build failure! 
And if it built successfully, rust proved that you get rid of whole bunch of bugs!
This is a huge improvement for development. 


In recent years, C/C++ tooling are improving, most important tool
I think is address sanitizer (ASAN) developed by google, it catches memory bugs by
inspect all read/write ops at runtime, combined with fuzzer test,
lots of bugs found across many C/C++ codebase, including linux 
[kernel](https://github.com/google/kasan/blob/master/FOUND_BUGS.md)
(as ASAN said: Over the years KASAN has found thousands of issues in the Linux kernel
so maintaining a full list is pointless. This page contains links to some
old bugs found with KASAN back in the days when it was being developed.
Just for historical purposes)
Gentoo developers even created [asantoo](https://wiki.gentoo.org/wiki/AddressSanitizer)
to build whole user environment with ASAN, thus every C/C++ software used
in daily work will be checked for memory bugs during usage. I tried it and quickly
found memory bugs here and there.
(more bugs definitely exist since ASAN can only find bugs of control flow
which actually run).
ASAN greatly improved C/C++ ecosystem, but also shows
how easily C/C++ can introduce memory bugs (these bugs may exist for years
silently). That makes me came to a conclusion: 

Today software industry almost built everything on top of
C/C++ (as kernel and system base libs), these low-level base
definitely needs improvement.

Since I don't know rust at that time,
I think ASAN is the answer(today ARM already supports hardware based ASAN
with less performance hit). And now we have rust and it omits 
many such bugs at build time instead of runtime, we should consider it
as new low-level implementation language, seriously. 
If still using C/C++, fighting with these countless low-level memory bugs
may consume more time than to implement new features.

Microsoft blog also shares similar point
```
While many experienced programmers can write correct systems-level code,
it’s clear that no matter the amount of mitigations put in place,
it is near impossible to write memory-safe code using
traditional systems-level programming languages at scale.
```


It's very hard to talk about rust without compared with C/C++, but that shouldn't
be just a 'language war' topic, although I used C++ as 
comparison (since I'm C++ developer for years), we should realized that 
rust can't be so successful without learning the good and bad
from C/C++ (and many other languages). C/C++ still being used in industry,
we can also bring the good ideal from rust back, to create better C/C++ code. 
For example rust design its mutex to embed data it intend to protect,
you have to use the RAII object returned by lock
method to access the data, thus impossible to mismatch the relationship between
data and corresponding mutex,such design can be easily ported to C++. 

There is a youtube video called
[Rust, a language for next 40 years](https://www.youtube.com/watch?v=A3AdN7U24iU)
shares a point: developers only migrate to new language if
it is 'much better', and he considered rust as the 'much better' language and
will be used as new system programming language in the next 40 years. 
another video [Is It Time to Rewrite OS in Rust?](https://www.youtube.com/watch?v=HgtRAbE1nBM)
shares another point: kernel developers don't care about
huge time spent on implementing correct behavior using err-prone 
memory unsafe language, since once it's correctly implemented,
it works for ever, so less error-prone language add little value to kernel
development. They're both interesting and I recommend readers to view them
to get other's POV as well.

At the time of writing (2021-08), support rust in kernel still 
being discussed in kernel mailing list, Let's wait and see if linux
kernel will accept it, and time will tell if rust will be the new
low-level system language.

