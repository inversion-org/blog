# Conditional love

- Adam Christie ([@fractos](https://github.com/fractos))

I've been working on making the definition of a service container using [`Inversion.Naid`](https://github.com/guy-murphy/inversion-dev/tree/master/Inversion.Naiad) (https://github.com/guy-murphy/inversion-dev/tree/master/Inversion.Naiad) behaviours more tidy, more obvious and safer from typos - a common problem area when dealing with service containers that are keyed on strings and use them throughout for conditional logic.

## The problem with strings...

The [`PrototypedBehaviour`](https://github.com/guy-murphy/inversion-dev/blob/master/Inversion.Process/Behaviour/PrototypedBehaviour.cs) (https://github.com/guy-murphy/inversion-dev/blob/master/Inversion.Process/Behaviour/PrototypedBehaviour.cs) base class is absurdly powerful. With it, you can define a mixed bag of conditions and configuration - everything that the behaviour needs to determine whether its Condition should trigger its Action, and then everything that an Action needs to fulfil its functionality, all in one place.

Driving the behaviour from what is essentially a bag of dictionaries is incredibly flexible, but comes with some risk; notably, that the various stanzas could easily contain a typo which could radically change the meaning of the configuration.

As well as changing behaviour configuration to use statically typed items where possible (for Flags, store names, Settings etc), I decided to make mini-factories for Conditions. I believe the result aids readability and also makes it less likely to make a mistake.

## Improving the behaviour definition

So, with some expansion and a little helper class to sort out the Conditions, this turns the following:

```c#
new LoadCustomerBehaviour("load-data",
    new Configuration.Builder
    {
        {"config", "param", "customer" },
        {"config", "output-key", "customer" },
        {"context", "type", "customer", "System.Int32" }
    }),

new LoadCustomerByNameBehaviour("load-data",
    new Configuration.Builder
    {
        {"config", "param", "customer" },
        {"config", "output-key", "customer" },
        {"control-state", "excludes", "customer" },
        {"context", "has", "customer"}
    }),
    ...
```

into this:

```c#
new LoadCustomerBehaviour(
    respondsTo: Events.Lifecycle.LoadData,
    config: new Configuration.Builder
    {
        Conditions.ContextTyped(name: "customer", type: typeof(System.Int32)),
        {"config", "param", "customer" },
        {"config", "output-key", "customer" }
    }),

new LoadCustomerByNameBehaviour(
    respondsTo: Events.Lifecycle.LoadData,
    config: new Configuration.Builder
    {
        Conditions.ControlStateExcludes(name: "customer"),
        Conditions.ContextHas(name: "customer"),
        {"config", "param", "customer" },
        {"config", "output-key", "customer" }
    }),
    ...
```

And that reads a lot better to me, especially when using named parameters explicitly where possible. Guy Murphy once described how much he liked the named parameters in C#, saying that they "look like a little DSL", which is definitely part of the charm for me too, aside from the obvious advantages for disambiguation and reducing mistakes.

## Conditions helper

The Conditions helper class itself is simply a generator of `Inversion.Process.Configuration.Element` instances, which effectively marshal the `NamedCases` found in [`Inversion.Process.Behaviour.Prototype`](https://github.com/guy-murphy/inversion-dev/blob/master/Inversion.Process/Behaviour/Prototype.cs) (https://github.com/guy-murphy/inversion-dev/blob/master/Inversion.Process/Behaviour/Prototype.cs).

```c#
public static class Conditions
    {
      ...
        public static IConfigurationElement ControlStateExcludes(string name)
        {
            return new Inversion.Process.Configuration.Element(
                ordinal: 0,
                frame: "control-state",
                slot: "excludes",
                name: name,
                value: String.Empty
            );
        }

        public static IConfigurationElement ContextHas(string name)
        {
            return new Inversion.Process.Configuration.Element(
                ordinal: 0,
                frame: "context",
                slot: "has",
                name: name,
                value: String.Empty
            );
        }

        public static IConfigurationElement ContextExcludes(string name)
        {
            return new Inversion.Process.Configuration.Element(
                ordinal: 0,
                frame: "context",
                slot: "excludes",
                name: name,
                value: String.Empty
            );
        }

        public static IConfigurationElement ContextTyped(string name, Type type)
        {
            return new Inversion.Process.Configuration.Element(
                ordinal: 0,
                frame: "context",
                slot: "type",
                name: name,
                value: type.FullName);
        }
        ...
```

By explicitly generating the conditional stanzas using helper functions, I believe it becomes much more obvious about the intention of the behaviour configuration, and much less prone to simple typos.

The next step for making the configuration water-tight is to start using statically-typed names for Control State and Context parameter names, which I'll experiment with shortly, to see if they are a good fit.
