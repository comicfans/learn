as a new language , rust gains lots of attaction, there're also sime C projects begin to migrate to rust (librsvg/curl/tor). 
The most interesting project is definitely try to bring rust to linux kernel development,
what makes it interesting does not only because it's driven by google 
(It fond support developers who're developing this project ,also using rust in their home developed fusicha OS), 
but also that rust may become first alternative programming language in kernel development, other than C.

Previously kernel development always written in C, there're some attempts to bring C++ in, but never success.
although C++ can be seen as C superset (from some ones' perspective), 
Linus personally hate it, he expressed many times "C++ is horrible language". If followed
the long 'bring rust to kernel' mail list 
(https://lore.kernel.org/ksummit/CANiq72kF7AbiJCTHca4A0CxDDJU90j89uh80S3pDqDt7-jthOg@mail.gmail.com/) 
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

then Bart Van Assche shows some C++ code that override new operator globally (I didn't posted them here)
to workaround the problem, and Linus replied:

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

Let's take Linus first argument about 'new operator'

userspace(none-kernel) programmer may be supprised by Linus' point, most language
(such as gc based java/c# or C++) design the library/runtime to work "magically"
when dealing with memory allocation(from kernel developer's POV) . for example in C++

```
std::string temp;
temp.push_back("I_am_very_long_string");
```
there is no explicit allocating call , but under the hood such code must allocate 
dynamic heap memory. Running this code in userspace is no big deal, it just said 
"I don't care about how memory come from,
just give me 1MB memory, or I will throw a Out-Of-Memory exception or just terminated", and most
userspace program don't care about OOM exception, they just let it crash whole program. 
 this makes sense for most scenario, since most program is designed to only work under
"normal" environment(say, with enough memory to finish the work), if such condition not 
satisified anymore, there's very little it can do to recover. suppose another leaking
program consumed all the memory, leads your program memory allocation failed,kill the leaking program
is the only way to recover. but to do this, you still need memory 
allocation (to list and find leaking one, thus OOM again and can't complete)


but this is not the case for Kernel.firstly there's no simple 'just gives me 1MB  memory' like malloc/new, 
kernel need to allocate memory with many flags depends on current running context, and secondly if such allocation 
failed, you can not simply throw OOM exception and crash, not only because exception is
too heavyweighted to be included in kernel, but also because a sensible kernel should never crash,
even under extream OOM case. As Bart Van Assche suggested, there're some workaround 
to make C++ new work in kernel (for example allocate memory 
manually first, and then using placement new, or using nothrow new), but the second argument
is the real reason why Linus not accept C++ as kernel development language:

```
but it doesn't change the fact that it doesn't actually *fix* any of the issues that make C problematic
...
 At least the argument is that Rust _fixes_ some of the C safety issues. C++ would not.
```

and  Miguel Ojeda (who posted 'support rust in kernel' patches) followed

```
The thing is, claims such as "C++ is as safe as Rust, you just need to
use modern C++ properly!" are still way too common online, and many
developers are unfamiliar with Rust, thus I feel we need to be crystal
clear (in a thread about Rust support) that it is a strict improvement
over C++, and not a small one at that.
```

Argues over different language are almost FUD topic, consider that C++ already 
being larged used by industry. but I suggest readers (escpecially C++ developers)to keep peace,
since rust grown popular largly because of firefox migration, which its codebase
originally developed in C++ for many years, developers behind firefox are
definitely knows C++ well, and kernel developers are deinitely knows C well,
maybe we can learn something from their argues.

so what's the issue that makes C problematic ?
short answser is memory safety, C is so bare-metal that it almost didn't forbit any incorrect semantic
code, for kernel development, it makes low-level access easier, but also too relaxed to code correctly,
 even for the most intellegent people who use it for years kernel developing.  
 
 I grep the kernel git log from 2.6.12-rc2(2005-Apr 1da177e4c3f) to about 5.13-rc6 (2021-Jun 6b00bc639) by
```
git log --grep=leak --grep=error --all-match --oneline | wc -l
```
got 4626 result, you may also grep for "double free" (2234) or "use after free" (9987) to know how many memory 
safely bugs introduced and fixed in linux kernel. please note, some of these bugs are logic related ( 
even written in other languages it can still happen), but lack of memmory safety in C definitly be the cause
of most memory errors.

(side notes from tor 
in https://blog.torproject.org/announcing-arti
```
Since 2016, we've been tracking all the security bugs that we've found in Tor, and it turns out that 
at least half of them were specifically due to mistakes that should be impossible in safe Rust code
```
and libcurl
```
19 of the last 22 vulnerabilities (since 2018) have C-induced memory unsafety as a cause.
```
)

These catalog of bugs share some common property: they're so "trivial", even not being
kernel developer, we can easily understand why these piece of code is incorrect (with help of commit), 
and they're also so error-prone,even experience kernel developers continuous introduce such bugs. 
here I pick a leak fix as example

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

seems easy to fix, right ? code have two allocations, but forget to deallocate first one if the 
second allocation failed. but the question is such leak bug is also easy to introduce, suppose similar
function need to allocate many times, this problem became harder. what makes it worse is maintainable,
since the cleanup function should be exactly same order as allocation (otherwise such cleanup table can't
be shared from different failure point), developer needs to be very careful to keep them
in sync when adjust order of allocation. break this rule may lead to leak/double free bugs.

So does C++ have improvement in this area (leak) ?  I think the answser is yes (Linus maybe not agree with it :)
RAII (as mentioned by Bart Van Assche), is C++ idom to address such problem, the trick is that constructor/destructor
will be run exactly only once during object lifetime, so put resource acquire/release in constructor/destructor
makes such object auto manage the resource (usually as a stack variable which lifetime binded to syntax scope).
similar mechanisum is also borrowed by many other languages
for example java 1.7 try-with-resource, C# 8 using, go defer function, and also rust drop,
even C commitee is looking into adding defer function to implement RAII like idom.
(http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2542.pdf)
From my personal view, before rust , C++ approach is better than Java/C# or go defer(also including proposed C defer),
I list my reasons here:

Java/C# forces developer to make resource type inherit particular interface to corperate
with resource manage syntax, but since cleanup function are just public method exposed by type,
developer are free to call these cleanup function at their will, that force cleanup function to 
always bookmark "resource already cleanup flag" in logic to avoid double-free alike bugs.


defer function have semantic confusions : 
the defer function defined point(in-between code) must be different to 
execute point (function end, or scope end), but what values do the argument (used by defer function) hold? 
do they 'capture' values at define point , or do they use most-written new value ? 
go prefer previous (https://tour.golang.org/flowcontrol/12)
```
The deferred call's arguments are evaluated immediately, but the function call is not executed until the surrounding function returns.
```
so 
```
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
lead a mismatch cleanup. this policy implicits that
the variable which hold resource can't be changed to point to new resource after
defer function defined(otherwise you're cleaning incorrect resource),
or you have to define new defer function everytime when new resource is created and assigned.


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
(I'm strongly opposed to the idea of bring 'reference capture in lambda' to C, in later part
we'll see why it will introduce more problems)

C++ RAII idom makes acquired resource
kept as member, so it has no confusion during cleanup, and more important, such idom makes
resource avaialbility exactly same as wrapper type lifetime,usch abstraction is straightforward. 
and this makes cleanup timepoint decoupled from fixed syntax scope. 
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
use unique_ptr to make ComplexResource lifetime outlive scope where it constructed, 
and it still hold auto management property (unique_ptr is also a RAII object which manage its holding pointer)

but C++ RAII also have weakness, let's also talk about them. first is the C++ copy semantic,

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
This also brings another C++ confusing question: unique_ptr seems being copied multi times,
doesn't this also trigger similar problem  ? the answser is no,  
unique_ptr implementation forbid copy (at compile time) so you can't copy unique_ptr 
(even by accident because that leads compile error) thus it prevent unexpected copy lead double free bug,
for example
```
unique_ptr<ComplexResource> p1(new ComplexResource());
unique_ptr<ComplexResource> p2 = p1; //won't compile, error
```

but this raises more confusing questions, if can't be copied,
how can function return it and how can returned value being
assigned to variable ? the answser is C++ have many other 'tricky' rules,
which makes these 'elision-copy' and 'move assignment'  instead of copy 
(although such rules doesn't match programmers 
straightforward view of code very much)

Even experienced C++ developer have to read the hundreds pages of C++ spec
to understand what rules is used in which condition, it's even harder to
understand these rules than resolving bugs in simpler way,  I think it must be one important reason
that kernel developers dislike C++ , since kenerl development is low-leveled,
developers want to control every detail straightforward as simple as C semantic.


even understood these tricky rules, C++ still have other problems, RAII only
works if we strictly follow some concepts, otherwise failed.
for example we expect destructor to run only once, but we can call it
just like normal function calls, as many times as we want, for example

```
{
ComplexResource resource; // resource init
resource::~ComplexResource(); // calling destructor already cleanup resource
} // scope exit, destructor runs again lead a double-free like bug
```
Developers may be suprsied by such behavior, why allow it ? since C++ have 'placement new'
which construct object over manually allocated memory (instead of built-in auto allocate mechanisum),
such created object won't be managed by C++ 'auto destrory at scope exit' rule,
(because compiler don't know how to deallocate your manually allocated memory at compile time)
thus manually calling destructor worked in this scenario, but it leaves a bad hole,
if you failed to track object construct method (normal constructed or placement new constructed),
you may experience double-free alike bugs, or leak alike bugs.

similar problem happened on smart_pointer
```
unique_ptr<ComplexResource> p1(new ComplexResource());
delete p1.get();
```

which leads a double-free alike bug.
some C++ developer may argue that I shouldn't use manual delete on pointer
which managed by smart pointer, but unforunitely, such rule is not enforced by compiler
and real world codebase are much more complex, such error pair may be in completely
different files, not so easy to spot. especially when you have many legcy code which
only accept raw pointer.

now let's see how rust improve this. at first, rust approach borrowed from C++ ,
constructor acquire resource, destructor (drop) cleanup it,
but rust non-POD (in C++ term) types use 'move semantic', which means
assignment of RAII type in rust are always transfered (without C++ whole bunch
of dark rules), so no wrapper (unique_ptr in C++ example, or you have to implement move constructor yourself) 
is needless. Interesting part came for
manually object destroy, rust default destruct object at scope end, but you can make it earlier by

```
ComplexResource resource = return_resource();
drop(resource);
```

firstly its alike calling destructor manullay or unique_ptr.reset,
but they're different, calling destructor manully on stack variable in C++
leads double free (alike) bugs, but rust
knows you don't need to run destructor again because you already drop it.
and this is also different to unique_ptr.reset, since unique_ptr only
knows whether to call destructor at runtime (depends on if internal pointer null),
but rust knows such behaivor at compile time, without any runtime
bookmark ! and future more, in C++, you may access nullptr after reset unique_ptr
by acceident 
```
unique_ptr<ComplexResource> res= return_resource();
res.reset();
use_res_again(res);  // still compile, even incorrect
```

but rust compiler find your mistake at compile time!
```
ComplexResource res = return_resource();
drop(res);
use_res_again(res); ------------> rust won't compile, it tells you res already moved in drop call so you can't use it anymore! 
```

How can this be done? because rust drop function argument declared as 'takes ownership',
since drop already taken its ownership, any later code won't be able to access it.
This feather really makes rust shine and unique, it prevent whole bunch of leak/use-after-free
alike bugs at compile time. Let's consider C(and many other languages) codes, 
variable accessiblility is binded to syntax scope ( innermost { } which beyond variable),
but the lifetime of resource it holds may be different, use-after-free alike 
bugs happend when resource is cleaned up but variable being accessed later(because it still accessiable in scope) ,
as long as program grow complex, human quickly failed to track these explored states combination,
but rust compiler track these invalid states for you, once again, at compile time.

as a comparison, although C++ move constructor can implement some sort of similar 'ownership transfer' behaivor,
but still allow access to 'already moved out' variable, thus use-after-free alike bugs can still happen. 

another important feather of rust is 'reference', it sounds like C++ reference, 
but we'll see how rust improve this feathuer by closing bad hole.

C++ reference can be viewed as a syntax sugar of pointer, but appears like a 'value'.
to makes it behaivor more alike a value, C++ requires reference must be initialized to point to some value,
but due to its pointer nature, compiler can not verify if it point to valid value,
nor can it assume such refernce always valid (just like raw pointer), for example
```
int *p = nullptr;
int &r = *p;  
```
compiles, and suprised to some readers, it's also correct (even at runtime) , because
reference variable are just pointer, initialize reference won't deference the pointee, thus no problem.
but read/write this reference later is same error as deference null pointer.
valid reference can also be 'invalidated' by pointee earlier destroy , for example

```
std::string init;
std::string &ref = init;
{
   std::string scoped;
   ref = scoped;
}
ref = "abc" // boom
```


how rust improve this ? it forbids reference variable to have longer lifetime than pointee variable
```
let a: &i32;
{
 let b:i32 = 0;
 a=&b;  <--------- rust tells you b live too short, already destroy at scope end
}
let c = a+1;   <------but a referenced b and used here so it's an error
```

another importand rule of rust reference is 'mutability',one variable can not have two mutable reference
, nor can have mutable and immuable reference at same time , it sounds like a read/write lock, 
but actually is a compile time rule (without runtime overhead). Let's take another C++ example to show 
why such rule is important 
```
std::vector<int> vec = {1,2,3};
auto it = vec.begin();
vec.push_back(4);
*it // ---> boom
```
but similar logic won't compile in rust:
```
let mut vec = vec![1,2,3];
let it = vec.iter();  ----> iter has immutable referenced to vec
vec.push(4);          ----> push do a mutable referenced to vec
let last = it.last(); ----> used immutable referenced again, build error
```

rust member function is alike C++ one's ,which can be marked as const
thus makes this pointer const or not, but different to C++, such 'mutable'
also prograte effects if returned type have reference to 'this' state,
since iter reference vec state, when such variable exist, rust keep the 
vec immutable, thus out-of-date reference bug won't happen.

such restriction does not only apply to simple scenairo, it provides such protection
even in nested logic, for example
```
let a: &i32;       

let vec = vec![1,2,3];
let it = vec.iter();
let a = it.last().unwrap();   // a is immutable reference to vec last element
vec.push(4);             // but vec want to do mutable change
let b = a;                    // a immutable reference to vec (through iter), again build error
```

rust also do some context anlysis than simply look for variable scope, if reorder last line assigment
before vec.push, thus mutable usage won't invalid the immutable reference, rust accepts it.

If such reference behaivor is based on dynamic control flow,
rust also provides helper library (Cell/RefCell) to implement it at runtime while still enforce same rules as compile time,
but raise a runtime crash if such rule breaked.

As a C++ developer , ownership and reference mutable concepts attacts me most, 
for years C++ development, you must heared some 'best practice' such as
'who allocate, who deallocate' to keeps object lifetime clearly binded to
some object/scope, or makes program structure tree liked, thus child object/scope
can safely reference parent node's, but in practices,
break these rules may only lead to nasty memory bugs without notice, 
when codebase grew, it's almost impossible for human to identify them.

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