# Repository Guide

## Structure

- `Orleans.Providers.MongoDB/` is the shipped provider library; its main integration surface is the Orleans builder/service extension code in `MongoDBSiloExtensions.cs` and related membership, reminders, storage, and serializer folders.
- `UnitTest/` contains xUnit tests and references both the provider and `libs/orleans/test/TesterInternal`; `Test/` contains the sample host, grain interfaces, and grain implementations.
- `libs/orleans` is a required git submodule containing the Orleans source used by the test project. Initialize it recursively before building or testing the full solution: `git submodule update --init --recursive`.

## Build And Test

- Build the library in the same configuration as CI with `dotnet build Orleans.Providers.MongoDB/Orleans.Providers.MongoDB.csproj --configuration Release`.
- Run the unit tests from `UnitTest/`: `dotnet test UnitTest/Orleans.Providers.MongoDB.UnitTest.csproj --configuration Release`. Tests use `MongoSandbox.Core` and start local MongoDB processes, so the platform-specific MongoSandbox runtime package must be available.
- The test project defaults to MongoDB 8. Test MongoDB 6 or 7 explicitly with `-p:MongoVersion=6` or `-p:MongoVersion=7`; CI runs the test command for all three versions.
- Run one test with the standard filter, for example: `dotnet test UnitTest/Orleans.Providers.MongoDB.UnitTest.csproj --filter FullyQualifiedName~StorageTests --configuration Release`.
- Package only the provider project with `dotnet pack Orleans.Providers.MongoDB/Orleans.Providers.MongoDB.csproj --configuration Release`; release CI supplies `/p:Version=<tag-version>` when publishing a `v*.*.*` tag.
- Benchmarks are a separate project; run them with `dotnet run --project Benchmarks/Orleans.Providers.MongoDB.Benchmarks.csproj --configuration Release -- -c 10`.

## Change Boundaries

- Provider behavior is organized by Orleans concern: `Membership/`, `Reminders/`, `StorageProviders/`, and `StorageProviders/Serializers/`. Update the corresponding focused tests under `UnitTest/` when changing one of these paths.
- Keep MongoDB client registration and Orleans integration changes aligned with the public extension methods in `MongoDBSiloExtensions.cs`, `MongoDBClientExtensions.cs`, and `ServiceCollectionExtensions.cs`; do not bypass those registrations in tests or examples.
- CI checks the `master` branch, builds with .NET 10 on Ubuntu, and checks out submodules recursively. A local solution build can include many projects from `libs/orleans`; use the provider project or `UnitTest` project for focused work.
