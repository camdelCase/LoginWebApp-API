# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

ASP.NET Core Web API (.NET 10, `net10.0`) that provides login. It uses Serilog for logging and Swashbuckle for Swagger UI (`/swagger`, Development only). There is no test project yet.

## Commands

```
dotnet build                          # build (solution file: LoginWebApp.Api.slnx)
dotnet run --launch-profile http      # http://localhost:5154
dotnet run --launch-profile https     # https://localhost:7255 + http://localhost:5154
dotnet ef migrations add <Name> --output-dir Data/Migrations
dotnet ef database update             # apply migrations to the LocalDB database
docker build -t loginwebapp-api .       # container listens on 8080 (http) / 8081 (https)
nswag run nswag.json                  # regenerate Clients/ApiClient.cs (app must be running on :5154)
```

`LoginWebApp.Api.http` holds sample register → login → `/api/user/me` requests.

## Authentication: two providers, one switch

`Auth:Provider` in configuration selects the provider. All wiring lives in `Model/Auth/AuthServiceExtensions.cs` (`AddAppAuthentication`). `AuthController` (`/api/user/auth/login`, `/register`, `/confirm`) serves the app's own login page in both modes and depends only on `IAuthService`, which has one implementation per provider. Both providers use JWT bearer authentication for protected endpoints:

- **Local** (set in `appsettings.Development.json`): `LocalAuthService` stores users in SQL Server LocalDB (`Model/AppDbContext`, connection string `DefaultConnection`), hashes passwords with `PasswordHasher<User>`, and issues HS256 tokens through `TokenService`. Users are active immediately, so `/confirm` returns 400. The signing key `Auth:Local:SigningKey` lives in **user secrets** (at least 32 characters), not in appsettings. Startup throws if Local is used in the Production environment.
- **Cognito** (the default in `appsettings.json`): `CognitoAuthService` forwards requests to the user pool through `InitiateAuth` (USER_PASSWORD_AUTH, so the app client must allow `ALLOW_USER_PASSWORD_AUTH`), `SignUp` and `ConfirmSignUp`. These are public Cognito APIs, so the SDK client uses anonymous credentials. Set `Auth:Cognito:ClientSecret` only if the app client has a secret; the `SECRET_HASH` is then added automatically. Cognito challenges such as MFA or a forced new password come back as a 403 and aren't handled yet. Token validation uses the pool's OpenID metadata; because Cognito access tokens have no `aud` claim, `OnTokenValidated` checks the `client_id` and `token_use=access` claims instead.

Local tokens deliberately use Cognito's claim names (`sub`, `username`, `cognito:groups`; see `Model/Auth/AuthClaimTypes.cs`), and `MapInboundClaims = false` keeps them unrenamed. Read claims through `AuthClaimTypes` so code works under both providers. Identify users by `sub`, never by a local `Users` row, because that table doesn't exist under Cognito.

## Logging

Serilog is configured entirely by the `Serilog` section of `appsettings.json` (console sink only; stdout is what Docker and AWS collect). The ASP.NET `Logging` section isn't used. `UseSerilogRequestLogging` is first in the pipeline, so it logs every request, including short-circuited 401s.

## Conventions

Controllers use block-scoped namespaces, constructor injection into `_camelCase` readonly fields, and `/// <summary>` docs on actions.
