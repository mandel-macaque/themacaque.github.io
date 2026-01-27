# Objective-C vs C#: A Conceptual Primer for Bindings

In the previous post I explained how we can design an SDK, that with the help
of code generators and analyzer, could behave like a DSL (Domain Specific Language).
In this article I am going to focus on given the needed background to understand what
it means to create a binding generator from Objc to C#

When creating bindings between Objective-C and C#, you’re not just translating syntax—you’re bridging two runtimes with
very different philosophies. Those differences show up most clearly in memory management, method dispatch, and abstraction
mechanisms (protocols vs interfaces). Understanding these mismatches is critical to producing correct, safe, and idiomatic
bindings.

## Objects and Memory Management: ARC vs Garbage Collection

### Objective-C: Reference Counting with ARC

*Objective-C* uses reference counting as its fundamental memory management model. Modern Objective-C relies on ARC 
(Automatic Reference Counting), which inserts retain, release, and autorelease calls at compile time.

*Key points*

- Every object has a retain count.
- Ownership matters.
- Object lifetimes are deterministic.

```objectivec
NSObject *obj = [[NSObject alloc] init]; // retain count +1
self.property = obj;                     // retain
[obj release];                            // release
```

ARC removes the need to write these calls manually, but the *ownership rules still exist:*

- *strong / retain:* keeps the object alive
- *weak:* does not retain, automatically zeroed when the object deallocates
- *assign:* unsafe for object references (no zeroing)

This ownership rules leak to the API design:

```objectivec
@property (nonatomic, strong) NSObject *owner;
@property (nonatomic, weak) NSObject *delegate;
```

This matters when designing an API in objc because:

- Cycles (strong ↔ strong) leak memory.
- Delegates must be weak by convention.

### C#: Garbage Collection

C# uses a tracing garbage collector, we might have long conversations on why this is a
better or worse approach; or you can be like python, and do both ;)

- Objects are allocated on the managed heap.
- Memory is reclaimed non-deterministically.
- Developers generally don’t think about object lifetimes.

```csharp
var obj = new object();
// No retain/release, GC handles cleanup
```

From the above, the important details we need to understand are:

- C# references are always strong by default.
- WeakReference&lt;T&gt; exists but is explicit and rare.
- Cycles are not a problem (most of the time).

The following table shows the mismatch between the two and what we 
have to be careful about:

| Concept     | Objective-C       | C#                         |
| ----------- | ----------------- | -------------------------- |
| Lifetime    | Deterministic     | Non-deterministic          |
| Cycles      | Leak              | Collected                  |
| Weak refs   | Built-in & common | Explicit & uncommon        |
| Destruction | `dealloc`         | Finalizers / `IDisposable` |

When binding ObjC to C#, you must:

- Preserve ObjC ownership semantics.
- Prevent premature deallocation of native objects.
- Avoid leaking native objects retained by managed references.

## Method Resolution: Message Passing vs Method Calls

### Objective-C: Message Passing

In Objective-C, methods are not called—they are messages sent at *runtime*.

```objectivec
[id object doSomething];
```

 If we [simplify](https://developer.apple.com/documentation/objectivec) the above a lot that can be translated to:

 ```objectivec
 objc_msgSend(object, @selector(doSomething));
 ```

Consequences:

- Method resolution happens at runtime.
- Objects can respond to selectors dynamically.
- Messages to nil are legal and do nothing.
- Method implementations can be swapped at runtime.

We can consider Objc to be a dynamic language more like Python or Smalltalk (or C like some smart people can argue)

### C#: Static Method Dispatch (Mostly)

C# uses compile-time method resolution, with runtime dispatch only for:

- *virtual* methods
- Interface calls
- Reflection / *dynamic*

```csharp
obj.DoSomething(); // must exist at compile time
```

We can summarize this as _"If it does not exist, it does not compile"_. This a *HUGE* simplification,
virtual methods, interfaces and dynamic work in a diff way, they are useful but come with a cost.

### A Classic ObjC Gotcha for Bindings

Apple often returns objects that behave like a class but aren’t actually that class, and you can see a lot
of this cases in the Microsoft.iOS bindings, I have had several issues with the NSData object and the NSUrlSession

```objectivec
NSString *s = [someApi returnsAString];
NSLog(@"%@", [s class]); // might be __NSCFString
```

This is legal because:

- ObjC relies on behavior, not concrete types.
- Class clusters are common (NSString, NSArray, NSNumber).

The fact that Apple does this is of a lot of importance in our API design. Bindings must:

- Avoid assuming concrete native types.
- Prefer base classes or protocols.
- Be defensive around casting.

## Protocols vs Interfaces

### Objective-C Protocols

Protocols define a contract of behavior, not inheritance. This is very heavily used and pushed in Swift
with the concept of [protocol oriented programming](https://www.hackingwithswift.com/sixty/9/5/protocol-oriented-programming).

```objectivec
@protocol MyProtocol <NSObject>
@required
- (void)requiredMethod;

@optional
- (void)optionalMethod;
@end
```

Features:

- Methods can be optional
- Checked at runtime via respondsToSelector:
- Used heavily for delegation

```objectivec
if ([delegate respondsToSelector:@selector(optionalMethod)]) {
    [delegate optionalMethod];
}
```

### C# Interfaces

Traditionally, C# interfaces:

- Required all members to be implemented
- Were purely abstract (changes were added thanks to the Microsoft.iOS folks)

```csharp
public interface IMyInterface
{
    void RequiredMethod();
}
```

This made binding ObjC protocols awkward, especially when optional methods were involved. The old
Xamarin API had to work around this situation when generating code and that is why the API for
protocols is kind of complicated (we will look at the old API in a future post).

C# 8.0 introduced default implementations, partially closing the gap:

```csharp
public interface IMyInterface
{
    void RequiredMethod();

    void OptionalMethod()
    {
        // default behavior
    }
}
```

This was added to fix a few issues:

- Allow SDKs to introduce new methods and not change API
- Help with Xamarin.iOS

The following table simplifies the differences:

| Feature          | ObjC Protocol | C# Interface        |
| ---------------- | ------------- | ------------------- |
| Optional methods | Yes           | No (until defaults) |
| Runtime checks   | Yes           | No                  |
| Duck typing      | Common        | Rare                |
| Delegation       | Idiomatic     | Less idiomatic      |

## Closing: Why This Matters for Bindings

Objective-C and C# are both “object-oriented,” but they express that idea very differently:

- ObjC favors runtime flexibility
- C# favors compile-time safety

When creating bindings, you are translating:

- Deterministic lifetimes → GC
- Message passing → method calls
- Behavioral typing → nominal typing

Understanding these differences upfront helps avoid:

- Memory leaks and crashes
= Invalid type assumptions
= Fragile or unidiomatic APIs on the C# side

In short: good bindings don’t just compile—they respect the semantics of both worlds. If you
think about it, Roslyn analyzer make perfect sense to allow the introduction of new semantics
to the C# language. Enforcing these new semantics allows to create better and more idiomatic bindings
with the safety net of a compile time check
