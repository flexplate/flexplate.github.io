---
published: true
layout: post
title: Automating version numbering in .Net Standard/Core projects
---
Note: This is the second in an irregular series on automating version numbering. For part 1, [click here]({{ site.baseurl }}{% link _posts/2020-06-12-automating-version-numbering.md %})

## A bit of background.

WinForms projects (and Windows programs in general) use a 4 part version number in the format `[major].[minor].[build].[revision]`. In previous versions of the .Net Framework we were able to tell MSBuild to automatically increment parts of these by setting either the revision or build part of the version number to an asterisk (*), either directly in _AssemblyInfo.cs_ or in the Visual Studio UI, which then updates the file itself.

However, .Net 5 (and its predecessors .Net Core and .Net Standard) depart from this behaviour by automatically generating _AssemblyInfo.cs_, leaving us devs with a conundrum - we don't want the backward step of having to increment our version numbers by hand again, how can we get the computer to do it for us?

## The .Net 5 approach

In .Net 5, everything we need to automate is now set up from the .csproj file - and even better, MSBuild lets us use variables and conditional code to generate our output. To begin with, let's look at this excellent example from multiple-time MVP Sacha Barber [(link)](https://sachabarbs.wordpress.com/2020/02/23/net-core-standard-auto-incrementing-versioning/):

```xml 
<PropertyGroup>
  <VersionSuffix>1.0.0.$([System.DateTime]::UtcNow.ToString(mmff))</VersionSuffix>
  <AssemblyVersion Condition=" '$(VersionSuffix)' == '' ">0.0.0.1</AssemblyVersion>
  <AssemblyVersion Condition=" '$(VersionSuffix)' != '' ">$(VersionSuffix)</AssemblyVersion>
  <Version Condition=" '$(VersionSuffix)' == '' ">0.0.1.0</Version>
  <Version Condition=" '$(VersionSuffix)' != '' ">$(VersionSuffix)</Version>
</PropertyGroup> 
```

I've taken the liberty of stripping out most of the code from this to show just the salient points as Sacha's excellent blog post included the whole .csproj file. As you can see, the file lets us define our own variables - _VersionSuffix_ is not a regular C# project file element. MSBuild also lets us use static methods from a lot of the .Net standard's built-in types. At the moment all we're interested in is System.DateTime, but for a complete list [see here](https://docs.microsoft.com/en-us/visualstudio/msbuild/property-functions?view=vs-2022). By defining multiple _AssemblyVersion_ (and _Version_) elements, we can use the _Condition_ attribute to make sure _VersionSuffix_ has a value - if it does, we use that. 

## Taking it further

This is great, but I wanted to try and replicate the exact numbering we get from earlier versions of MSBuild. In those, setting the build part of the version string to "*" would set that part to the number of days since the 1st of January 2000, and the revision part to the number of seconds since midnight divided by 2. Why Microsoft chose that specifically is beyond me, but since I was trying to get as close as possible to the old behaviour I wanted to implement it. This requires a little more work:

```xml
<Millennium>$([System.DateTime]::Parse(`2000,1,1`))</Millennium>
<VersionBuildPart>$([System.DateTime]::UtcNow.Subtract($(Millennium)).Days)</VersionBuildPart>
<VersionRevisionPart>$([System.Convert]::ToUInt16($([MSBuild]::Divide($([System.DateTime]::UtcNow.TimeOfDay.TotalSeconds),2))))</VersionRevisionPart>
<AssemblyVersion Condition=" '$(VersionBuildPart)' == '' OR '$(VersionRevisionPart)' == ''">1.0.0.0</AssemblyVersion>
<AssemblyVersion Condition=" '$(VersionBuildPart)' != '' AND '$(VersionRevisionPart)' != ''">1.0.$(VersionBuildPart).$(VersionRevisionPart)</AssemblyVersion>
```

How this works should be pretty obvious, but for the sake of completeness:
* We instantiate a new DateTime called _Millennium_ to 1/1/2000.
* We subtract that from the static DateTime UtcNow and store the resulting TimeSpan's Days property in a variable called _VersionBuildPart_.
* We divide the number of seconds from UtcNow by 2, convert it to an integer using an MSBuild built-in method and store that as _VersionRevisionPart_.
* If either part is blank, we set the _AssemblyVersion_ to a set value of 1.0.0.0. (Not likely, as it's much more likely that the build process will fail if it can't process the dates properly.)
* Otherwise, we concatenate our set major and minor version numbers (1.0 in this case) with our calculated build and revision numbers.

## Determinism

There's one last hoop to jump through when automating this stuff: since the code is being modified at build-time, by necessity the resulting binary is going to differ with every build because of this, we need to tell MSBuild that we don't want the build to be deterministic. Placing this in the same PropertyGroup as the above version numbering code is sufficient:

```xml
<Deterministic>False</Deterministic>
```

Now, hit build and watch as your version number increments itself just like it does in .Net Framework.
