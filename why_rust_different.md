as a new language,  rust gains lots of attactions, there're also some C projects begin to migrate to rust (librsvg/curl/tor). The most interesting project is trying to bring rust to Linux kernel development, what makes it interesting does not only because it's driven by Google (It fund development in linux kernel, and using rust in their home-developed fuchsia OS), but also that rust may become first alternative programming language in kernel development, other than C.

Previously kernel development always written in C, there're some attempts to bring C++ in, but never success.
although C++ can be seen as C superset (from some ones' perspective), 
Linus personally hate it, he expressed many times "C++ is horrible language". If followed
the long [bring rust to kernel](https://lore.kernel.org/ksummit/CANiq72kF7AbiJCTHca4A0CxDDJU90j89uh80S3pDqDt7-jthOg@mail.gmail.com/)  mail list
, we can find Linus still holds his opinion.

This mail list starts at 2021-06-25 , Linus (please note it is Linus Torlards, not Linus Walleij) 
keeps silent until 2021-07-07 , after Bart Van Assche brings up topic 'why C++ not in kernel'
 
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


userspace(none-kernel) developer may be surprised by Linus' point, most language
(such as gc based java/c# or C++) design the library/runtime to work "magically"
when dealing with memory allocation(from kernel developer's POV) . for example in C++

```
std::string temp;
temp.push_back("I_am_very_long_string");
```
there is no explicit allocation call , but implicitly allocate dynamic heap 
memory under the hood. Writing such code in userspace is no big deal, it just said 
"I don't care about how memory came from,just give me 1MB memory, 
or I will throw Out-Of-Memory exception". 
most userspace programs don't care about it and just crash.  
this makes sense for the most scenario since they're designed to only work 
under "normal" conditions(say, with enough memory to finish the work), 
if such condition not satisfied anymore, there's very little it can do to recover.
suppose another leaking program consumed all the memory, leads your program OOM,
kill the leaking one is the only way to recover. But to do this, you still need memory 
allocation (to list and find leaking one through OS API,
thus OOM again and can't complete)

But this is not the case for Kernel. Firstly there's no simple 
'just gives me 1MB memory' like malloc/new, 
kernel need to allocate different memory with different flags depends on 
current running context, and secondly if such allocation 
failed, you can not simply throw OOM exception and crash, not only because 
exception is treated as 'valueless' in kernel, but also because 
a sensible kernel should never crash even under rare OOM case.

then Bart Van Assche shows some C++ code that override new operator globally
(I didn't post them here) to resolve the problem, and Linus replied:


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

can improve kernel development, but Linus simply rejected this ideal.


Miguel Ojeda (who posted 'support rust in kernel' patches) posted a sample 
that how easily C++ can introduce memory bugs without notice, 
even with moden standard.

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
that C++ already being largely used by industry. C++ is superset of C
and it accepts buggy logic C code thus lead same bug, that is understandable,
so some C++ best practise and new standard(such as smart pointer instead of raw pointer)
still being needed to make code less error-prone
(otherwise it makes no differences to C), 
OTOH memory safe language are almost same meanings to GC based,
How can a language ,say rust, being same low-level as C/C++ while still 
can avoid such memory bugs ? I think it's primary question of many readers.

Before answer that question, we may firstly find what's the issue that 
```   makes C problematic? 
``` 
short answser is just memory safety, C is so bare-metal that 
it almost didn't forbid any incorrect 'semantic' code, for kernel development,
it makes low-level access easier, but also too relaxed to code correctly,
even for the most intelligent people who use it for years.  
 
 I grep the kernel git log from 2.6.12-rc2(2005-Apr 1da177e4c3f) to about 5.13-rc6 (2021-Jun 6b00bc639) by
```
git log --grep=leak --grep=error --all-match --oneline | wc -l
```
got 4626 result, you may also grep for "double free" (2234) or
"use after free" (9987) to know how many memory safely bugs introduced 
and fixed in linux kernel. Please note, some of these bugs are logic related ( 
written in other languages may not help), but lack of memory safety in C is 
the cause of most of them.

as a side note, Tor blogged
in https://blog.torproject.org/announcing-arti
```
Since 2016, we've been tracking all the security bugs that we've found in Tor, and it turns out that 
at least half of them were specifically due to mistakes that should be impossible in safe Rust code
```
and libcurl
```
19 of the last 22 vulnerabilities (since 2018) have C-induced memory unsafety as a cause.
```

These catalog of bugs share some common property: they're so "trivial",
even not being kernel developer, we can easily understand why these piece of 
code is incorrect (after reading the fix commit), and fix is very short
(just add missing release) and they're also so easy to happen,
even experienced kernel developers introduce them here and there,
and hard to be discovered/reproduced, without depth knowledge of logic and
control flow, you don't even notice it should be a bug by simply look around(
before reading the fix commit)


here I pick a simple leak fix as example

```
619fee9eb13 

 net: fec: fix the potential memory leak in fec_enet_init()

    If the memory allocated for cbd_base is failed, it should
    free the memory allocated for the queues, otherwise it causes
    memory leak.

    And if the memory allocated for the queues is failed, it can
    return error directly.

```

Let's pick the importand part out to see what happened:

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

Seems easy to understand, right ? two allocations happened,
but forget to deallocate first one if the second failed.
but suppose similar logic need to allocate many times, it became 
more and more problematic. Maintainability also deduced,
since the cleanup function should be exactly same order as allocation
(otherwise it can't be shared by jumped from different failure point),
developer needs to be very careful to keep them
in sync when adjust order of code. Break this rule lead to leak/double free bugs.

So does C++ have improvement in this area?  Personally I think the answer is 
yes (Linus maybe not agree with it !)
RAII (as mentioned by Bart Van Assche), is C++ idom to address leak alike bugs,
put resource acquire/release in constructor/destructor since they'll be
run only once during object lifetime, such object auto manage the resource 
(usually as a stack variable which lifetime binded to syntax scope).
similar mechanisum is also borrowed by many other languages,
for example java 1.7 try-with-resource, C# 8 using, go defer function,
and also rust. Even C commitee is looking into adding defer function to 
implement RAII like idom.
(http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2542.pdf)
From my personal view, before rust , C++ approach is better than Java/C# or 
go defer(also including proposed C defer), I list my reasons here:

Java/C# forces developer to make resource type inherit particular interface 
to cooperate with the syntax, but since cleanup function are just 
public method exposed by type, developer are free to call these functions 
at their will, that force cleanup function to always bookmark 
"resource already cleanup flag" in logic to avoid double-free alike bugs.


defer function have semantic confusions : 
the defer function defined point(in-between code) must be different to 
execute point (function end, or scope end), but what values do the argument 
(used by defer function) hold? do they capture values at define point ,
or use most-written value ? 
go prefer previous (https://tour.golang.org/flowcontrol/12)
```
The deferred call's arguments are evaluated immediately, but the function call is not executed until the surrounding function returns.
```
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


C porposed defer function also have such confusions:

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
(I think it is bad idea to bring 'reference capture in lambda' to C, 
dissucssed later)

C++ RAII makes acquired resource as member,
so it has no confusion during cleanup, and more important, such abstraction
is straightforward, resource availability is exactly same as wrapper type 
lifetime, which makes cleanup timepoint decoupled from fixed syntax scope. 
for example

```
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

but C++ RAII also have weakness, let's also talk about them. First is the copy semantic,

if your ComplexResource is written as 
```
struct ComplexResource{
   void *resource;
};
```
then
```
unique_ptr<ComplexResource> resouce = return_resource();
ComplexResource temp = *source
```
lead a double free problem, because copy semantic makes
temp var also holding resouce and it will also try to cleanup at scope end.
So If you created a RAII wrapped type, also need to forbid such shadow copy.

and second may be the move semantic (and complex rule behind it). To avoid 
unexpected copy while still
allow some assignment to work, C++ introduces move semantic, 
it allows type to declare move constructor(unique_ptr do), 
thus makes return unique_ptr or assign between them behaves as 'transfer'
instead of copy.


These new semantics also makes C++ rules more complex, and introduces more
holes. you must heard about the rule of three, now it became five.
semantic inconsist across these five leads to nasty bugs,
since assignment appeared as a simple 'do nothing important' statement,
easily being ignored. 

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
default new?, or stack based?), thus inconsist destruct can happen and leads
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
free to using values as your wish, some exposed raw pointer in API to avoid
ABI problem, some expose smart_ptr in API to enforce RAII idom, you quickly
get lost on what to do with them.

C++ language rules are also complex,
Even experienced developer have to read hundreds pages of C++ spec
to understand what rules is used in which condition.
(as comparison, go language described as 'boring',
but boring go are easy to understood)
It may be already harder to understand these rules than resolving bugs in 
simpler way,  I think this must be one important reason that kernel developers 
dislike C++, kernel developers need to control semantic detail, and prefer
language which semantic straightforward and simple as C.

Now let's see how rust improve this.Firstly rust clarify value semantic,
only POD (in C++ term) type (and compound of POD) can be copied, copy is 
just byte copy, but such type can't have destructor.
Non-POD type (any type that has destructor is Non-POD) can't be copied, assign
of on such type always using 'move semantic'.

Such clarification makes implementation much simpler: Copy just needs a 
'mark declaration' because implementation are always mem copy, and none-POD
type don't need to implement move constructor at all(
because whole variable moved, copy forbidden so no need to custom move
behavior).  RAII type no longer to worry
about shadow copy (forbidden), and no wrapper needed(unique_ptr for example),
much less hidden logic happened behind.


It also makes calling semantic simpler:
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

although res already moved to use_resource_transfered(or just reset),
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
   use_resource_transfered(res); // so here can't use res anymore!
}
```

Consider RAII idom, we want resource availability is bind to RAII object
lifetime, but when ownership transfer happened (underline resource moved
to new RAII object, or being destroyed earlier), RAII object itself
should also end its lifetime, but if it's still accessible
(due to variable itself not exit syntax scope), use-after-free bug can triggered
through this hole. Again, this is not what we intend to, but As program grown
complex, human quickly failed to track these explored states combination,
but rust compiler track these invalid states for you at compile time.


Another important feature of rust is 'reference', it also brings from C++, 
but we'll see how rust improved by closing the bug hole.

C++ reference can be viewed as a syntax sugar of pointer,
but appears like a 'value'. to makes it behaves more alike value,
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

```
std::string init;
std::string &ref = init;
{
   std::string scoped;
   ref = scoped;
}
ref = "abc" // boom
```

how rust improve this ? it forbids reference variable to have longer lifetime
than pointee variable
```
let a: &i32;
{
 let b:i32 = 0;
 a=&b;  <--------- rust tells you b live too short, already destroy at scope end
}
let c = a+1;   <------but a referenced b and used here so it's an error
```

because move semantic in rust is well defined, lifetime of every variable can
be precisely tracked to check this invariant.

Another important rule of rust reference is 'mutability',
one variable can not have two mutable reference, nor can have mutable and
immutable reference at same time , it sounds like a read/write lock, 
but actually is a compile time rule (without runtime overhead). 
it may sound like a simple "won't change value without notification" improve,
but actually much more important than it. 


Again Let's take another C++ example to show important this rule is:
```
std::vector<int> vec = {1,2,3};
for(auto v: vec){
  vec.push_back(v); // ---> boom 
}
```
You may also try this in other languages, java/C# report runtime exception
due to collect modified during iterate, python3/js loop runs for ever until
OOM. go range expr snapshot copy of vec so it worked.
but such logic won't compile in rust:
```
let mut vec = vec![1,2,3];
for e in &vec{  //-------> e here is immutable reference to vec 
  vec.push(*e); //-------> but push requires a mutable reference so conflict!
}
```

in this code element to iterate referenced vec (to be
precise,it's the iterator object which referenced the vec object), 
rust knows in for loop scope, shouldn't mutate vec since it will
invalid iterator ,so vec.push is error because it requires vec itself as
'mutable reference' (rust member function can mark this pointer mutable or not,
just like C++ const/non-const member functions).

such restriction does not only apply to simple scenairo, it even works
in chained nested logic, for example
```
let a: &i32;       

let vec = vec![1,2,3];
let it = vec.iter();
let a = it.last().unwrap();   // a is immutable reference to vec last element, but through iter, not directly with vec
vec.push(4);                  // vec want to do mutable change, thus its iter will be invalid, and reference to the iterator also being invalid
let b = a;                    // using invalid reference(such relationship anlysised through two indirect reference) , build error
```

rust also do some context anlysis than simply look for variable scope, if reorder last line assigment
before vec.push, thus mutable usage won't invalid the immutable reference, rust accepts it.
since these reference mutablility tracked through whole control flow,
you won't got unexpected invalid reference forever , no matter 
how hard you tried to:). C++ developers will find modify through alias
to break rust's protection is a no go, because every reference type's usage
scope are also tracked by rust, it will catch all these invalid access.


As a C++ developer , ownership and reference mutable concepts attacts me most, 
for years C++ development, you must heard some 'best practice' such as
'who allocate, who deallocate' to keeps object lifetime clearly bind to
some object/scope, or makes program structure tree liked, thus child object/scope
can safely reference parent node's, but in practices,
break these rules may only lead to nasty memory bugs without notice, 
But in rust, developers will find break these practise simply makes
code not build, and if it built, program just get rid of whole bunch of
low-level bugs! This is a huge improvement for development.

and with rust ability to track ownership/reference mutable, working with
third party libraries is much easier, any API exposed object lifetime
///// here


To address this, google developed address sanitizer to catch these bugs by inspect
all memory read/write action at runtime, combined with fuzzer test,
lots of memory bugs found in many C/C++ codebase, including linux kernel.
it does not only greatly improved C/C++ ecosystem, but also shows how easily C/C++
can introduce memory bugs. Gentoo developers even create 'asantoo' to build whole linux environment
with ASAN eanbled, I tried to use it in my daily work, and quickly found
memory bugs happened here and there. and more bugs may (almostly) exist since
ASAN can only find bugs for actually running control flow.
That makes me to come to a conclusion: If still using C/C++ as low level
implementation language, we may, spend most of time to fight with memory bugs,
thus no time left to build a better ecosystem. so we should consider rust,
seriously, since as Linus said, `Rust _fixes_ some of the C safety issues`.
I personally interpret his second half as 'C++ improved ,but still leaves the bad door open'.

I also suggest interested developers to try rust lambda function , rust model
lambda with ownership/reference mutable in mind, thus you enjoy same protection as variable 
ownership/reference mutable,  while it builds, you can assume any variable usage or lambda
moving around won't introduce dangling reference /leak/ use-after-free bugs. and after
experiencing rust lambda, you may understand why I say its bad idea to bring 'reference capture in lambda' to C,
without restrict object ownership/lifetime track feather, capture by reference create varialbles
seems like a (owned) value, with implictly backed by pointer, which makes use-after-free alike bugs easier to introduce.
consider similar C++ (which already supports capture by reference in lambda)
```
{
   std::function<int()> lambda;
   {
      unique_ptr<int> temp(new int(3));
      lambda = [&](){ return *temp;};
   }
   lambda () // boom
}
```

but similar bugs can't be built in rust, interested developers can try themselves, 
to see if you can fool rust to accept your codes which has similar problem.


Rust also have other great feathers, someone may mention Send/Sync, since they depends on unsafe 
implementation (unsafe means rust can't verfiy the correctness of some low level operations, 
developer takes responsbility to implement it correctly. such as raw memory operation, calling into foreign function ), 
I don't think they're as important as ownership/reference mutable.
of course rust is not perfect, for example self-referenced structure code will be 
biolpate (there's another interesting topic which called rust makes bad-structured program bad-smelled).
async ecosystem still needs improvement, and so on. but for the most basic part,
rust do provides soild building blocks to construct problem with less bugs. 
developers may find rust code 'just works if build', and this is a big steps for development.

It's very hard to talk about rust without compared with C/C++, but that shouldn't
be just a 'language war' topic, programming languages are learning from each other,
again consider that mozilla developed rust because they're tired to fixing memory bugs of C++.
C++ still being used in industry, we can also bring back the ideal from rust back, 
to create better C++ code.  for example rust design its Mutex to embed the data to protect
as private member, you must own the RAII object of lock method returned to access the data,
it provides better semantic than using individual mutex and data. makes access-without-mutex
bugs impossible at build time. and such mutex design can be easily ported to C++.

at the time of writing (2021-07), support rust in kernel still being disuccessed in mailing thread,
we can see if it can be accepted, and time will tell if rust really improved our development.
