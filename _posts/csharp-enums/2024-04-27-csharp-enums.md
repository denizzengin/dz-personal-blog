---
title: Understanding the Fundamentals of Enums in C# 🚀
date: 2024-04-27
modified: 2024-04-27
tags: [.NET, c#, enums]
description: You’ll find the details of enums in c#
# image: "/net-keyed-service/keyed-service-output.png"
comments: true
---

This blog post delves into the fundemental concepts of enums in C#, offering a clear understanding of their syntax, usage, and best practices.

**Microsoft** official description for enums is,
> “An enumeration type (or enum type) is a value type defined 
by a set of named constants of the underlying integral numeric type”. 

The next question is why we need to be named constant values in our code base.
The reasons are readability, maintainability and more control over it. These things came to my mind but most likely there is more than. You can write your code with hard-coded `strings` instead of `enums`.

**Note:** There are a few naming conventions for `enums` and I highly recommend them when defining enums. 
It could be confusing to define them without rules in your code base. 
For example, one developer adds an enum like `AccountTypes` and another adds `CustomerType`. The difference is obvious the earlier one is plural but the other is singular form. 

> 
✔️ DO use a singular type name for an enumeration unless its values are bit fields.
>
✔️ DO use a plural type name for an enumeration with bit fields as values, also called flags enum.
>
❌ DO NOT use an "Enum" suffix in enum type names.

You can find the most updated recommendations [here](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/names-of-classes-structs-and-interfaces) by Microsoft.


Let’s imagine you are using these strings in a lot of flow controls and just seeing it like below,
{% highlight c# %}
/*
    primary student : 1
    secondary student : 2
*/
if(firstStudent == "1") // you can use int or string whichever you want. These are both the same.
{
    // do some work here
}
{% endhighlight %}

It could be better with enums regarding **readability, maintainability and clear understanding** for other developers who share the code base with you. For instance, this is the simplest and purest definition of enums,

{% highlight c# %}
public enum StudentType 
{
    Unknown,    // 0
    Primary,    // 1 
    College,    // 2 
    Univertsity // 3
}
{% endhighlight %}

In c# world, enum is `ValueType` and it sets the default underlying type as `int(Int32)`. You can imagine that enum is a group of key-values. In database, you can hold them as numeric and it's fine but in your code base, using enums enrich them much more regarding readability.

If you don't give a initial value to enum it begins with 0. So, `StudentType.Unknown` 's underlying type is `0`. Also, other enum values will be incremented one by one.

**Note :** If you are sure of the range of values ​​that the number value can take you should pick the best integral type that fits your need. In our sample, We know StudentType cannot be more than `4`. Therefore, We should inherit from byte because byte can be up to `256(2^8)` and that’s enough.(`Int32` could be up to `-2,147,483,648 to 2,147,483,647`) In this way, I would have used the memory better than the previous one.

**Note :** integral types refer to numeric values. [Here](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/integral-numeric-types) you can find some pre-defined integral numeric types. So, you cannot inherit an enum from a `string` or `char`.

{% highlight c# %}
public enum StudentType : byte
{
    Unknown,    // 0
    Primary,    // 1 
    College,    // 2 
    Univertsity // 3
}
{% endhighlight %}

We can deep dive into enums now. 

- How can we store more than one enum value in a variable? 
- What's the use case of it?

Fortunately, c# provides a solution via the `Flags` attribute and it's used when we need to use some related enum values 
as an inner group. For instance, we need to add a feature which checks `Student` is `Primary` or `College` and returns true if yes otherwise returns false and goes a different flow.

{% highlight c# %}
// First solution
[Flags]
public enum StudentType 
{
    Unknown,
    Primary,
    College,
    Univertsity
}
{% endhighlight %}

We added the `[Flags]` attribute. It enables enum values as bitfields and provides us storing more than one enum value in a variable.
However, still there is a problem. Run below code,

{% highlight c# %}
StudentType studentTypeNone = StudentType.Unknown;
StudentType studentTypePrimary = StudentType.Primary;
StudentType studentTypeCollege = StudentType.College;

StudentType groupedType =  studentTypeNone | studentTypePrimary | studentTypeCollege ;
Console.WriteLine($"{groupedType}");
{% endhighlight %}

You see below output,
```
Univertsity
```

We would want to store multiple enum values in our variable but accidentally it produced the `Univertsity`. It can cause our programs to behave unexpectedly. It happened because `Flags` attribute is not enough itself. We need to modify `StudentType` enum numeric values.

Previous values and binary representations as below,

```bash

 Input A | Input B | Output
 --------|---------|-------
   0     |    0    |   0
   0     |    1    |   1
   1     |    0    |   1
   1     |    1    |   1

                     00000000 = (None)
                     00000001 = (Primary)
bitwise logical or = 00000001
                     00000010 = (College)
bitwise logical or = 00000011 (is equal to 3 base on 10 system)

                     00000011 = (University)
```



It's shift left operation. We can prevent this duplication give values as below,

{% highlight c# %}
[Flags]
public enum StudentType 
{
    Unknown = 1, 
    Primary = 2,
    College = 4,
    Univertsity = 8
}
{% endhighlight %}
**Note:** You should not give 0 value to the first if you want to contain that value in group because it's inefective element.

Now you will see output like,
```
Unknown, Primary, College
```

Also, another way of defining values using the shift left operator. It could be useful when there are many enum values and it's hard to calculate the next value.

{% highlight c# %}
[Flags]
public enum StudentType 
{
    Unknown = 1,           // 0001
    Primary = 1 << 1,      // 0010 shift one step to left
    College = 1 << 2,      // 0010 shift two step to left
    Univertsity = 1 << 3,  // 0100 shift two step to left
}
{% endhighlight %}

Now you will see the same output as previous,
```
Unknown, Primary, College
```

Here is the response to how we check enum group contains or does not a given enum,
 ```c#
  Console.WriteLine(groupedType.HasFlag(StudentType.Univertsity)); // HasFlag 
 ```

# Also, c# enums doesn't support implicit conversion. The below code gives compiling error,
{% highlight c# %}
StudentType studentTypeNone = StudentType.Unknown;
int studentTypeNoneUnderlyingValue = studentTypeNone; // invalid
{% endhighlight %}

```
Error	CS0266	Cannot implicitly convert type 'EnumsSample.StudentType' to 'int'. An explicit conversion exists...
```

Fixed code,

{% highlight c# %}
StudentType studentTypeNone = StudentType.Unknown;
int studentTypeNoneUnderlyingValue = (int)studentTypeNone; // explicit cast
{% endhighlight %}


In conclusion, There is more than one way or solution to your problem based on your needs.

  > **x** If you need to group some enums, you can use `Flags` attribute but you should be careful giving them values.

  > **x** If you don't need to group, you can go without Flags event and even without giving any values.


There is one more trick alternative to enums which `Microsoft` provide us to create the custom enum base enum type and add more functionality like implicit casting. I plan to write a separate blog post about this and want to keep this short😊

### Sample Repository

- [c# enums](https://github.com/denizzengin/blog-csharp-enums)


### Resources

- [Enumeration types (C# reference)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/enum)
- [Naming Enumerations](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/names-of-classes-structs-and-interfaces)