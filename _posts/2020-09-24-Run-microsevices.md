---
layout: post
title: Run Microservices Solution with PowerShell
category: blogs
tag: Microservices
---

*Decide what you need to install locally, run everything else in docker containers.*

Below I will present the powershell example that starts a list of services.

<pre><code class="language-cs">
param ($Mode = $(throw "Mode parameter is required."))

function RunMongo {
	cd $path/docker/mongo
	docker-compose up -d
}

function RunKafka {
	cd $path/docker/kafka
	docker-compose up -d
}

function RunAPIGateway {
	cd $path/src/Services/APIGateway/Akov.APIGateway
	echo 'APIGateway'
	Start-Process "cmd" -ArgumentList "/c dotnet build -c $Mode && dotnet run"
}

function RunServices {
	cd $path/src/Services/AuthorizationService/Akov.AuthorizationService
	echo 'AuthorizationService'
	Start-Process "cmd" -ArgumentList "/c dotnet build -c $Mode && dotnet run"

	cd $path/src/Services/AccountService/Akov.AccountService
	echo 'AccountService'
	Start-Process "cmd" -ArgumentList "/c dotnet build -c $Mode && dotnet run"
}

# On this line it could be helpful to run the cd command
$path = $pwd
RunMongo
RunKafka
RunAPIGateway
RunServices 
# On this line it could be helpful to run the cd command again
</code></pre>

Save this code into `Run.ps1` and execute:
<pre><code class="nohighlight">.\Run.ps1 -Mode Debug</code></pre>