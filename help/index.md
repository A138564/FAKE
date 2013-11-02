# FAKE - F# Make - A DSL for build tasks

"FAKE - F# Make" is a build automation system with capabilities which are similar to **make** and **rake**. 
It is using an easy domain-specific language (DSL) so that you can start using it without learning F#.
If you need more than the default functionality you can either write F# or simply reference .NET assemblies.

### Simple Example

    #r @"tools\FAKE\tools\FakeLib.dll" // include Fake lib
	open Fake 

	
	Target "Test" (fun _ ->
		trace "Testing stuff..."
	)

	Target "Deploy" (fun _ ->
		trace "Heavy deploy action"
	)

	"Test"            // define the dependencies
	   ==> "Deploy"
	
	Run "Deploy"

This build script has two targets. The "Deploy" target has exactly one dependency, namely the "Test" target. Invoking the "Deploy" target (line 16) will cause FAKE to invoke the "Test" target as well.

## Who is using FAKE?

Some of our users are:

* [msu solutions GmbH](http://www.msu-solutions.de/)
* E.On Global Commodities UK
* [Octokit](https://github.com/octokit/octokit.net/) by GitHub
* [BlueMountainCapital](https://github.com/BlueMountainCapital/Deedle)
* [fsharpx](https://github.com/fsharp/fsharpx)
* [FSharp.Data](https://github.com/fsharp/FSharp.Data)
* [FSharp.Charting](https://github.com/fsharp/FSharp.Charting)
* [Portable.Licensing](https://github.com/dnauck/Portable.Licensing)
* [FeatureSwitcher](https://github.com/mexx/FeatureSwitcher)

## How to get FAKE

You can download the latest builds from [http://teamcity.codebetter.com](http://teamcity.codebetter.com). You don't need to register, a guest login is ok.

* [Latest stable build](http://teamcity.codebetter.com/viewLog.html?buildId=lastSuccessful&buildTypeId=bt335&tab=artifacts&guest=1)
* [Latest development build](http://teamcity.codebetter.com/viewLog.html?buildId=lastSuccessful&buildTypeId=bt166&tab=artifacts&guest=1)
* [Changelog](changelog.html)

<div class="row">
  <div class="span1"></div>
  <div class="span6">
    <div class="well well-small" id="nuget">
      FAKE is also available <a href="https://nuget.org/packages/FAKEe">on NuGet</a>.
      To install the tool, run the following command in the <a href="http://docs.nuget.org/docs/start-here/using-the-package-manager-console">Package Manager Console</a>:
      <pre>PM> Install-Package FAKE</pre>
    </div>
  </div>
  <div class="span1"></div>
</div>

# Using FAKE

If you want to learn about FAKE you should read the ["Getting started with FAKE"](gettingstarted.html) tutorial.

## Targets

Targets are the main unit of work in a "FAKE - F# Make" script. Targets have a name and an action (given as a code block).

	// The clean target cleans the build and deploy folders
	Target "Clean" (fun _ -> 
		CleanDirs ["./build/"; "./deploy/"]
	)

### Build target order

You can specify the build order using the ==> operator:
	
	// "FAKE - F# Make" will run these targets in the order Clean, BuildApp, Default
	"Clean" 
	  ==> "BuildApp" 
	  ==> "Default"

If one target should only be run on a specific condition you can use the =?> operator:
	
	"Clean" 
	  ==> "BuildApp"
	  =?> ("Test",hasBuildParam "test")  // runs the Test target only if FAKE was called with parameter test
	  ==> "Default"

It's also possible to specify the dependencies for targets:

    // Target Default is dependent from target Clean and BuildApp
    // "FAKE - F# Make" will ensure to run these targets before Default
    "Default"  <== ["Clean"; "BuildApp"]

### Running targets

You can execute targets with the "run"-command:

	// Executes Default target
	Run "Default"

### Final targets

Final target can be used for TearDown functionality. 
These targets will be executed even if the build fails but have to be activated via ActivateFinalTarget().

	// FinalTarget will be excuted even if build fails
	FinalTarget "CloseSomePrograms" (fun _ ->
		// close stuff and release resources
	)

	// Activate FinalTarget somewhere during build
	ActivateFinalTarget "CloseSomePrograms"

## FileSets

"FAKE - F# Make" uses similar include and exclude patterns as NAnt and MSBuild. 

### File includes

	// Includes all *.csproj files under /src/app by using the !+ operator
	!+ "src/app/**/*.csproj"

	// Includes all *.csproj files under /src/app and /test with the ++ operator
	!+ "src/app/**/*.csproj"
	  ++ "test/**/*.csproj"

### File excludes

	// Includes all files under /src/app but excludes *.zip files
	!+ "src/app/**/*.*"
	  -- "*.zip"

### Scan vs. ScanImmediately

"FAKE - F# Make" provides two scan methods: Scan() and ScanImmediately().

Scan is a lazy method and evaluates the FileSet as late as possible ("on-demand").
If the FileSet is used twice, it will be reevaluated.

The following code defines a lazy FileSet:

	// Includes all *.csproj files under /src/app and scans them lazy
	let apps = 
	  !+ "src/app/**/*.csproj"
		|> Scan

The same FileSet by using the !! operator:

    // Includes all *.csproj files under /src/app and scans them lazy
    let apps = !! "src/app/**/*.csproj"

ScanImmediately() scans the FileSet immediatly at time of its definition
and memoizes it. 

	// Includes all files under /src/app but excludes *.zip files
	//	  eager scan ==> All files memoized at the time of this definition
	let files = 
	  !+ "src/app/**/*.csproj"
		-- "*.zip"
		|> ScanImmediately

## UnitTests

### NUnit

	// define test dlls
	let testDlls = !! (testDir + @"/Test.*.dll")
	 
	Target "NUnitTest" (fun _ ->
		testDlls
			|> NUnit (fun p -> 
				{p with 
					ToolPath = nunitPath; 
					DisableShadowCopy = true; 
					OutputFile = testDir + "TestResults.xml"})
    )
							 
### MSpec
	// define test dlls
	let testDlls = !! (testDir + @"/Test.*.dll")
	 
	Target "MSpecTest" (fun _ ->
		testDlls
			|> MSpec (fun p -> 
				{p with 
					ExcludeTags = ["LongRunning"]
					HtmlOutputDir = testOutputDir						  
					ToolPath = ".\toools\MSpec\mspec.exe"})
    )

### xUnit.net

	// define test dlls
	let testDlls = !! (testDir + @"/Test.*.dll")

	Target "xUnitTest" (fun _ ->
        testDlls
            |> xUnit (fun p -> 
                {p with 
                    ShadowCopy = false;
                    HtmlOutput = true;
                    XmlOutput = true;
                    OutputDir = testDir })
    )

## Sample script

This sample script
  
* Assumes "FAKE - F# Make" is located at ./tools/FAKE
* Assumes NUnit is located at ./tools/NUnit  
* Cleans the build and deploy paths
* Builds all C# projects below src/app/ and puts the output to .\build
* Builds all NUnit test projects below src/test/ and puts the output to .\build
* Uses NUnit to test the generated Test.*.dll's
* Zips all generated files to deploy\MyProject-0.1.zip
  
You can read the [getting started guide](gettingstarted.html) to build such a script.

    // include Fake libs
    #r "tools/FAKE/FakeLib.dll"
    
    open Fake
    
    // Directories
    let buildDir  = @".\build\"
    let testDir   = @".\test\"
    let deployDir = @".\deploy\"
    
    // tools
    let nunitPath = @".\Tools\NUnit"
    let fxCopRoot = @".\Tools\FxCop\FxCopCmd.exe"
    
    // Filesets
    let appReferences  = 
        !+ @"src\app\**\*.csproj" 
          ++ @"src\app\**\*.fsproj" 
            |> Scan
    
    let testReferences = !! @"src\test\**\*.csproj"
        
    // version info
    let version = "0.2"  // or retrieve from CI server
    
    // Targets
    Target "Clean" (fun _ -> 
        CleanDirs [buildDir; testDir; deployDir]
    )
    
    Target "BuildApp" (fun _ ->
        AssemblyInfo 
            (fun p -> 
            {p with
                CodeLanguage = CSharp;
                AssemblyVersion = version;
                AssemblyTitle = "Calculator Command line tool";
                AssemblyDescription = "Sample project for FAKE - F# MAKE";
                Guid = "A539B42C-CB9F-4a23-8E57-AF4E7CEE5BAA";
                OutputFileName = @".\src\app\Calculator\Properties\AssemblyInfo.cs"})
                  
        AssemblyInfo 
            (fun p -> 
            {p with
                CodeLanguage = CSharp;
                AssemblyVersion = version;
                AssemblyTitle = "Calculator library";
                AssemblyDescription = "Sample project for FAKE - F# MAKE";
                Guid = "EE5621DB-B86B-44eb-987F-9C94BCC98441";
                OutputFileName = @".\src\app\CalculatorLib\Properties\AssemblyInfo.cs"})          
          
        // compile all projects below src\app\
        MSBuildRelease buildDir "Build" appReferences
            |> Log "AppBuild-Output: "
    )
    
    Target "BuildTest" (fun _ ->
        MSBuildDebug testDir "Build" testReferences
            |> Log "TestBuild-Output: "
    )
    
    Target "NUnitTest" (fun _ ->  
        !! (testDir + @"\NUnit.Test.*.dll")
            |> NUnit (fun p -> 
                {p with 
                    ToolPath = nunitPath; 
                    DisableShadowCopy = true; 
                    OutputFile = testDir + @"TestResults.xml"})
    )
    
    Target "xUnitTest" (fun _ ->  
        !! (testDir + @"\xUnit.Test.*.dll")
            |> xUnit (fun p -> 
                {p with 
                    ShadowCopy = false;
                    HtmlOutput = true;
                    XmlOutput = true;
                    OutputDir = testDir })
    )
    
    Target "FxCop" (fun _ ->
        !+ (buildDir + @"\**\*.dll") 
            ++ (buildDir + @"\**\*.exe") 
            |> Scan  
            |> FxCop (fun p -> 
                {p with                     
                    ReportFileName = testDir + "FXCopResults.xml";
                    ToolPath = fxCopRoot})
    )
    
    Target "Deploy" (fun _ ->
        !+ (buildDir + "\**\*.*") 
            -- "*.zip" 
            |> Scan
            |> Zip buildDir (deployDir + "Calculator." + version + ".zip")
    )
    
    // Build order
	"Clean"
      ==> "BuildApp" <=> "BuildTest"
      ==> "FxCop"
      ==> "NUnitTest"
      =?> ("xUnitTest",hasBuildParam "xUnitTest")  // runs the target only if FAKE was called with parameter xUnitTest
      ==> "Deploy"
     
    // start build
    Run "Deploy"
