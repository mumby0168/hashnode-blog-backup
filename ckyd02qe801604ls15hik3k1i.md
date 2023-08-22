---
title: "Nuget Packages & Github Actions"
seoTitle: "Nuget Packages & Github Actions"
seoDescription: "Nuget packages are great and allow developers to make use of useful libraries put together by other developers. The version number is often important to dev"
datePublished: Thu Jan 13 2022 13:21:09 GMT+0000 (Coordinated Universal Time)
cuid: ckyd02qe801604ls15hik3k1i
slug: github-actions-and-nuget-packages
cover: https://cdn.hashnode.com/res/hashnode/image/unsplash/q8kR_ie6WnI/upload/v1642079787303/5Od4up4oG.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1642079975783/K1XCAyehA.png
tags: github, dotnet, dotnetcore, github-actions-1

---

# Deploying Nuget Packages (GitHub Actions)

Nuget packages are great and allow developers to make use of useful libraries put together by other developers. The version number is often important to developers many use the common pattern of `Major.Minor.Patch` in order to communicate changes to a package with other developers.

`Major` version increase usually represents a change to the public API surface that would likely require a developer to update their code that uses the API provided. `Minor` version increases, maybe a new feature to the package that maybe adds to the API surface, while maintaining backward compatibility with the previous versions. `Patch` usually contains a set of bug fixes that again are backward compatible.

## Integrating with CI/CD

There are many, many nuget packages which are open source, and authors often encourage contributors to help maintain & evolve their packages. The most common way for this to be done currently is usually via GitHub. This allows developers, who are using a package, to see the source code, raise issues, fix bugs, and also fork the repository to later be merged back into the main repository with any changes they might make.

GitHub also offers Actions that allow you to perform continuous integration (CI) & continuous delivery (CD) of a piece of software. We can make use of GitHub tools to help us manage, deploying a package to [nuget.org](https://www.nuget.org/). 

## The Deployment Plan

So in order to do this, we want to really be creating a release of the package and as this happens and the version is specified by the individual making the release along with, detailing what that release includes. Then a workflow is triggered to build, test & deploy the package to nuget.org. GitHub offers this in the form of releases which at the same time tags a commit on a branch usually `main/master`  with the version number specified. This [article](https://guides.github.com/activities/citable-code/) details how to create a release in GitHub here is a sneak peek at what that form looks like and what information it can include. The final step once we have created this release we want a GitHub Actions workflow to run, build, test, and pack our source code, apply for the version number and push it up to [nuget.org](https://www.nuget.org/) which we will do in the next section.

> Note where the version is specified.


![Screenshot 2022-01-13 at 13.13.24.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1642079608657/s1r054BWb.png)

## Putting the Plan into (GitHub) Action(s)

So in order to implement this, we need a new pipeline within the repository that contains the library we want to package up. This is triggered when a tag is applied to a branch, more than likely the main branch. A sample pipeline is shown below, each step is explained after this snippet.

```yaml
name: Release Package Version
on:
  push:  
    tags:
     - v*     
jobs: 
  build:
    if: github.event.base_ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:    
    
    - name: Print Tag Ref
      run: echo $GITHUB_REF
      
    - name: Extract Version Number
      uses: actions-ecosystem/action-regex-match@v2
      id: regex-match
      with:
        text: ${{ github.ref }}
        regex: '[0-9.]+'
       
    - name: Print Version Number
      run: echo '${{ steps.regex-match.outputs.match }}'
      
    - uses: actions/checkout@v2
    - name: Setup .NET  
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 5.0.x
        
    - name: Restore dependencies
      run: dotnet restore ./Fruitless/Fruitless.csproj
      
    - name: Build
      run: dotnet build ./Fruitless/Fruitless.csproj
      
    - name: Pack
      run: dotnet pack ./Fruitless/Fruitless.csproj -p:PackageVersion='${{ steps.regex-match.outputs.match }}' --output packages
      
    - name: Publish Package
      run: nuget push **\*.nupkg -NoSymbols -Source 'https://api.nuget.org/v3/index.json' -ApiKey ${{secrets.NUGET_API_KEY}}
      
    - name: Upload Package
      uses: actions/upload-artifact@v2
      with:
        name: fruitless-pkg-v${{ steps.regex-match.outputs.match }}
        path: packages/
```

### The pipeline explained

1. So the initial section of this pipeline is stating that this pipeline will only be triggered when a tag is applied to a branch that starts with a `v`.

2. The next section details a single job to run called `build` and this build has an if the condition that means if the branch is not `main` then this job will not run and will be skipped. This contains a set of steps detailed below.

3. It then prints the tag name that has triggered this pipeline.

4. It is then using a community provided regex tool to match the `Major.Minor.Patch` section from a string. (A tag of `v1.2.3` would extract `1.2.3`).

5. It then simply prints out the extracted version number extract via the match step.

6. Next it is just setting up the dotnet sdk version to use.

7. Then the project is built using `dotnet build` (tests can also be run at this point using `dotnet test`).
 
8. Then the package is packed using the package version extracted from step 4.

9. Last but not least, the package is pushed using an API key for [nuget.org](https://www.nuget.org/) stored as a secret for this GitHub repository.

10. Finally, the `.nupkg` file/files are uploaded as an artifact for that build inside of GitHub actions.

## Final Words

This is a simple way to get started with versioning packages and having them published via GitHub actions. The repository that this sample comes from is [here](https://github.com/mumby0168/tag_release_package). You can also see my trial and error here (everything doesn't always work the first time).

This can be expanded on in order to remove the regex part of the pipeline is to make use of the nuget package [MinVer](https://github.com/adamralph/minver). This approach has been taken on one of my OSS projects and can be found [here](https://github.com/IEvangelist/azure-cosmos-dotnet-repository).

There are a few concepts glossed over here in terms of how GitHub Actions works and some of its syntax the documentation can be found [here](https://docs.github.com/en/actions).