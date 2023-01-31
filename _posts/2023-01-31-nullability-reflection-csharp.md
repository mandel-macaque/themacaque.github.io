# Detecting a nullable Type via reflection

Lately we have been enabling, [step](https://github.com/xamarin/xamarin-macios/pull/15163) by [step](https://github.com/xamarin/xamarin-macios/pull/15162), nullability on xamarin-macios. 
The idea being that the nullability annotations on our API will help Xamarin iOS and macOS users reduce their runtime issues due to a unexpected NRE (NullReferenceException). 
But xamarin-macios is special, most of the API is generated from interfaces, which makes the process more complicated.

---

## How is Xamarin.iOS and Xamarin.Mac built

Generating bindings for an API as big as the one provided by Apple in their platforms is a incredibly hard thing to do, on top of already an impossible task,
one of the main goals of Xamarin back in the day was to provide next day support. As you can imaging, tackling such a task manually is
nearly impossible without a huge amount of developers. Thankfully, from very early in the project, the bindings were mostly generated.
I cannot say who were the original inceptors of the design but I have a very strong feeling (and some git logs) that the original idea was
from [Miguel](https://twitter.com/migueldeicaza). Understanding how Xamarin on iOS is build is important to understand why it has taking us 
so long to enable nullability over all the API.

Building Xamarin iOS is based on a very smart design. Rather than manually writing all the [PInvokes](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/pinvoke) 
manually, the project uses a what I like to call a two pass compilation. The process is as follows:

1. Compile an intermediate assembly that contains the API definitions to be bound. This APIs are a contract (using interfaces) of the code that needs to be
   generated. The idea is that this intermediate assemblies contain all the required information to later use reflection to infer the APIs to bind. Because
   not all can be guessed from the [Reflection types](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/reflection) from C#, the
   API definitions use a number of attribute to add metadata. The metadata goes from what platforms support a given API, to if an async method can be provided.
2. There is a command line application that we call the generator that loads the intermediate assembly via reflection. The generator uses reflection to get all the
   data it needs and writes all the glue code needed to create a full API.
3. The last step of the build is to gather all those generated files (the use the extension *.g.cs) and combine them with some manual code (Xamarin is not a 1:1 map
   of Apples APIs we add some extra APIs or some csharp sugar) and create the last file.

The following code is an example of one of those API definitions:

```csharp
[NoWatch, NoTV, iOS (10, 0)]
[MacCatalyst (13, 1)]
[DisableDefaultCtor]
[BaseType (typeof (UIFeedbackGenerator))]
interface UIImpactFeedbackGenerator {

    [Export ("initWithStyle:")]
     NativeHandle Constructor (UIImpactFeedbackStyle style);

    [Export ("impactOccurred")]
    void ImpactOccurred ();

    [iOS (13, 0)]
    [MacCatalyst (13, 1)]
    [Export ("impactOccurredWithIntensity:")]
    void ImpactOccurred (nfloat intensity);
}
```

And the corresponding generated code:

```csharp
[Register("UIImpactFeedbackGenerator", true)]
[Unavailable (PlatformName.WatchOS, ObjCRuntime.PlatformArchitecture.All)]
[Unavailable (PlatformName.TvOS, ObjCRuntime.PlatformArchitecture.All)]
[Introduced (PlatformName.iOS, 10,0, ObjCRuntime.PlatformArchitecture.All)]
public unsafe partial class UIImpactFeedbackGenerator : UIFeedbackGenerator {
        [BindingImpl (BindingImplOptions.GeneratedCode | BindingImplOptions.Optimizable)]
        static readonly IntPtr class_ptr = Class.GetHandle ("UIImpactFeedbackGenerator");
        public override IntPtr ClassHandle { get { return class_ptr; } }
        [BindingImpl (BindingImplOptions.GeneratedCode | BindingImplOptions.Optimizable)]
        [EditorBrowsable (EditorBrowsableState.Advanced)]
        protected UIImpactFeedbackGenerator (NSObjectFlag t) : base (t)
        {
        }

        [BindingImpl (BindingImplOptions.GeneratedCode | BindingImplOptions.Optimizable)]
        [EditorBrowsable (EditorBrowsableState.Advanced)]
        protected internal UIImpactFeedbackGenerator (IntPtr handle) : base (handle)
        {
        }

        [Export ("initWithStyle:")]
        [BindingImpl (BindingImplOptions.GeneratedCode | BindingImplOptions.Optimizable)]
        public UIImpactFeedbackGenerator (UIImpactFeedbackStyle style)
                : base (NSObjectFlag.Empty)
        {
                global::UIKit.UIApplication.EnsureUIThread ();
                if (IsDirectBinding) {
                        InitializeHandle (global::ObjCRuntime.Messaging.IntPtr_objc_msgSend_IntPtr (this.Handle, Selector.GetHandle ("initWithStyle:"), (IntPtr) (long) style), "initWithStyle:");
                } else {
                        InitializeHandle (global::ObjCRuntime.Messaging.IntPtr_objc_msgSendSuper_IntPtr (this.SuperHandle, Selector.GetHandle ("initWithStyle:"), (IntPtr) (l
ong) style), "initWithStyle:");
                }
        }
        [Export ("impactOccurred")]
        [BindingImpl (BindingImplOptions.GeneratedCode | BindingImplOptions.Optimizable)]
        public virtual void ImpactOccurred ()
        {
                global::UIKit.UIApplication.EnsureUIThread ();
                if (IsDirectBinding) {
                        global::ObjCRuntime.Messaging.void_objc_msgSend (this.Handle, Selector.GetHandle ("impactOccurred"));
                } else {
                        global::ObjCRuntime.Messaging.void_objc_msgSendSuper (this.SuperHandle, Selector.GetHandle ("impactOccurred"));
                }
        }
        [Export ("impactOccurredWithIntensity:")]
        [Introduced (PlatformName.iOS, 13,0, ObjCRuntime.PlatformArchitecture.All)]
        [BindingImpl (BindingImplOptions.GeneratedCode | BindingImplOptions.Optimizable)]
        public virtual void ImpactOccurred (nfloat intensity)
        {
        #if ARCH_32
                throw new PlatformNotSupportedException ("This API is not supported on this version of iOS");
        #else
                global::UIKit.UIApplication.EnsureUIThread ();
                if (IsDirectBinding) {
                        global::ObjCRuntime.Messaging.void_objc_msgSend_nfloat (this.Handle, Selector.GetHandle ("impactOccurredWithIntensity:"), intensity);
                } else {
                        global::ObjCRuntime.Messaging.void_objc_msgSendSuper_nfloat (this.SuperHandle, Selector.GetHandle ("impactOccurredWithIntensity:"), intensity);
                }
        #endif
        }
} /* class UIImpactFeedbackGenerator */
```

**PS:** In this post I am ignoring the amazing work that [Rolf](https://twitter.com/rolfkvinge) does in teh runtime and the native code needed to write the trampolines
between C# and ObjC, but that is probably a good topic for another post.
**PS PS**: All this work was done A LONG time before code generators were a thing, and I would argue they are not powerful enough for our use case.

## Nullability before dotnet had nullability

Full nullability support to dotnet (that includes references types) was added in C# 7 yet objectiveC had Nullability annotations earlier than that. Perse that was not an issue
for Xamarin since we could annotate the not null parameters in our API definitions and generate the right code, for example:

```csharp
[NoWatch, NoTV, iOS (15, 0), MacCatalyst (15, 0)]
[BaseType (typeof (UIPresentationController))]
[DisableDefaultCtor]
interface UISheetPresentationController {

    [Export ("initWithPresentedViewController:presentingViewController:")]
    [DesignatedInitializer]
    NativeHandle Constructor (UIViewController presentedViewController, [NullAllowed] UIViewController presentingViewController);

    // rest of the API

}
```

That would generated the following:
```csharp
[Export ("initWithPresentedViewController:presentingViewController:")]
[DesignatedInitializer]
[BindingImpl (BindingImplOptions.GeneratedCode | BindingImplOptions.Optimizable)]
public UISheetPresentationController (UIViewController presentedViewController, UIViewController? presentingViewController)
        : base (NSObjectFlag.Empty)
{
#if ARCH_32
        throw new PlatformNotSupportedException ("This API is not supported on this version of iOS");
#else
        global::UIKit.UIApplication.EnsureUIThread ();
        var presentedViewController__handle__ = presentedViewController!.GetNonNullHandle (nameof (presentedViewController));
        var presentingViewController__handle__ = presentingViewController.GetHandle ();
        if (IsDirectBinding) {
                InitializeHandle (global::ObjCRuntime.Messaging.IntPtr_objc_msgSend_IntPtr_IntPtr (this.Handle, Selector.GetHandle ("initWithPresentedViewController:
presentingViewController:"), presentedViewController__handle__, presentingViewController__handle__), "initWithPresentedViewController:presentingViewController:");
        } else {
                InitializeHandle (global::ObjCRuntime.Messaging.IntPtr_objc_msgSendSuper_IntPtr_IntPtr (this.SuperHandle, Selector.GetHandle ("initWithPresentedViewC
ontroller:presentingViewController:"), presentedViewController__handle__, presentingViewController__handle__), "initWithPresentedViewController:presentingViewController:");
        }
#endif
}
```

This approached worked very very well for those parts of the API we control, and it was great when nullability for reference types did not exist. Yet, the approach before
nullability had a few annoying corner cases. There is no way to add attributes to the following:

### Generic type definitions

You cannot do:
```csharp
public Dictionary<string, [NotNull] MyObject> Data { get; set;}
```

### Delegates

This is perse a limitation of the generator, but the following does not work:

```csharp
delegate NSComparisonResult NSComparator ([NotNull] NSObject obj1, [NotNull] NSObject obj2);
```

Once nullability was added to C# 7 we wanted to add annotations to all the code, and while we supported a number of uses cases, using attributes would not fix
the situation in which we had to annotate generic types.

## Moving forward

dotnet 6 brought with it [NullabilityInfoContext](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.nullabilityinfocontext?view=net-7.0) the API we needed to
be able to use ? as a way to annotate nullability in our API definitions, the problem, Xamarin iOS had to maintain support
to code older than dotnet 6. This supposed a problem, we could nto use the new APIs, yet it gave me an idea. Rather that doing some complicated use of pramgas or two different
versions of the generator, I decided to do my own implementation of the NullabilityInfoContext for pre dotnet 6 code. That was added and landed in this [PR](https://github.com/xamarin/xamarin-macios/pull/17384).

The code follows the [documentation from the dotnet team](https://github.com/dotnet/roslyn/blob/main/docs/features/nullable-metadata.md) to read the nullability metadata. In reality,
the process is very simple. 

1. If we are looking at a value type. Check if the compiler converted int? into Nullable<int>
2. If we are looking at a reference type read the CustomAttributeData from the declaring type and decide accordingly.

The two attributes we want to look at are:

- System.Runtime.CompilerServices.NullableAttribute
- System.Runtime.CompilerServices.NullableContextAttribute

Both of the use bytes to let you know what nullability state the type has, being:

- 0: Unknown
- 1: Not nullable
- 2: Nullable

Teh implementation I wrote is very simple and uses recursion, could have been implemented as a stack, but I prefer clarity for this code which should die soon. Ideally, this implementation
will only be used by the generator during the time we have to compile it pre dotnet6 (we already have a dotnet 6 version). Here is the very simple implementation:

```csharp
using System;
using System.Collections.Generic;
using System.Collections.ObjectModel;
using System.Linq;
using System.Reflection;

#nullable enable

namespace bgen {

	// extension shared between NET and !NET make our lives a little easier
	public static class NullabilityInfoExtensions {
		public static bool IsNullable (this NullabilityInfo self)
			=> self.ReadState == NullabilityState.Nullable;
	}
#if !NET

	// from: https://github.com/dotnet/roslyn/blob/main/docs/features/nullable-metadata.md
	public enum NullabilityState : byte {
		Unknown = 0,
		NotNull = 1,
		Nullable = 2,
	}

	public class NullabilityInfo {
		public NullabilityState ReadState { get; set; } = NullabilityState.NotNull;
		public Type? Type { get; set; }
		public NullabilityInfo? ElementType { get; set; }
		public NullabilityInfo []? GenericTypeArguments { get; set; }
	}

	// helper class that allows to check the custom attrs in a method,property or field
	// in order for the generator to support ? in the bindings definition. This class should be
	// replaced by NullabilityInfoContext from the dotnet6 in the future. The API can be maintained (hopefully).
	public class NullabilityInfoContext {

		static readonly string NullableAttributeName = "System.Runtime.CompilerServices.NullableAttribute";
		static readonly string NullableContextAttributeName = "System.Runtime.CompilerServices.NullableContextAttribute";

		public NullabilityInfoContext () { }

		public NullabilityInfo Create (PropertyInfo propertyInfo)
			=> Create (propertyInfo.PropertyType, propertyInfo.DeclaringType, propertyInfo.CustomAttributes);

		public NullabilityInfo Create (FieldInfo fieldInfo)
			=> Create (fieldInfo.FieldType, fieldInfo.DeclaringType, fieldInfo.CustomAttributes);

		public NullabilityInfo Create (ParameterInfo parameterInfo)
			=> Create (parameterInfo.ParameterType, parameterInfo.Member, parameterInfo.CustomAttributes);

		static NullabilityInfo Create (Type memberType, MemberInfo? declaringType,
			IEnumerable<CustomAttributeData>? customAttributes, int depth = 0)
		{
			var info = new NullabilityInfo { Type = memberType };

			if (memberType.IsByRef) {
				var e = memberType.GetElementType ();
				info = Create (e, declaringType, customAttributes);
				// override the returned type because it is returning e and not ref e
				info.Type = memberType;
				return info;
			}

			if (memberType.IsArray) {
				info.ElementType = Create (memberType.GetElementType ()!, declaringType, customAttributes, depth + 1);
			}

			if (memberType.IsGenericType) {
				// we need to get the nullability type of each of the generics, we use an array to make sure
				// order is kept
				var genericArguments = memberType.GetGenericArguments ();
				info.GenericTypeArguments = new NullabilityInfo [genericArguments.Length];
				for (int i = 0; i < genericArguments.Length; i++) {
					info.GenericTypeArguments [i] = Create (genericArguments [i], declaringType,
						customAttributes, depth + (i + 1)); // the depth can be complicated
				}
			}

			// there are two possible cases we have to take care of:
			//
			// 1. ValueType: If we are dealing with a value type the compiler converts it to Nullable<ValueType>
			// 2. ReferenceType: Ret types do not use an interface, but a custom attr is used in the signature

			if (memberType.IsValueType) {
				var nullableType = Nullable.GetUnderlyingType (memberType);
				info.ReadState = nullableType != null ? NullabilityState.Nullable : NullabilityState.NotNull;
				info.Type = memberType;
				return info;
			}

			// at this point, we have to use the attributes (metadata) added by the compiler to decide if we have a
			// nullable type or not from a ref type. From https://github.com/dotnet/roslyn/blob/main/docs/features/nullable-metadata.md
			//
			// The byte[] is constructed as follows:
			// Reference type: the nullability (0, 1, or 2), followed by the representation of the type arguments in order including containing types
			// Nullable value type: the representation of the type argument only
			// Non-generic value type: skipped
			// 	Generic value type: 0, followed by the representation of the type arguments in order including containing types
			// Array: the nullability (0, 1, or 2), followed by the representation of the element type
			// Tuple: the representation of the underlying constructed type
			// 	Type parameter reference: the nullability (0, 1, or 2, with 0 for unconstrained type parameter)

			// interesting case when we have constrains from a generic method
			if ((customAttributes?.Count () ?? 0) == 0 && memberType.CustomAttributes is not null)
				customAttributes = memberType.CustomAttributes;

			var nullable = customAttributes?.FirstOrDefault (
				x => x.AttributeType.FullName == NullableAttributeName);

			var flag = NullabilityState.Unknown;
			if (nullable is not null && nullable.ConstructorArguments.Count == 1) {
				var attributeArgument = nullable.ConstructorArguments [0];
				if (attributeArgument.ArgumentType == typeof (byte [])) {
					var args = (ReadOnlyCollection<CustomAttributeTypedArgument>) attributeArgument.Value!;
					if (args.Count > 0 && args [depth].ArgumentType == typeof (byte)) {
						flag = (NullabilityState) args [depth].Value;

					}
				} else if (attributeArgument.ArgumentType == typeof (byte)) {
					flag = (NullabilityState) attributeArgument.Value!;
				}

				info.Type = memberType;
				info.ReadState = flag;
				return info;
			}

			// we are using the context of the declaring type to decide if the type is nullable
			for (var type = declaringType; type is not null; type = type.DeclaringType) {
				var context = type.CustomAttributes
					.FirstOrDefault (x => x.AttributeType.FullName == NullableContextAttributeName);
				if (context is null ||
					context.ConstructorArguments.Count != 1 ||
					context.ConstructorArguments [0].ArgumentType != typeof (byte))
					continue;
				if (NullabilityState.Nullable == (NullabilityState) context.ConstructorArguments [0].Value!) {
					info.Type = memberType;
					info.ReadState = NullabilityState.Nullable;
					return info;
				}
			}

			// we need to consider the generic constrains
			if (!memberType.IsGenericParameter)
				return info;

			// if we do have a custom null atr in any of them, use it
			if (memberType.GenericParameterAttributes.HasFlag (GenericParameterAttributes.NotNullableValueTypeConstraint))
				info.ReadState = NullabilityState.NotNull;

			return info;
		}
	}
#endif

}
```

I have not complited all the work, but soon I should have added the needed support for the generator which will fix a number [of](https://github.com/xamarin/xamarin-macios/issues/4826)
[API](https://github.com/xamarin/xamarin-macios/issues/9814) [bugs](https://github.com/xamarin/xamarin-macios/issues/17109).

This post has to goals:

1. Let you know we are working on it and you will get nullability soon.
2. Provide the sample code needed to understand what is the c# compiler doing when you use ? in you value types and what it does in your reference types.

I hope you find it as interesting as I do. If you already knew about this, sorry for making you read.
