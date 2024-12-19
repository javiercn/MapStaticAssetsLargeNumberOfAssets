# Guidance for handling large assets collections with MapStaticAssets

It is recommended that you trim the set of assets in the application to those that are actually needed. All static assets need to be processed at build and publish time to be served through MapStaticAssets and leverage the optimizations applied to them. This can happen if you are using tools like libman, npm or yarn to pull in JS packages, icon libraries, etc.

If your app contains a large amount of assets, you may want to consider the following strategies to reduce the number of assets that need to be processed:

## Identify the folders in your app that contain a large number of assets

The following powershell command can be used to list all the folders in your project that contribute a large number of assets to your app. This command will list all the folders in your project that contain more than the given threshold number of files in their subtree.

```powershell
function Get-TotalFileCounts {
  param (
    [string]$Path,
    [int]$Threshold = 100
  )

  $folders = Get-ChildItem -Path $Path -Recurse -Directory
  $results = @()

  foreach ($folder in $folders) {
    $exclusiveCount = (Get-ChildItem -Path $folder.FullName -File).Count
    $inclusiveCount = (Get-ChildItem -Path $folder.FullName -Recurse -File).Count

    $results += [PSCustomObject]@{
      Path           = $folder.FullName
      ExclusiveCount = $exclusiveCount
      InclusiveCount = $inclusiveCount
    }
  }

  $filteredResults = $results | Where-Object { $_.InclusiveCount -ge $Threshold }
  $sortedResults = $filteredResults | Sort-Object { $_.InclusiveCount } -Descending;
  return $sortedResults;
}
```

## Exclude those folders from processing

You can do so by adding the following snippet to your project file:

```xml
<PropertyGroup>
  <!-- Include here the files to exclude -->
  <ExcludedAssetPatterns>wwwroot\lib\**</ExcludedAssetPatterns>
  <DefaultItemExcludes>$(DefaultItemExcludes);$(ExcludedAssetPatterns)</DefaultItemExcludes>
</PropertyGroup>

<!-- Just for the elements to be shown in VS -->
<ItemGroup>
  <None Include="$(ExcludedAssetPatterns)" />
</ItemGroup>
```

With this, the folders will not be processed by static web assets and won't be taken into account. During development, static web assets still maps the wwwroot folder in the project, so you can still access the files in the excluded folders. That said, since those files are not processed, you need to add a call to `UseStaticFiles` to your app to serve them.

Assets that are processed by StaticWebAssets will be handled by MapStaticAssets and will benefit from all the optimizations applied to them (compression, caching, content-unique URLs, etc), the remaining assets will be served by the default static files middleware.

## Ensure that files get copied to the output directory during publish

If you exclude a folder from processing, you may need to ensure that the files in that folder are copied to the output directory during publish. You can do so by adding the following snippet to your project file:

```xml
<Target Name="AddExcludedFiles" BeforeTargets="GetCopyToPublishDirectoryItems">
  <ItemGroup>
    <ExcludedAsset Include="$(ExcludedAssetPatterns)" CopyToPublishDirectory="PreserveNewest" />
  </ItemGroup>
  <AssignTargetPath Files="@(ExcludedAsset)" RootFolder="$(MSBuildProjectDirectory)">
    <Output TaskParameter="AssignedFiles" ItemName="ContentWithTargetPath" />
  </AssignTargetPath>
</Target>
```

## Add a call to UseStaticFiles in your Program.cs

```diff
using MapStaticAssetsLargeNumberOfAssets.Components;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddRazorComponents();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error", createScopeForErrors: true);
    // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
    app.UseHsts();
}

app.UseHttpsRedirection();
+app.UseStaticFiles();

app.UseAntiforgery();

app.MapStaticAssets();
app.MapRazorComponents<App>();

app.Run();
```
