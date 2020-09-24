---
layout: post
title: A New Microservice with PowerShell
category: blogs
tag: Microservices
---

Since the choice of microservices implies that there will be a list of them, it is convenient to have a script that creates a new solution and adds common elements required for each service.

<pre><code class="language-cs">
param ($ServiceName = $(throw "ServiceName parameter is required."))

function CreateApiService {

	#Creates the service
	dotnet new webapi -o $ServiceName
	#Adds packages
	dotnet add ".\$ServiceName\$($ServiceName).csproj" package FluentValidation
	dotnet add ".\$ServiceName\$($ServiceName).csproj" package AutoMapper
	dotnet add ".\$ServiceName\$($ServiceName).csproj" package Serilog.Sinks.Logentries

	#Creates xUnit tests project
	dotnet new xunit -o "$($ServiceName).Tests"
	#Adds reference to the service
	dotnet add ".\$($ServiceName).Tests\$($ServiceName).Tests.csproj" reference ".\$ServiceName\$($ServiceName).csproj"

	#Creates the solution
	dotnet new sln -n $ServiceName
	#Adds references to the projects
	dotnet sln "$($ServiceName).sln" add "$ServiceName\$($ServiceName).csproj"
	dotnet sln "$($ServiceName).sln" add "$($ServiceName).Tests\$($ServiceName).Tests.csproj"
}

CreateApiService 
</code></pre>
Save this code into `NewService.ps1`. Then execution is as easy as scripting.
<pre><code class="nohighlight">.\NewService.ps1 -ServiceName Akov.MyService</code></pre>

If running scripts is forbidden in `PowerShell` by default, then execute this command first.
<pre><code class="nohighlight">Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass</code></pre>

Bear in mind that in .NET 5 the reference to `Swashbuckle.AspNetCore` will be included automatically while creating a new Web API project.