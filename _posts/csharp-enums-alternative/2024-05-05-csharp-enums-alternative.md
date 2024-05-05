---
title: Enumeration, Alternative Way of Implementing and Using Enums in C# 🚀
date: 2024-05-05
modified: 2024-05-05
tags: [.NET, c#, enums, Enumeration]
description: Alternative to c# enums
comments: true
---

In this post, I will try to explaing an alternative way to use instead of `enums`. I already mentioned enums [here](https://denizzengin.com/csharp-enums/). `Enums` can be used in many places in our code base, especially in control flows. It's enough for most cases but some situations can be problematic. Let's say you are using the same enum in many control flows to decide the behaviour of the process. In this case, we scatter the business information throughout the program. At that point, we break down **Open-Closed** principle and make it hard to maintain because if we need a new enum value we should review all these and most likely change them explicitly.

For example, you have a system and there are service points to serve customer as a bank. In order to represent this, we can create an enum `ServicePointType` as below,

{% highlight c# %}
public enum ServicePointType : byte
{
    IndependentAtm = 1,
    BankAtm,
    Bank    
}
{% endhighlight %}

We need to set a max amount for daily withdrawing limit. We can simply achieve that like this,

{% highlight c# %}
public interface IWithdraw
{
    public bool Withdraw(Account account);
}

public class Account
{
    public decimal Balance { get; set; }

    public ServicePointType ServicePointType { get; set; }

    public ServicePointTypeEnumeration? ServicePointTypeEnumeration { get; set; }
}

public class WithdrawService : IWithdraw
{
    public bool Withdraw(Account account)
    {
        if (!CanWithdraw(account))
        {
            return false;
        }

        // todo

        return true;
    }

    private static bool CanWithdraw(Account account)
    {

        // todo validations

        return GetLimit(account.ServicePointType) >= account.Balance;
    }

    private static decimal GetLimit(ServicePointType servicePointType) => servicePointType switch
    {
        ServicePointType.IndependentAtm => 5_000.000m,
        ServicePointType.BankAtm => 50_000.000m,
        ServicePointType.Bank => decimal.MaxValue,
        _ => default
    };
}
{% endhighlight %}

In the above code, we simulate an imaginary scenario that checks limit to withdraw money. Suppose there was a new `ServicePointType`, we would have to add that value into `WithdrawService`. We can achieve the same functionality without a switch statement and more object-oriented way. We can do this with `Enumeration` class which c# provides us.

Let's add a new `Enumeration` class as a base,

{% highlight c# %}
public class Enumeration : IComparable
{
    public int Id { get; private set; }

    public string Name { get; private set; }

    protected Enumeration(int id, string name) => (Id, Name) = (id, name);

    public override string ToString() => Name;

    public static IEnumerable<T> GetAll<T>() where T : Enumeration =>
        typeof(T).GetFields(BindingFlags.Public |
                            BindingFlags.Static |
                            BindingFlags.DeclaredOnly)
                .Select(x => x.GetValue(null))
                .Cast<T>();

    public int CompareTo(object? other)
    {
        ArgumentNullException.ThrowIfNull(other);

        return Id.CompareTo(((Enumeration)other).Id);
    }
}
{% endhighlight %}

We created `Enumeration` class as a base class. It's very similar to typical `enums`. There are `Id` and `Name` fields that represent it. The constructor has a protected keyword because only subclasses can access and use it. We override the `ToString` method to give the same behaviour as `ServicePointType` because when we call `Console.WriteLine(ServicePointType.Bank))` it will write `Bank` text to output. The next method we implemented is `GetAll<T>` to be able to use instead of `Enum.GetValue<TEnum>()`. It uses `System.Reflection` the same as `enum`. The last method is `CompareTo(object? other)` which comes from `IComparable` interface to compare and sort their values. It's same as Microsoft official implementation as you can find [here](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/enumeration-classes-over-enum-types).

Now, we can inherit `Enumeration` class as below,

{% highlight c# %}
public class ServicePointTypeEnumeration : Enumeration
{
    public static readonly ServicePointTypeEnumeration IndependentAtm = new(1, nameof(IndependentAtm), 5_000.000m);

    public static readonly ServicePointTypeEnumeration BankAtm = new(2, nameof(BankAtm), 50_000.000m);

    public static readonly ServicePointTypeEnumeration Bank = new(3, nameof(Bank), decimal.MaxValue);

    protected ServicePointTypeEnumeration(int id, string name, decimal limit) : base(id, name)
    {
        Limit = limit;
    }

    public decimal Limit { get; set; }
}
{% endhighlight %}

In the above, we created three instances of `ServicePointTypeEnumeration` and they are shared across application. We give them related initial values in the place where they were created. We also extended with `Limit` property to access from outside without needing a switch statement.

**Note:** As you recognize, one of the most important points is that we are using `class` type(**reference type**) instead of `enum` which is the **value type**. Therefore, we can enrich our objects as a benefit of object-oriented.

Let's modify our `WithdrawService` class to use it,

{% highlight c# %}
public class WithdrawService : IWithdraw
{
    public bool Withdraw(Account account)
    {
        if (!CanWithdraw(account))
        {
            return false;
        }

        // todo

        return true;
    }

    private static bool CanWithdraw(Account account)
    {

        // todo validations

        return account.ServicePointTypeEnumeration!.Limit >= account.Balance;
    }
}
{% endhighlight %}

The difference from the previous implementation is that we don't need `GetLimit(ServicePointType servicePointType)` method anymore because we already have `limit` value internally set. All we need is to access `Limit` property.

This is not limited just Limit property. There can be any other object-related field you can extend or change behaviour. For example, `enums` doesn't support implicit conversion(you can find more detail [here](https://denizzengin.com/csharp-enums/#also-c-enums-doesnt-support-implicit-conversion-the-below-code-gives-compiling-error)). We can implement it easily. All we need is to add one line of code to `Enumeration` base class as below,

{% highlight c# %}
public static implicit operator int(Enumeration enumeration) => enumeration.Id;
{% endhighlight %}

Now, code is valid as below,
{% highlight c# %}
int bank = ServicePointTypeEnumeration.Bank;
{% endhighlight %}

As a result, enums are pretty enough in most cases but there could be some scenarios we can give solutions from a different perspective via `Enumeration`. If you're unsure about needing it, you should probably go with `enum`.

### Sample Repository

- [c# enums extended](https://github.com/denizzengin/blog-csharp-enums-alternative)


### Resources

- [Use enumeration classes instead of enum types (C# reference)](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/enumeration-classes-over-enum-types)
- [Enumeration classes](https://lostechies.com/jimmybogard/2008/08/12/enumeration-classes/)