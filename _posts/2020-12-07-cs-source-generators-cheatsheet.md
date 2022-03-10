---
layout: post
title: C# Source Generators Cheatsheet
date: 2020-12-07 13:52:49
tags:
  - C#
  - Source Generators

excerpt_separator: <!--end_excerpt-->
---

Project files and code snippets for C# source generators.

<!--end_excerpt-->

- [Csproj Files](#csproj-files)
    - [**`Application.csproj`**](#applicationcsproj)
    - [**`CodeGeneration.csproj`**](#codegenerationcsproj)
    - [**`CodeGeneration.Test.csproj`**](#codegenerationtestcsproj)
- [Testing](#testing)
  - [Adding Additional Texts](#adding-additional-texts)
    - [**`CustomAdditionalText.cs`**](#customadditionaltextcs)
  - [Test Code](#test-code)
    - [**`Program.cs`**](#programcs)
  - [Adding Attributes for Customization](#adding-attributes-for-customization)
    - [**`AttributeExtensions.cs`**](#attributeextensionscs)
- [Issues](#issues)

## Csproj Files

This structure is for code generation within the solution.

#### **`Application.csproj`**

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    ...
    <!-- Use following lines to write the generated files to disk. -->
    <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
    <CompilerGeneratedFilesOutputPath>$(BaseIntermediateOutputPath)\GeneratedFiles</CompilerGeneratedFilesOutputPath>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\CodeGeneration\CodeGeneration.csproj"
                      OutputItemType="Analyzer"
                      ReferenceOutputAssembly="false" />
  </ItemGroup>

  <ItemGroup>
    <!-- Use following lines to add additional files to source generation. -->
    <AdditionalFiles Include="SomeFile" />
  </ItemGroup>
</Project>
```

#### **`CodeGeneration.csproj`**

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.2">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp.Workspaces" Version="3.8.0" />
  </ItemGroup>

</Project>
```

#### **`CodeGeneration.Test.csproj`**

This is a simple command line project, but can be converted to a unit test project easily.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp3.1</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\CodeGeneration\CodeGeneration.csproj" />
  </ItemGroup>
</Project>
```

## Testing

### Adding Additional Texts

You should extend the `AdditionalText` class to be able to add additional files to compilation.

#### **`CustomAdditionalText.cs`**

```c#
using System.IO;
using System.Threading;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.Text;

public class CustomAdditionalText : AdditionalText
{
    private readonly string _text;

    public override string Path { get; }

    public CustomAdditionalText(string path)
    {
        Path = path;
        _text = File.ReadAllText(path);
    }

    public override SourceText GetText(CancellationToken cancellationToken = new CancellationToken())
    {
        return SourceText.From(_text);
    }
}
```

### Test Code

Create a compilation with given syntax trees and additional texts. Use `CSharpGeneratorDriver` class to run your generators.

#### **`Program.cs`**

```c#
using System;
using System.Collections.Generic;
using System.Collections.Immutable;
using System.Linq;
using System.Reflection;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;

public class Program
{
    private static void GenerateSource(IEnumerable<string> sources, IEnumerable<string> additionalTextPaths)
    {
        List<MetadataReference> references = new List<MetadataReference>();
        Assembly[] assemblies = AppDomain.CurrentDomain.GetAssemblies();

        foreach (Assembly assembly in assemblies)
        {
            if (!assembly.IsDynamic)
            {
                references.Add(MetadataReference.CreateFromFile(assembly.Location));
            }
        }

        List<SyntaxTree> syntaxTrees = new List<SyntaxTree>();
        List<AdditionalText> additionalTexts = new List<AdditionalText>();

        foreach (string source in sources)
        {
            SyntaxTree syntaxTree = CSharpSyntaxTree.ParseText(source);
            syntaxTrees.Add(syntaxTree);
        }

        foreach (string additionalTextPath in additionalTextPaths)
        {
            AdditionalText additionalText = new CustomAdditionalText(additionalTextPath);
            additionalTexts.Add(additionalText);
        }

        CSharpCompilation compilation = CSharpCompilation.Create(
            "original",
            syntaxTrees,
            references,
            new CSharpCompilationOptions(OutputKind.DynamicallyLinkedLibrary));

        GeneratorDriver driver = CSharpGeneratorDriver
            .Create(new YourSourceGenerator())
            .AddAdditionalTexts(ImmutableArray.CreateRange(additionalTexts));
        
        driver.RunGeneratorsAndUpdateCompilation(
            compilation,
            out Compilation outputCompilation,
            out ImmutableArray<Diagnostic> diagnostics);

        bool hasError = false;

        foreach (Diagnostic diagnostic in diagnostics.Where(x => x.Severity == DiagnosticSeverity.Error))
        {
            hasError = true;
            Console.WriteLine(diagnostic.GetMessage());
        }

        if(!hasError)
        {
            Console.WriteLine(string.Join("\r\n", outputCompilation.SyntaxTrees));
        }
    }
}
```

### Adding Attributes for Customization

You can create attributes that can be used to customize the source generation process by the client project. This is not supported as a feature by Roslyn, for now. (See [this](https://github.com/dotnet/roslyn/issues/49753) issue.) But, there is a workaround.

- Add your attribute as a generated source file.
- Create a new compilation with the new syntax tree.
- Use the `IAssemblySymbol` from the resulting compilation to resolve your attribute. (You can customize `GetAttributeProperty` method.)

#### **`AttributeExtensions.cs`**

```c#
using System;
using System.Collections.Immutable;
using System.Linq;
using System.Reflection;
using System.Text;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.Text;

public static class AttributeExtensions
{
    public static IAssemblySymbol AddAttribute(
        this GeneratorExecutionContext context,
        string attributeName,
        string defaultClassName)
    {
        {% raw %}string attribute = $@"
namespace YourNamespace
{{
    [System.AttributeUsage(System.AttributeTargets.Assembly)]
    internal class {attributeName} : System.Attribute
    {{
        public string SomeValue {{ get; }}

        public {attributeName}(string someValue)
        {{
            SomeValue = someValue;
        }}
    }}
}}
";{% endraw %}

        SourceText sourceText = SourceText.From(attribute.Trim(), Encoding.UTF8);
        context.AddSource(attributeName, sourceText);

        // Create a new compilation with the new attribute source text. The assembly of
        // that compilation will be able to resolve the references to the attribute correctly.
        CSharpParseOptions? options = (context.Compilation as CSharpCompilation)
            ?.SyntaxTrees[0]
            .Options as CSharpParseOptions;

        Compilation compilation = context.Compilation
            .AddSyntaxTrees(CSharpSyntaxTree.ParseText(sourceText, options));

        // Use this assembly to get the attribute arguments.
        return compilation.Assembly;
    }

    public static string? GetAttributeProperty(
        this IAssemblySymbol assembly,
        string attributeName)
    {
        AttributeData? attributeData = assembly
            .GetAttributes()
            .FirstOrDefault(x => x.AttributeClass?.ToString() == $"YourNamespace.{attributeName}");

        if (attributeData == null)
        {
            return null;
        }

        ImmutableArray<TypedConstant> attributeArguments = attributeData.ConstructorArguments;

        return attributeArguments.FirstOrDefault().Value as string;
    }
}
```

## Issues

WPF projects do not support code generation for now. (See [this](https://github.com/dotnet/wpf/issues/3404) issue.)
Workaround is to extract necessary parts to a class library and run code generation there. (Which is mostly not possible in practice.)
This is possibly true for Blazor projects, too.
