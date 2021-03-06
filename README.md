<div align="center">
  <img alt="fairybread" src="logo.svg" height="200px">
  <p>
    Input validation for HotChocolate.
  </p>
  <p>
	  <a href="https://github.com/benmccallum/fairybread/releases"><img alt="GitHub release" src="https://img.shields.io/github/release/benmccallum/fairybread.svg"></a>
	  <a href="https://www.nuget.org/packages/FairyBread"><img alt="Nuget version" src="https://img.shields.io/nuget/v/FairyBread"></a>
	  <a href="https://www.nuget.org/packages/FairyBread"><img alt="NuGet downloads" src="https://img.shields.io/nuget/dt/FairyBread"></a>
  </p>
</div>

## Getting started

Install the package.

```bash
dotnet add package FairyBread
```

Hook up FairyBread in your `Startup.cs`.

```c#
// Add the FluentValidation validators
services.AddValidatorsFromAssemblyContaining<FooInputDtoValidator>();

// Add FairyBread
services.AddFairyBread(options =>
{
    options.AssembliesToScanForValidators = new[] { typeof(FooInputDtoValidator).Assembly };
});

// Configure FairyBread middleware using HotChocolate's ISchemaBuilder
HotChocolate.SchemaBuilder.New()
    ...
    .UseFairyBread()
    .Create()
    .MakeExecutable(options =>
    {
        options
           .UseDefaultPipeline()
           // Note: If you've already got your own IErrorFilter
           // in the pipeline, you should have it call this one
           // as part of its error handling, to rewrite the
           // validation error
           .AddErrorFilter<DefaultValidationErrorFilter>();
    });
```

Set up [FluentValidation](https://github.com/FluentValidation/FluentValidation) validators like you usually would on
the CLR types backing your HotChocolate input types.

```c#

public class UserInput { ... }

public class UserInputType : InputObjectType<UserInput> { ... }

public class UserInputValidator : AbstractValidator<UserInput> { ... }

// An example GraphQL field in HotChocolate
public Task CreateUser(UserInput userInput) { ... }
```

### When validation will fire

By default, FairyBread will validate any argument that:
* is an `InputObjectType` on a mutation operation,
* is manually opted-in at the field level with:
    * `[Validate]` on the resolver method argument in pure code first
    * `.UseValidation()` on the argument definition in code first
* is manually opted-in at the input type level with:
    * `[Validate]` on the CLR backing type in pure code first
    * `.UseValidation()` on the InputObjectType descriptor in code first
* it's CLR type or `InputObjectType` are marked with `[Validate]` (i.e. opt-in this type of argument globally).

> Note: by default, query operations are notably excluded. This is mostly for performance reasons.

If the default doesn't suit you, you can always change it by configuring `IFairyBreadOptions.ShouldValidate`. See [Customization](#Customization).
Some of the default implementations are defined publicly on the default options class so they can be re-used and composed to form your own implementation.

### How validation errors will be handled

Errors will be written out into the GraphQL execution result in the `Errors` property. By default, 
extra information about the validation will be written out in the Extensions property. 

For example:

> (note, this is the serialized `IExecutionResult`, not a GraphQL server response)

```
{
  Data: ...,
  Errors: [
    {
      Message: 'Validation errors occurred.',
      Code: 'FairyBread_ValidationError',
      Path: [
        'write'
      ],
      Locations: [
        {
          Line: 1,
          Column: 12
        }
      ],
      Extensions: {
        code: 'FairyBread_ValidationError',
        Failures: [
          {
            ErrorCode: 'EqualValidator',
            ErrorMessage: '\'Some Integer\' must be equal to \'1\'.',
            PropertyName: 'SomeInteger',
            ResourceName: 'EqualValidator',
            AttemptedValue: -1,
            FormattedMessagePlaceholderValues: {
              ComparisonValue: 1,
              PropertyName: 'Some Integer',
              PropertyValue: -1
            }
          },
          ...
        ]
      }
    }
  ]
}
```

### Dealing with multi-threaded execution issues

GraphQL resolvers are inherently multi-threaded; as such, you can run into issues injecting things like an EntityFramework `DbContext` into a field resolver (or something it uses) which doesn't allow muli-thread usage. One solution for this is to resolve `DbContext` into its own "scope", rather than the default scope (for an ASP.NET Core application is tied to the HTTP Request).

With FairyBread, you might need to do this if one of your validators uses a `DbContext` (say to check if a username already exists on a create user mutation). Good news is, it's as easy as marking your validator with `IRequiresOwnScopeValidator` and we'll take care of the rest.

```c# 
public class UserInputValidator : AbstractValidator<UserInput>, IRequiresOwnScopeValidator
{
    public UserInputValidator(SomeDbContext db) { ... } // db will be a unique instance for this validation operation
}
```

### Where to next?

For more examples, please see the tests.

## Customization

FairyBread was built with customization in mind. At configuration time, you can tweak the default settings as needed:

```c#
services.AddFairyBread(options =>
{
    options.AssembliesToScanForValidators = new[] { typeof(MyValidator).Assembly };
    options.ShouldValidate = (ctx, arg) => ...;
    options.ThrowIfNoValidatorsFound = true/false;
});
```

But it goes further than that. You can completely swap in your own options, validator provider, 
validator result handler and so on to get the functionality you need by simply adding your own 
implementation of the relevant interface before adding FairyBread, for example:

```c#
services.Add<IValidationResultHandler, CustomValidationResultHandler>();
```

If you want to just change one element of a default implementation, they aren't `sealed` and 
their methods are `virtual` so have at it. For instance, the `CustomValidationResultHandler` above might want to
modify the way error codes are set on the `IError`, which can be done like so:

```c#
public class CustomValidationResultHandler : DefaultValidationResultHandler
{
    protected override IErrorBuilder ExtractError(IMiddlewareContext context, ValidationFailure failure)
        => base.ExtractError(context, failure)
            .SetExtension(nameof(failure.ErrorCode), $"CustomPrefix:{failure.ErrorCode}");
}
```

Check out <a href="src/FairyBread.Tests/CustomizationTests.cs">CustomizationTests.cs</a> for complete examples.

## Backlog

See issues.

## What the heck is a fairy bread?

A (bizarre) Australian food served at children's parties. Since I'm Australian and HotChocolate has a lot of 
project names with sweet tendencies, this seemed like a fun name for the project.

According to [wikipedia](https://en.wikipedia.org/wiki/Fairy_bread):
> "Fairy bread is sliced white bread spread with butter or margarine and covered with sprinkles or "hundreds and thousands", served at children's parties in Australia and New Zealand. It is typically cut into triangles."
