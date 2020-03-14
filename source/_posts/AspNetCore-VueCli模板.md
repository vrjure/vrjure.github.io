---
title: AspNetCore+VueCli模板
tags:
  - AspNetCore
  - Vue
categories: 笔记
copyright: true
reward: true
date: 2020-02-28 22:46:06
thumbnail:
---


## 摘要

AspNetCore一直以来都有前端框架的Spa开发模板。AspNetCore 2.0时，Vue，React，Angular这三大前端框架都有相应的Spa开发模板。但是到了AspNetCore 3.0之后，微软改变了实现方式，理由是这三大框架各自有自己的Cli，原来的实现方式已经没有必要了。于是三个Spa开发模板就剩了两个(React,Angular)。然而剩下的Vue正是我需要的，但是它没有...

## Angular和React的集成

前阵子尝试从Github上了解React和Angular上的集成实现。发现在启动AspNetCore服务的同时运行它们的Cli,并且AspNetCore会将请求代理到前端的开发服务器。这么看来的话，Vue的集成开发模板应该也可以这样实现。毕竟Vue也有自己的Cli。无奈对webpack的配置一直云里雾里的。幸运的是，今天无意中发现了一篇博客 [ASP.NET Core Vue CLI Templates]。正好讲的是VueCli和AspNetCore 3.0的Spa开发模板的实现。于是乎尝试了一番。

## [ASP.NET Core Vue CLI Templates]

博客中有提到三种vue和AspNetCore的搭配的方法。

1. 前后端分别独立的开发方式
  适合多个人的团队，分别开发前端和后端

2. AspNetCore官方模板中的方法
  将vue开发服务器作为AspNetCore的一个子进程运行，并让AspNetCore作为代理服务器。

3. 将AspNetCore官方模板中的方法反转一下
  将vue开发服务器作代理使用，AspNetCore服务器不作为代理服务器

## 模板搭建

这里尝试的是第三种方法，使用vue开发服务器作代理。

### 环境

自己的测试环境:

- AspNetCore 3.1.2
- visual code
- @vue/cli 4.2.2
- vue包管理工具yarn
- 在webapi模板上更改

[ASP.NET Core Vue CLI Templates] 中的环境

- AspNetCore 3.0
- visual code/visual studio
- vue包管理工具yarn
- 在自带的react spa模板上更改

这里使用的webapi模板搭建的,包管理工具也改成了yarn。和博客中的介绍大同小异。区别是:我觉得,有助于理解👌。

### 创建webapi模板和vue模板

#### 创建webapi模板

创建模板并添加SpaServices扩展

```cs
dotnet new webapi -o AspNetCoreVueCliTemplate
cd AspNetCoreVueCliTemplate
dotnet add package cd Microsoft.AspNetCore.SpaServices.Extensions --version 3.1.2
```

#### 创建vue模板

在项目目录下创建vue模板

```cs
vue create clientapp
```

vue创建过程中,按照提示一步步来，包管理工具选择yarn,使用npm也没有关系

#### 添加中间件VueDevelopmentServerMiddleware

##### 从[GitHub](https://github.com/DaniJG/ASPCoreVueCLITemplates/tree/master/alternative-spa-template)将Middleware中的内容全部搬运到自己项目下(包括文件夹)。完成后项目结构如下：

  ![middleware](https://u62pgq.bn.files.1drv.com/y4m9xxeXTdO0Y2y9rzTERsEMgQjL621_95Qls6cpi_k-GrxattPp-WDtJ0-Gpph8K_VZNg9hVzMEBeH6mRX_RF2jJOyc9WOe12O4xCR3L2mxxt4qe5UJIW189YluXf2q_YcpLp5eNqVoyjZOzxn53HIuy1eXWCwXh79GUQGfp3b9swU-1EQRBibKAFL3Xj8r2yyKKggc1t_byQVtNNOHWufJg)

   YarnScriptRunner是NpmScriptRunner改的

##### 修改项目使用的包管理工具配置npm->yarn

*使用npm的跳过。*

###### 将NpmScriptRunner.cs中有关使用npm执行的命令全部换成yarn  

```cs
public YarnScriptRunner(string workingDirectory, string scriptName, string arguments, IDictionary<string, string> envVars)
{
    if (string.IsNullOrEmpty(workingDirectory))
    {
        throw new ArgumentException("Cannot be null or empty.", nameof(workingDirectory));
    }

    if (string.IsNullOrEmpty(scriptName))
    {
        throw new ArgumentException("Cannot be null or empty.", nameof(scriptName));
    }

    var yarnExe = "yarn";
    var completeArguments = $"run {scriptName} {arguments ?? string.Empty}";
    if (RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
    {
        // On Windows, the yarn executable is a .cmd file, so it can't be executed
        // directly (except with UseShellExecute=true, but that's no good, because
        // it prevents capturing stdio). So we need to invoke it via "cmd /c".
        yarnExe = "cmd";
        completeArguments = $"/c yarn {completeArguments}";
    }

    var processStartInfo = new ProcessStartInfo(yarnExe)
    {
        Arguments = completeArguments,
        UseShellExecute = false,
        RedirectStandardInput = true,
        RedirectStandardOutput = true,
        RedirectStandardError = true,
        WorkingDirectory = workingDirectory
    };

    if (envVars != null)
    {
        foreach (var keyValuePair in envVars)
        {
            processStartInfo.Environment[keyValuePair.Key] = keyValuePair.Value;
        }
    }

    var process = LaunchNodeProcess(processStartInfo);
    StdOut = new EventedStreamReader(process.StandardOutput);
    StdErr = new EventedStreamReader(process.StandardError);
}
```

###### 修改AspNetCoreVueCliTemplate.csproj中的配置

主要添加初始环境配置,修改后如下:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>netcoreapp3.1</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.SpaServices.Extensions" Version="3.1.2" />
  </ItemGroup>

  <ItemGroup>
    <!-- Don't publish the SPA source files, but do show them in the project files list -->
    <Content Remove="$(SpaRoot)**" />
    <None Remove="$(SpaRoot)**" />
    <None Include="$(SpaRoot)**" Exclude="$(SpaRoot)node_modules\**" />
  </ItemGroup>

  <ItemGroup>
    <Content Include="clientapp\yarn.lock" />
    <Content Include="clientapp\package.json" />
  </ItemGroup>

  <Target Name="DebugEnsureNodeEnv" BeforeTargets="Build" Condition=" '$(Configuration)' == 'Debug' And !Exists('$(SpaRoot)node_modules') ">
    <!-- Ensure Node.js is installed -->
    <Exec Command="node --version" ContinueOnError="true">
      <Output TaskParameter="ExitCode" PropertyName="ErrorCode" />
    </Exec>
    <Error Condition="'$(ErrorCode)' != '0'" Text="Node.js is required to build and run this project. To continue, please install Node.js from https://nodejs.org/, and then restart your command prompt or IDE." />
    <Message Importance="high" Text="Restoring dependencies using 'yarn'. This may take several minutes..." />
    <Exec WorkingDirectory="$(SpaRoot)" Command="yarn install" />
  </Target>

  <Target Name="PublishRunWebpack" AfterTargets="ComputeFilesToPublish">
    <!-- As part of publishing, ensure the JS resources are freshly built in production mode -->
    <Exec WorkingDirectory="$(SpaRoot)" Command="yarn install" />
    <Exec WorkingDirectory="$(SpaRoot)" Command="yarn run build" />

    <!-- Include the newly-built files in the publish output -->
    <ItemGroup>
      <DistFiles Include="$(SpaRoot)dist\**" />
      <ResolvedFileToPublish Include="@(DistFiles->'%(FullPath)')" Exclude="@(ResolvedFileToPublish)">
        <RelativePath>%(DistFiles.Identity)</RelativePath>
        <CopyToPublishDirectory>PreserveNewest</CopyToPublishDirectory>
      </ResolvedFileToPublish>
    </ItemGroup>
  </Target>
</Project>
```

###### 还有就是Startup.cs

主要是添加修改vue的中间件、静态文件处理程序、以及路由。修改后如下:

```cs
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    // This method gets called by the runtime. Use this method to add services to the container.
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        services.AddSpaStaticFiles(configuration => {
            configuration.RootPath = "clientapp/dist";
        });
    }

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }
        else
        {
            app.UseHsts();
        }

        app.UseHttpsRedirection();

        app.UseStaticFiles();

        app.UseSpaStaticFiles();

        app.UseRouting();

        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllerRoute(name:"default", pattern: "{controller}/{action=Index}/{id?}");
        });

        if (env.IsDevelopment())
        {
            app.UseVueDevelopmentServer();
        }
    }
}
```

#### 配置webpack,为vue服务器设置代理
  
```ts
module.exports = {
  configureWebpack: {
    // The URL where the .Net Core app will be listening.
    //    See https://cli.vuejs.org/config/#devserver-proxy and https://webpack.js.org/configuration/dev-server#devserverproxy
    // Instead of hardcoding something lile https://localhost:5001/,
    // read the ASPNET_URL environment variable, injected by VueDevelopmentServerMiddleware
    devServer: {
      // When running in IISExpress, the env variable wont be provided. Hardcode it here based on your launchSettings.json
      proxy: process.env.ASPNET_URL
      //proxy: 'https://localhost:5001'
    },
    // Use source map for debugging in VS and VS Code
    devtool: 'source-map',
    // Breakpoints in VS and VSCode wont work since the source maps conside clien-app the project root, rather than its parent folder
    output: {
      devtoolModuleFilenameTemplate: info => {
        const resourcePath = info.resourcePath.replace('./src', './clientapp/src')
        return `webpack:///${resourcePath}?${info.loaders}`
      }
    }
  }
}
```

## 调试

**visual code vue脚本调试** 要添加 Debug for Chrome 插件，并配置Launch.json中的启动参数:
新增 **Launch Chrome** 和 **.NETCore Launch (console)** (将浏览器启动禁用)两个启动项，并添加 **.Net+Browser** 启动选项(*同时启动.NET Core Launch (console)和Launch Chrome*)

```json
{
  // Use IntelliSense to find out which attributes exist for C# debugging
  // Use hover for the description of the existing attributes
  // For further information visit https://github.com/OmniSharp/omnisharp-vscode/blob/master/debugger-launchjson.md
  "version": "0.2.0",
  "compounds": [
    {
        "name": ".Net+Browser",
        "configurations": [ ".NET Core Launch (console)", "Launch Chrome" ]
    }
  ],
  "configurations": [
    {
        "name": ".NET Core Launch (console)",
        "type": "coreclr",
        "request": "launch",
        "preLaunchTask": "build",
        // If you have changed target frameworks, make sure to update the program path.
        "program": "${workspaceFolder}/bin/Debug/netcoreapp3.1/AspNetCoreVueCliTemplate.dll",
        "args": [],
        "cwd": "${workspaceFolder}",
        "stopAtEntry": false,
        "launchBrowser": {
            "enabled": false
        },
        "env": {
            "ASPNETCORE_ENVIRONMENT": "Development"
        },
        "sourceFileMap": {
            "/Views": "${workspaceFolder}/Views"
        }
    },
    {
        "name": "Launch Chrome",
        "request": "launch",
        "type": "chrome",
        "url": "http://localhost:5000",
        "webRoot": "${workspaceFolder}/clientapp/src",
        "sourceMaps": true
    },
    {
        "name": ".NET Core Launch (web)",
        "type": "coreclr",
        "request": "launch",
        "preLaunchTask": "build",
        // If you have changed target frameworks, make sure to update the program path.
        "program": "${workspaceFolder}/bin/Debug/netcoreapp3.1/AspNetCoreVueCliTemplate.dll",
        "args": [],
        "cwd": "${workspaceFolder}",
        "stopAtEntry": false,
        // Enable launching a web browser when ASP.NET Core starts. For more information: https://aka.ms/VSCode-CS-LaunchJson-WebBrowser
        "serverReadyAction": {
            "action": "openExternally",
            "pattern": "^\\s*Now listening on:\\s+(https?://\\S+)"
        },
        "env": {
            "ASPNETCORE_ENVIRONMENT": "Development"
        },
        "sourceFileMap": {
            "/Views": "${workspaceFolder}/Views"
        }
    },
    {
        "name": ".NET Core Attach",
        "type": "coreclr",
        "request": "attach",
        "processId": "${command:pickProcess}"
    }
  ]
}
```

这样设置后，vs code调试窗口就会出现 **.Net+Browser** 选项.启动后就可以正常开发调试了。

## 结束

主要就是这几部分。这样就可以实现AspNetCore和vue的集成了。关于AspNetCore 和 Vue 都还处于入门阶段。很多观念较为模糊。希望能坚持研究下去，即使仅仅作为一个.net 爱好者。

以上。

[ASP.NET Core Vue CLI Templates]: https://www.dotnetcurry.com/aspnet-core/1500/aspnet-core-vuejs-template
