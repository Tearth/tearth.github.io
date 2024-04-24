---
title: "GetHashCode inside CLR: Value types"
date: "2020-06-15"
description: "Magic hidden in the internal implementation of GetHashCode in CLR - how exactly this method works for reference and value types."
categories:
  - ".NET internals"
  - "Performance comparisons"
tags:
  - ".NET"
  - ".NET Core"
  - "CLR"
  - "GetHashCode"
issue: 4
sidebar: "right"
---

In the [previous article](https://tearth.dev/posts/gethashcode-inside-clr-reference-types/), we talked a bit about hash codes and how they are implemented for reference types - it turned out that it's just a simple multiplication of thread ID and a random number. Today, we will do the same thing for value types, which are far more complex due to their representation in memory. In the end, I will show a small benchmark to prove that every struct defined by the programmer should override [GetHashCode](https://docs.microsoft.com/en-us/dotnet/api/system.object.gethashcode?view=netcore-3.1) method. Time to dig into [CLR source code](https://github.com/dotnet/runtime)!

<!--more-->

## Inside CLR

Let's start in the same way as before. Reference type used [RuntimeHelpers.GetHashCode](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.runtimehelpers.gethashcode?view=netcore-3.1) within its [Object.GetHashCode](https://docs.microsoft.com/en-us/dotnet/api/system.object.gethashcode?view=netcore-3.1) implementation to generate hash code. In value types, there is no any proxy - method is directly marked as implemented by CLR:

{{< highlight csharp "linenos=table" >}}
/// <summary>Returns the hash code for this instance.</summary>
/// <returns>A 32-bit signed integer that is the hash code for this instance.</returns>
[SecuritySafeCritical]
[__DynamicallyInvokable]
[MethodImpl(MethodImplOptions.InternalCall)]
public override extern int GetHashCode();
{{< / highlight >}}

We can see a couple of interesting attributes which weren't present earlier. [SecuritySafeCritical](https://docs.microsoft.com/en-us/dotnet/api/system.security.securitysafecriticalattribute?view=netcore-3.1) attribute indicates that method can be accessed by partially trusted types and members - but as we can read in the documentation, this doesn't have any effect in .NET Core. The next one is `__DynamicallyInvokable` attribute which is not officially documented, however basing on the [GitHub issue](https://github.com/dotnet/runtime/issues/30809) and [StackOverflow post](https://stackoverflow.com/a/12552079/12928142) we can assume that it's used to mark methods as compatible with some sort of internal Windows 8 optimization features. The third one, [MethodImplAttribute](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.methodimplattribute?view=netcore-3.1), is already known for us and indicates that implementation is considered as part of the CLR itself. Mapping of C# method into native one in CLR can be found in [/src/coreclr/src/vm/ecalllist.h](https://github.com/dotnet/runtime/blob/master/src/coreclr/src/vm/ecalllist.h#L131) file:

{{< highlight cpp "linenos=table" >}}
FCFuncElement("GetHashCode", ValueTypeHelper::GetHashCode)
{{< / highlight >}}

As the next step, we have to find a definition of the `ValueTypeHelper::GetHashCode` native method - it can be found in [/src/coreclr/src/vm/comutilnative.cpp](https://github.com/dotnet/runtime/blob/ab49e0f9dcd56958148827c6b47428b56187b5f8/src/coreclr/src/vm/comutilnative.cpp#L1984) file:

{{< highlight cpp "linenos=table,hl_lines=48 53" >}}
// The default implementation of GetHashCode() for all value types.
// Note that this implementation reveals the value of the fields.
// So if the value type contains any sensitive information it should
// implement its own GetHashCode().
FCIMPL1(INT32, ValueTypeHelper::GetHashCode, Object* objUNSAFE)
{
    FCALL_CONTRACT;

    if (objUNSAFE == NULL)
        FCThrow(kNullReferenceException);

    OBJECTREF obj = ObjectToOBJECTREF(objUNSAFE);
    VALIDATEOBJECTREF(obj);

    INT32 hashCode = 0;
    MethodTable *pMT = objUNSAFE->GetMethodTable();

    // We don't want to expose the method table pointer in the hash code
    // Let's use the typeID instead.
    UINT32 typeID = pMT->LookupTypeID();
    if (typeID == TypeIDProvider::INVALID_TYPE_ID)
    {
        // If the typeID has yet to be generated, fall back to GetTypeID
        // This only needs to be done once per MethodTable
        HELPER_METHOD_FRAME_BEGIN_RET_1(obj);
        typeID = pMT->GetTypeID();
        HELPER_METHOD_FRAME_END();
    }

    // To get less colliding and more evenly distributed hash codes,
    // we munge the class index with two big prime numbers
    hashCode = typeID * 711650207 + 2506965631U;

    BOOL canUseFastGetHashCodeHelper = FALSE;
    if (pMT->HasCheckedCanCompareBitsOrUseFastGetHashCode())
    {
        canUseFastGetHashCodeHelper = pMT->CanCompareBitsOrUseFastGetHashCode();
    }
    else
    {
        HELPER_METHOD_FRAME_BEGIN_RET_1(obj);
        canUseFastGetHashCodeHelper = CanCompareBitsOrUseFastGetHashCode(pMT);
        HELPER_METHOD_FRAME_END();
    }

    if (canUseFastGetHashCodeHelper)
    {
        hashCode ^= FastGetValueTypeHashCodeHelper(pMT, obj->UnBox());
    }
    else
    {
        HELPER_METHOD_FRAME_BEGIN_RET_1(obj);
        hashCode ^= RegularGetValueTypeHashCode(pMT, obj->UnBox());
        HELPER_METHOD_FRAME_END();
    }

    return hashCode;
}
FCIMPLEND
{{< / highlight >}}

First, CLR prepares and reads the ID of the target type using the `GetTypeID` method - it's used to calculate the base hash code (`hashCode = typeID * 711650207 + 2506965631U`) which will be proceeded later with the algorithm appropriate for the situation.

Now, we have a set of mysterious methods which check what exactly do we have to do now with our calculated earlier hash code:
 - [pMT->HasCheckedCanCompareBitsOrUseFastGetHashCode](https://github.com/dotnet/runtime/blob/611555e00a80b4b60030fe4483b81494664975b8/src/coreclr/src/vm/methodtable.h#L1069) - returns true the specified type contains [enum_flag_CanCompareBitsOrUseFastGetHashCode](https://github.com/dotnet/runtime/blob/611555e00a80b4b60030fe4483b81494664975b8/src/coreclr/src/vm/methodtable.h#L313) flag with valid value.
 - [pMT->CanCompareBitsOrUseFastGetHashCode](https://github.com/dotnet/runtime/blob/611555e00a80b4b60030fe4483b81494664975b8/src/coreclr/src/vm/methodtable.h#L1046) - returns true if the [enum_flag_CanCompareBitsOrUseFastGetHashCode](https://github.com/dotnet/runtime/blob/611555e00a80b4b60030fe4483b81494664975b8/src/coreclr/src/vm/methodtable.h#L313) flag is set - it means, that the specified type is simple and its byte representation can be directly used to generate hash code or comparison.
 - [CanCompareBitsOrUseFastGetHashCode(pMT)](https://github.com/dotnet/runtime/blob/ab49e0f9dcd56958148827c6b47428b56187b5f8/src/coreclr/src/vm/comutilnative.cpp#L1734) - checks if the type is simple and its bytes can be directly used to generate hash code or comparison.
 
CLR implements two different ways to calculate hash code for value types: **fast** and **regular**. The first one is the simplest because it operates directly on the bytes in memory which represents the specified type instance. The second one is more complex because it has to deal with the internal reference types, but I will describe it more later. Now we will try to find out how do we know which algorithm from those two should we use. Let's see the [CanCompareBitsOrUseFastGetHashCode(pMT)](https://github.com/dotnet/runtime/blob/ab49e0f9dcd56958148827c6b47428b56187b5f8/src/coreclr/src/vm/comutilnative.cpp#L1734) implementation:

{{< highlight cpp "linenos=table" >}}
static BOOL CanCompareBitsOrUseFastGetHashCode(MethodTable* mt)
{
    CONTRACTL
    {
        THROWS;
        GC_TRIGGERS;
        MODE_COOPERATIVE;
    } CONTRACTL_END;

    _ASSERTE(mt != NULL);

    if (mt->HasCheckedCanCompareBitsOrUseFastGetHashCode())
    {
        return mt->CanCompareBitsOrUseFastGetHashCode();
    }

    if (mt->ContainsPointers()
        || mt->IsNotTightlyPacked())
    {
        mt->SetHasCheckedCanCompareBitsOrUseFastGetHashCode();
        return FALSE;
    }

    MethodTable* valueTypeMT = MscorlibBinder::GetClass(CLASS__VALUE_TYPE);
    WORD slotEquals = MscorlibBinder::GetMethod(METHOD__VALUE_TYPE__EQUALS)->GetSlot();
    WORD slotGetHashCode = MscorlibBinder::GetMethod(METHOD__VALUE_TYPE__GET_HASH_CODE)->GetSlot();

    // Check the input type.
    if (HasOverriddenMethod(mt, valueTypeMT, slotEquals)
        || HasOverriddenMethod(mt, valueTypeMT, slotGetHashCode))
    {
        mt->SetHasCheckedCanCompareBitsOrUseFastGetHashCode();

        // If overridden Equals or GetHashCode found, stop searching further.
        return FALSE;
    }

    BOOL canCompareBitsOrUseFastGetHashCode = TRUE;

    // The type itself did not override Equals or GetHashCode, go for its fields.
    ApproxFieldDescIterator iter = ApproxFieldDescIterator(mt, ApproxFieldDescIterator::INSTANCE_FIELDS);
    for (FieldDesc* pField = iter.Next(); pField != NULL; pField = iter.Next())
    {
        if (pField->GetFieldType() == ELEMENT_TYPE_VALUETYPE)
        {
            // Check current field type.
            MethodTable* fieldMethodTable = pField->GetApproxFieldTypeHandleThrowing().GetMethodTable();
            if (!CanCompareBitsOrUseFastGetHashCode(fieldMethodTable))
            {
                canCompareBitsOrUseFastGetHashCode = FALSE;
                break;
            }
        }
        else if (pField->GetFieldType() == ELEMENT_TYPE_R8
                || pField->GetFieldType() == ELEMENT_TYPE_R4)
        {
            // We have double/single field, cannot compare in fast path.
            canCompareBitsOrUseFastGetHashCode = FALSE;
            break;
        }
    }

    // We've gone through all instance fields. It's time to cache the result.
    // Note SetCanCompareBitsOrUseFastGetHashCode(BOOL) ensures the checked flag
    // and canCompare flag being set atomically to avoid race.
    mt->SetCanCompareBitsOrUseFastGetHashCode(canCompareBitsOrUseFastGetHashCode);

    return canCompareBitsOrUseFastGetHashCode;
}
{{< / highlight >}}

This method returns true if the specified type can be processed using fast algorithm. Looking at its implementation, we can clearly separate a set of rules:
 - Use **fast** algorithm when:
   - The type is already checked and [enum_flag_CanCompareBitsOrUseFastGetHashCode](https://github.com/dotnet/runtime/blob/611555e00a80b4b60030fe4483b81494664975b8/src/coreclr/src/vm/methodtable.h#L313) flag is set to true.
   - All internal fields are value types.
   - there are no any holes between internal fields.
   - [Equals](https://docs.microsoft.com/en-us/dotnet/api/system.valuetype.equals?view=netcore-3.1) and [GetHashCode](https://docs.microsoft.com/en-us/dotnet/api/system.valuetype.gethashcode?view=netcore-3.1) aren't overridden.
 - Use **regular** algorithm when:
   - The type is already checked and [enum_flag_CanCompareBitsOrUseFastGetHashCode](https://github.com/dotnet/runtime/blob/611555e00a80b4b60030fe4483b81494664975b8/src/coreclr/src/vm/methodtable.h#L313) flag is set to false.
   - Some of the internal fields are reference types.
   - Internal fields has been aligned to improve performance.
   - [Equals](https://docs.microsoft.com/en-us/dotnet/api/system.valuetype.equals?view=netcore-3.1) or [GetHashCode](https://docs.microsoft.com/en-us/dotnet/api/system.valuetype.gethashcode?view=netcore-3.1) has been overridden.

First, let's see at the **fast** algorithm because it's easier to understand. We can found it in [FastGetValueTypeHashCodeHelper](https://github.com/dotnet/runtime/blob/ab49e0f9dcd56958148827c6b47428b56187b5f8/src/coreclr/src/vm/comutilnative.cpp#L1858) method:

{{< highlight cpp "linenos=table" >}}
static INT32 FastGetValueTypeHashCodeHelper(MethodTable *mt, void *pObjRef)
{
    CONTRACTL
    {
        NOTHROW;
        GC_NOTRIGGER;
        MODE_COOPERATIVE;
    } CONTRACTL_END;

    INT32 hashCode = 0;
    INT32 *pObj = (INT32*)pObjRef;

    // this is a struct with no refs and no "strange" offsets, just go through the obj and xor the bits
    INT32 size = mt->GetNumInstanceFieldBytes();
    for (INT32 i = 0; i < (INT32)(size / sizeof(INT32)); i++)
        hashCode ^= *pObj++;

    return hashCode;
}
{{< / highlight >}}

The algorithm is very simple and in fact, it's just a loop that takes all bytes which represents the type's instance and does XOR operation. It's worth noting that we don't distinguish every internal field separately - it would be a waste of time. 

Not let's see at the second algorithm, **regular**, implemented in [RegularGetValueTypeHashCode](https://github.com/dotnet/runtime/blob/ab49e0f9dcd56958148827c6b47428b56187b5f8/src/coreclr/src/vm/comutilnative.cpp#L1878) method:

{{< highlight cpp "linenos=table" >}}
static INT32 RegularGetValueTypeHashCode(MethodTable *mt, void *pObjRef)
{
    CONTRACTL
    {
        THROWS;
        GC_TRIGGERS;
        MODE_COOPERATIVE;
    } CONTRACTL_END;

    INT32 hashCode = 0;

    GCPROTECT_BEGININTERIOR(pObjRef);

    BOOL canUseFastGetHashCodeHelper = FALSE;
    if (mt->HasCheckedCanCompareBitsOrUseFastGetHashCode())
    {
        canUseFastGetHashCodeHelper = mt->CanCompareBitsOrUseFastGetHashCode();
    }
    else
    {
        canUseFastGetHashCodeHelper = CanCompareBitsOrUseFastGetHashCode(mt);
    }

    // While we shouln't get here directly from ValueTypeHelper::GetHashCode, if we recurse we need to
    // be able to handle getting the hashcode for an embedded structure whose hashcode is computed by the fast path.
    if (canUseFastGetHashCodeHelper)
    {
        hashCode = FastGetValueTypeHashCodeHelper(mt, pObjRef);
    }
    else
    {
        // it's looking ugly so we'll use the old behavior in managed code. Grab the first non-null
        // field and return its hash code or 'it' as hash code
        // <TODO> Note that the old behavior has already been broken for value types
        //              that is qualified for CanUseFastGetHashCodeHelper. So maybe we should
        //              change the implementation here to use all fields instead of just the 1st one.
        // </TODO>
        //
        // <TODO> check this approximation - we may be losing exact type information </TODO>
        ApproxFieldDescIterator fdIterator(mt, ApproxFieldDescIterator::INSTANCE_FIELDS);

        FieldDesc *field;
        while ((field = fdIterator.Next()) != NULL)
        {
            _ASSERTE(!field->IsRVA());
            if (field->IsObjRef())
            {
                // if we get an object reference we get the hash code out of that
                if (*(Object**)((BYTE *)pObjRef + field->GetOffsetUnsafe()) != NULL)
                {
                    PREPARE_SIMPLE_VIRTUAL_CALLSITE(METHOD__OBJECT__GET_HASH_CODE, (*(Object**)((BYTE *)pObjRef + field->GetOffsetUnsafe())));
                    DECLARE_ARGHOLDER_ARRAY(args, 1);
                    args[ARGNUM_0] = PTR_TO_ARGHOLDER(*(Object**)((BYTE *)pObjRef + field->GetOffsetUnsafe()));
                    CALL_MANAGED_METHOD(hashCode, INT32, args);
                }
                else
                {
                    // null object reference, try next
                    continue;
                }
            }
            else
            {
                CorElementType fieldType = field->GetFieldType();
                if (fieldType == ELEMENT_TYPE_R8)
                {
                    PREPARE_NONVIRTUAL_CALLSITE(METHOD__DOUBLE__GET_HASH_CODE);
                    DECLARE_ARGHOLDER_ARRAY(args, 1);
                    args[ARGNUM_0] = PTR_TO_ARGHOLDER(((BYTE *)pObjRef + field->GetOffsetUnsafe()));
                    CALL_MANAGED_METHOD(hashCode, INT32, args);
                }
                else if (fieldType == ELEMENT_TYPE_R4)
                {
                    PREPARE_NONVIRTUAL_CALLSITE(METHOD__SINGLE__GET_HASH_CODE);
                    DECLARE_ARGHOLDER_ARRAY(args, 1);
                    args[ARGNUM_0] = PTR_TO_ARGHOLDER(((BYTE *)pObjRef + field->GetOffsetUnsafe()));
                    CALL_MANAGED_METHOD(hashCode, INT32, args);
                }
                else if (fieldType != ELEMENT_TYPE_VALUETYPE)
                {
                    UINT fieldSize = field->LoadSize();
                    INT32 *pValue = (INT32*)((BYTE *)pObjRef + field->GetOffsetUnsafe());
                    for (INT32 j = 0; j < (INT32)(fieldSize / sizeof(INT32)); j++)
                        hashCode ^= *pValue++;
                }
                else
                {
                    // got another value type. Get the type
                    TypeHandle fieldTH = field->GetFieldTypeHandleThrowing();
                    _ASSERTE(!fieldTH.IsNull());
                    hashCode = RegularGetValueTypeHashCode(fieldTH.GetMethodTable(), (BYTE *)pObjRef + field->GetOffsetUnsafe());
                }
            }
            break;
        }
    }

    GCPROTECT_END();

    return hashCode;
}
{{< / highlight >}}

Here we can see one interesting thing. If CLR decides to use the regular version of hash code computing, then it will use only the first internal field. It means, if you have a structure with fields: Age (int), Height (int), Weight (int), and Ref (object), then only a change of Age's value will affect a value returned from the non-overridden [GetHashCode](https://docs.microsoft.com/en-us/dotnet/api/system.object.gethashcode?view=netcore-3.1) method. Let's see an example:

{{< highlight csharp "linenos=table" >}}
using System;
using System.Diagnostics;

namespace GetHashCodeCore
{
    public struct PersonWithRef
    {
        public readonly int Age;
        public readonly int Height;
        public readonly int Weight;
        public readonly object Ref;

        public PersonWithRef(int age, int height, int weight)
        {
            Age = age;
            Height = height;
            Weight = weight;
            Ref = new object();
        }
    }

    class Program
    {
        static void Main(string[] args)
        {
            var pRef1 = new PersonWithRef(20, 160, 60);
            var pRef2 = new PersonWithRef(20, 170, 70);
            var pRef3 = new PersonWithRef(30, 170, 70);

            Console.WriteLine($"PersonWithRef 1 hash code: {pRef1.GetHashCode()}");
            Console.WriteLine($"PersonWithRef 2 hash code: {pRef2.GetHashCode()}");
            Console.WriteLine($"PersonWithRef 3 hash code: {pRef3.GetHashCode()}");

            Console.Read();
        }
    }
}
{{< / highlight >}}

```
PersonWithRef 1 hash code: -1101417524
PersonWithRef 2 hash code: -1101417524
PersonWithRef 3 hash code: -1101417530
```

Two first persons have the same age and different height and width. What's more important, they have also the same hash code, which is consistent with the observations made in CLR source code. The third person has the same height and weight as the second person, but different age - because only the age is used to perform calculations, the hash code is different.

So let's summarize what we know at this moment in relation to C#. We will get hash code based on all instance bytes if the type is an integral or a struct (where all internal fields are also value types and there are no holes between them). If some of these requirements aren't fulfilled, then CLR will calculate hash code using the regular version of the algorithm (so only the first field will be used).

I wrote "integral" and not "simple type" not without reason - `bool`, `float` and `double` have their own overrided versions of [GetHashCode](https://docs.microsoft.com/en-us/dotnet/api/system.object.gethashcode?view=netcore-3.1), fortunelty written in pure C#. Let's check them:

{{< highlight csharp "linenos=table" >}}
// ---------
// bool
// ---------
public override int GetHashCode()
{
    return (m_value) ? True : False;
}

// ---------
// float
// ---------
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public override int GetHashCode()
{
    var bits = Unsafe.As<float, int>(ref Unsafe.AsRef(in m_value));

    // Optimized check for IsNan() || IsZero()
    if (((bits - 1) & 0x7FFFFFFF) >= 0x7F800000)
    {
        // Ensure that all NaNs and both zeros have the same hash code
        bits &= 0x7F800000;
    }

    return bits;
}

// ---------
// double
// ---------
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public override int GetHashCode()
{
    var bits = Unsafe.As<double, long>(ref Unsafe.AsRef(in m_value));

    // Optimized check for IsNan() || IsZero()
    if (((bits - 1) & 0x7FFFFFFFFFFFFFFF) >= 0x7FF0000000000000)
    {
        // Ensure that all NaNs and both zeros have the same hash code
        bits &= 0x7FF0000000000000;
    }

    return unchecked((int)bits) ^ ((int)(bits >> 32));
}
{{< / highlight >}}

As we can see, `bool` type just returns 1 if it's set to true or 0 if it's set to false. On the other side, `float` and `double` are a bit more complex because of IEEE 754 nature - the number can be positively zero or negatively zero. It would have no sense to have different hash codes for them because from the human perspective both are equal. Additionally, `double` does XOR operation on both 32-bit halves, which is understandable because the hash code is 32-bit.

## Benchmark

As we can see, there is a lot of checks and other operations to calculate hash code for a value type, even if CLR uses a simple fast version of the algorithm. That's why we should always override [GetHashCode](https://docs.microsoft.com/en-us/dotnet/api/system.object.gethashcode?view=netcore-3.1) in value types and provide our version which will be much faster. Let's prove it by modifying the previous code by adding a simple benchmark (using [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) library):

{{< highlight csharp "linenos=table" >}}
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Jobs;
using BenchmarkDotNet.Running;

namespace GetHashCodeCore
{
    public struct PersonWithoutGetHashCode
    {
        public readonly int Age;
        public readonly int Height;
        public readonly int Weight;

        public PersonWithoutGetHashCode(int age, int height, int weight)
        {
            Age = age;
            Height = height;
            Weight = weight;
        }
    }

    public struct PersonWithGetHashCode
    {
        public readonly int Age;
        public readonly int Height;
        public readonly int Weight;

        public PersonWithGetHashCode(int age, int height, int weight)
        {
            Age = age;
            Height = height;
            Weight = weight;
        }

        public override int GetHashCode()
        {
            return Age ^ Height ^ Weight;
        }
    }

    [DisassemblyDiagnoser]
    [SimpleJob(RuntimeMoniker.NetCoreApp31)]
    public class GetHashCodeTest
    {
        private PersonWithoutGetHashCode _personWithoutGetHashCode;
        private PersonWithGetHashCode _personWithGetHashCode;

        [Benchmark]
        public void TestPersonWithoutGetHashCode()
        {
            var result = _personWithoutGetHashCode.GetHashCode();
            _personWithoutGetHashCode = new PersonWithoutGetHashCode(result, 160, 50);
        }

        [Benchmark]
        public void TestPersonWithGetHashCode()
        {
            var result = _personWithGetHashCode.GetHashCode();
            _personWithGetHashCode = new PersonWithGetHashCode(result, 160, 50);
        }
    }

    class Program
    {
        public static void Main(string[] args)
        {
            BenchmarkRunner.Run<GetHashCodeTest>();
        }
    }
}
{{< / highlight >}}

We have two similar structures - one with default [GetHashCode](https://docs.microsoft.com/en-us/dotnet/api/system.object.gethashcode?view=netcore-3.1) implementation and one with our own. Every benchmark does two things: first, it calculates hash code for the appropriate struct instance, and second, it creates a new one with the `Age` value stored in the `result` variable. This prevents the compiler from precalculating elements or doing other optimizations.

```
BenchmarkDotNet=v0.12.1, OS=Windows 10.0.18363.959 (1909/November2018Update/19H2)
Intel Core i5-8300H CPU 2.30GHz (Coffee Lake), 1 CPU, 8 logical and 4 physical cores
.NET Core SDK=5.0.100-preview.5.20279.10
  [Host]        : .NET Core 3.1.6 (CoreCLR 4.700.20.26901, CoreFX 4.700.20.31603), X64 RyuJIT
  .NET Core 3.1 : .NET Core 3.1.6 (CoreCLR 4.700.20.26901, CoreFX 4.700.20.31603), X64 RyuJIT

Job=.NET Core 3.1  Runtime=.NET Core 3.1
```

|                       Method |       Mean |     Error |    StdDev | Code Size |
|----------------------------- |-----------:|----------:|----------:|----------:|
| TestPersonWithoutGetHashCode | 29.6406 ns | 0.2940 ns | 0.2750 ns |      73 B |
|    TestPersonWithGetHashCode |  0.1220 ns | 0.0065 ns | 0.0061 ns |      32 B |


The difference is very visible - default [GetHashCode](https://docs.microsoft.com/en-us/dotnet/api/system.object.gethashcode?view=netcore-3.1) (using CLR implementation) is several dozen times slower than our custom one. This is because CLR doesn't need to check anymore if some of the internal fields in the structure are a reference or there are some holes between them. We just XOR all fields and return the result.

Just for curious, we can check what assembly code has been generated for both methods. The first one is for `TestPersonWithoutGetHashCode`, and we can see that there is a lot of operations - this is just a preparations to call a `call 00007FFCA6AE3B50`, which is the internal call.

{{< highlight asm "linenos=table" >}}
; GetHashCodeCore.GetHashCodeTest.TestPersonWithoutGetHashCode()
    push      rsi
    sub       rsp,20
    mov       rsi,rcx
    mov       rcx,offset MT_GetHashCodeCore.PersonWithoutGetHashCode
    call      CORINFO_HELP_NEWSFAST
    add       rsi,8
    lea       rcx,[rax+8]
    mov       rdx,[rsi]
    mov       [rcx],rdx
    mov       edx,[rsi+8]
    mov       [rcx+8],edx
    mov       rcx,rax
    call      00007FFCA6AE3B50
    mov       [rsi],eax
    mov       dword ptr [rsi+4],0A0
    mov       dword ptr [rsi+8],32
    add       rsp,20
    pop       rsi
    ret
{{< / highlight >}}

The second one, related to `TestPersonWithGetHashCode`, has been inlined by the compiler and simply retrieves values from the struct instance and then does a XOR operation.

{{< highlight asm "linenos=table" >}}
; GetHashCodeCore.GetHashCodeTest.TestPersonWithGetHashCode()
    add       rcx,18
    mov       rax,rcx
    mov       edx,[rax]
    xor       edx,[rax+4]
    xor       edx,[rax+8]
    mov       [rcx],edx
    mov       dword ptr [rcx+4],0A0
    mov       dword ptr [rcx+8],32
    ret
{{< / highlight >}}

## Summary

This is the last part of the short "GetHashCode Inside CLR" series. We saw how `GetHashCode` method is implemented for reference and value types inside Common Language Runtime and how exactly these values are calculated. I think knowledge about nuances like this can be really useful for developers and help them to better understand the language.