# CdCSharp.BuildTools

A flexible and extensible build automation framework for .NET projects that enables asset generation, template management, and Node.js tooling integration.

## 🚀 Features

- **Asset Generation System**: Automatically discover and generate CSS, JavaScript, and other assets through a plugin-based architecture
- **Template Management**: Deploy build configuration files (package.json, vite.config.js, etc.) from code
- **Node.js Integration**: Seamlessly integrate npm, npx, and Vite into your .NET build pipeline
- **Order Control**: Manage asset generation order with the `Order` property
- **Convention-based Discovery**: Automatic discovery of generators through attributes
- **Cross-platform**: Works on Windows, Linux, and macOS

## 📦 Installation

### As a NuGet Package

```bash
dotnet add package CdCSharp.BuildTools
```

### As a .NET Tool

```bash
dotnet tool install --global CdCSharp.BuildTools
```

## 🎯 Quick Start

### 1. Create an Asset Generator

```csharp
using CdCSharp.BuildTools;
using CdCSharp.BuildTools.Attributes;

[AssetGenerator(Order = 1)]
public class MyCssGenerator : IAssetGenerator
{
    public string FileName => "styles.css";
    public string Name => "My CSS Generator";

    public async Task<string> GetContent()
    {
        return """
        /* Auto-generated CSS */
        :root {
            --primary-color: #007bff;
        }
        """;
    }
}
```

### 2. Define Build Templates

```csharp
using CdCSharp.BuildTools.Attributes;

public class MyTemplates
{
    [BuildTemplate("package.json")]
    public static string GetPackageJson() => """
    {
      "name": "my-project",
      "version": "1.0.0",
      "devDependencies": {
        "vite": "latest"
      }
    }
    """;

    [BuildTemplate("vite.config.js", Overwrite = true)]
    public static string GetViteConfig() => """
    import { defineConfig } from 'vite';
    export default defineConfig({
        // Your Vite configuration
    });
    """;
}
```

### 3. Configure Build Pipeline

```csharp
using CdCSharp.BuildTools;

// In your build project's Program.cs or MSBuild task
string projectPath = args.Length > 0 ? args[0] : ".";
await BuildToolsManager.Build(projectPath);
```

### 4. Customize Paths (Optional)

```csharp
using CdCSharp.BuildTools.Attributes;

[assembly: BuildToolsConfiguration(
    CssBundlePath = "Assets/Css",
    TypesPath = "ClientApp/Types",
    WwwRootCssPath = "wwwroot/styles",
    WwwRootJsPath = "wwwroot/scripts"
)]
```

## 🏗️ Architecture

### Core Components

#### BuildContext
Manages project paths and directory structure:

```csharp
var context = new BuildContext("/path/to/project");
string cssPath = context.CssBundlePath;  // ProjectPath/CssBundle
string outputCss = context.OutputCssPath;  // ProjectPath/wwwroot/css
```

#### BuildPipeline
Orchestrates the build process:

```csharp
var pipeline = new BuildPipeline(context);
await pipeline.ExecuteAsync();
// 1. Initializes Node.js tools
// 2. Generates assets
// 3. Builds CSS and JS with Vite
```

#### NodeToolsManager
Manages Node.js tooling:

```csharp
var nodeTools = new NodeToolsManager(context);
await nodeTools.VerifyNodeInstalledAsync();
await nodeTools.EnsurePackagesInstalledAsync();
await nodeTools.BuildCssAsync();
await nodeTools.BuildJsAsync();
```

### Discovery System

The framework automatically discovers:

- **Asset Generators**: Classes implementing `IAssetGenerator` with `[AssetGenerator]` attribute
- **Build Templates**: Static methods with `[BuildTemplate]` attribute

```csharp
// Discovery happens automatically
IReadOnlyList<IAssetGenerator> generators = AssetGeneratorDiscovery.Discover();
```

## 📝 Attributes Reference

### AssetGeneratorAttribute

Marks a class as an asset generator:

```csharp
[AssetGenerator(Order = 10)]
public class MyGenerator : IAssetGenerator { }
```

**Properties:**
- `Order` (int): Execution order (default: 0). Lower numbers execute first.

### BuildTemplateAttribute

Marks a method as a build template provider:

```csharp
[BuildTemplate("config.json", Overwrite = true)]
public static string GetConfig() => "{ }";
```

**Properties:**
- `RelativePath` (string, required): File path relative to project root
- `Overwrite` (bool): Whether to overwrite existing files (default: false)

### BuildToolsConfigurationAttribute

Configures default paths at assembly level:

```csharp
[assembly: BuildToolsConfiguration(
    CssBundlePath = "CustomCss",
    TypesPath = "CustomTypes",
    WwwRootCssPath = "wwwroot/custom-css",
    WwwRootJsPath = "wwwroot/custom-js"
)]
```

## 🔧 Advanced Usage

### Custom Generator Order

Control the execution order of generators:

```csharp
[AssetGenerator(Order = 1)]  // Runs first
public class ResetCssGenerator : IAssetGenerator { }

[AssetGenerator(Order = 5)]  // Runs after Reset
public class ThemeGenerator : IAssetGenerator { }

[AssetGenerator(Order = 10)] // Runs last
public class ComponentGenerator : IAssetGenerator { }
```

### Dynamic Content Generation

Generate content based on runtime conditions:

```csharp
[AssetGenerator]
public class DynamicGenerator : IAssetGenerator
{
    public string FileName => "_dynamic.css";
    public string Name => "Dynamic CSS";

    public async Task<string> GetContent()
    {
        var colors = await FetchColorsFromApi();
        var sb = new StringBuilder();
        
        sb.AppendLine(":root {");
        foreach (var color in colors)
        {
            sb.AppendLine($"  --{color.Name}: {color.Value};");
        }
        sb.AppendLine("}");
        
        return sb.ToString();
    }
}
```

### Template Composition

Create reusable template components:

```csharp
public class BuildTemplates
{
    private static string GetBaseViteConfig() => """
        import { defineConfig } from 'vite';
        """;

    [BuildTemplate("vite.config.js")]
    public static string GetViteConfig() =>
        GetBaseViteConfig() + """
        export default defineConfig({
            build: {
                outDir: 'wwwroot'
            }
        });
        """;
}
```

## 🎨 Real-World Example

See the complete implementation in [CdCSharp.BlazorUI.BuildTools](examples/BlazorUI.BuildTools) which uses this framework to:

- Generate CSS for component families (Input, Picker, etc.)
- Manage design tokens and typography
- Build theme systems
- Bundle TypeScript and CSS with Vite

## 🛠️ Integration with MSBuild

Create a build tool project:

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>net10.0</TargetFramework>
    </PropertyGroup>

    <ItemGroup>
        <PackageReference Include="CdCSharp.BuildTools" Version="*" />
    </ItemGroup>
</Project>
```

Reference it in your main project:

```xml
<Target Name="BuildAssets" BeforeTargets="BeforeBuild">
    <Exec Command="dotnet run --project ../MyProject.BuildTools/MyProject.BuildTools.csproj $(ProjectDir)" />
</Target>
```

## 📋 Requirements

- **.NET 10.0** or higher
- **Node.js** (for Vite integration) - automatically verified during build

## 🤝 Contributing

Contributions are welcome! This framework is designed to be extended:

1. Implement `IAssetGenerator` for new asset types
2. Add `[BuildTemplate]` methods for new configuration files
3. Extend `NodeToolsManager` for additional Node.js tools

## 📄 License

MIT License - see LICENSE file for details

## 🔗 Related Projects

- **CdCSharp.BlazorUI.BuildTools**: Reference implementation for Blazor component libraries
- **CdCSharp.Pangea**: Full-stack application framework using this build system

## 📚 Documentation

For more examples and detailed guides, visit the [documentation](docs/README.md).

---

**Made with ❤️ for the .NET community**