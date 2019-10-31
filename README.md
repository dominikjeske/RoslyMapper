# What is RoslyMapper
A convention-based object-object mapper in .NET based on Roslyn

# Why

This tool is inspired by Automapper - its advantages but above all disadvantages that could cost time and money. We can find articles like those below that describes disadvantages and common mistakes of using autommaper:

- https://cezarypiatek.github.io/post/why-i-dont-use-automapper/
- https://jimmybogard.com/automappers-design-philosophy/

To summarize we have:

**Disadvantages:**

- Static analyzing  is not working and properties looks like they are not in use
- We cannot find all references of properties
- A convention-based generator is working in runtime so all errors are visible after applications starts and sometimes we don't detect bug until it gets to production
- Generated code is kind of black magic - it is working or not. We find out in production ;) 
- "With great power comes great responsibility" - tool have many features and it is tempting for people to overuse so business logic is leaking from domain into infrastructure
- Performance  of runtime generated code will never be as fast as code compiled during build by c# compiler
- Debugging mapping code is impossible

**Advantages:**

With all that evil we can ask why people are using Automapper.

- We have many models with similar properties in different layers to it is very tedious to write them by hand
- Even when we take a time to write manual mapper AutoMapper give us guard so when some property is added to model we can detect this (unfortunately only in runtime)
- AutoMapper have many build in features like LINQ support that can be really time saver
- Last but not least - people are lazy and have tendency to use new toys or reuse toys that are used by others


# Philosophy

A main philosophy of RoslyMapper is to create simple mapping experience that is created in build time and prevents from common misusings.

# How

Lets take two classes that we would like to map

```csharp
public class Src
{
    public string Name { get; set; }
    public int Age { get; set; }
    public int IgnoredDestination { get; set; }
    public int DefaultValuedProperty { get; set; }
    public int CustomValue { get; set; }
}

public class Dst
{
    public string Name { get; set; }
    public int Age { get; set; }
    public int IgnoredDestination { get; set; }
    public int DefaultValuedProperty { get; set; }
    public int CustomValue { get; set; }
}
```

When we want make some simple mapping of objects with same property name we can write adapter like this

```csharp
[Adapter]
public partial class MyAdapter : IMapper<Src, Dst>
{
}
```

this code by the power given by Roslyn and [CodeGeneration.Roslyn](https://github.com/AArnott/CodeGeneration.Roslyn) will generate another part of this class that will be looking like below. 

```csharp
public partial class MyAdapter : BaseAdapter<Src, Dst>
{
    public Dst Map(Src source)
    {
        var destination = new Dst();
        destination.Name = source.Name;
        destination.Age = source.Age;
        destination.DefaultValuedProperty = source.DefaultValuedProperty;
        destination.CustomValue = source.CustomValue;
        destination.IgnoredDestination = source.IgnoredDestination;
        return destination;
    }
}
```

We can of course have more complex mapping so we can use some configuration for this

```csharp
public class Src
{
    public string _name { get; set; }
    public int _age { get; set; }
    public int _ignoredSource { get; set; }
    public int? _defaultValuedProperty { get; set; }
}

[Adapter]
public partial class MyAdapter : IMapper<Src, Dst>
{
    protected override void Configure(IConfigureMapping<Src, Dst> config)
    {
        config.ForMember(x => x.IgnoredDestination).Ignore();             // Ignore destination mapping
        config.ForMember(x => x.DefaultValuedProperty).WithDefault(7);    // Use default value when null
        config.ForMember(x => x.CustomValue).WithValue(s => s._age + 1);  // Use some source filed with simple calculation
        config.IgnoreSourceMember(x => x._ignoredSource);                 // Ignore source member
        config.WithSourceResolver<UnderscoreResolver>();                  // Inform tha we have to skip underscore
        config.WithPropertiesComparer<IgnoreCaseComparer>();              // We do not take case into account while comparing
    }
}
```

Above mapping will generate this code

```csharp
public partial class MyAdapter : BaseAdapter<Src, Dst>
{
    public Dst Map(Src source)
    {
        var destination = new Dst();
        destination.Name = source._name;
        destination.Age = source._age;
        destination.DefaultValuedProperty = source._defaultValuedProperty ?? 7;
        destination.CustomValue = source._age + 1;
        return destination;
    }
}
```

Most important things of this process are

- Code is generated inside obj directory and is accessible instantly (without manual compilation) so when we invoke F12 we can choose to navigate to this file and of course during debugging or static analyzing the reference is visible by the tools
- Code is generated inside obj directory and is accesibly instanlty so when we invoke F12 we can choose to navigate to this file and of course during debugging or static analysing the reference is visible by the tools
- There is no external tool or IDE dependency
- There is no reflection or expression tree's - just pure c# code
- All error will be visible in Error window just like other compiler errors
- There is no magic places - we see all the code that will be used
- When some property will be added in future we will fail but in build time
- Most use case of this tool promote simple cases - if you want to go to database to map some property it is probably time to write code manually
- Performance  should be the same as other manually created (only build time can be slightly longer)
  
# Future

If this tool will have audience I'm thinking about

- Public source code
- Creating nuget
- Add support for creating objects using constructor
- Add support for creating objects using its static Create method
- ???
