# Cloud Security

Steeltoe provides a number of security-related services that simplify using Cloud Foundry-based security services in ASP.NET applications.

These providers enable the use of Cloud Foundry security services (such as [UAA Server](https://github.com/cloudfoundry/uaa) and [Single Sign-On for VMware Tanzu](https://techdocs.broadcom.com/us/en/vmware-tanzu/platform-services/single-sign-on-for-tanzu/1-16/sso-tanzu/index.html)) for authentication and authorization.

You can choose from the following providers when adding Cloud Foundry security integration:

* [Single Sign-on with OAuth2 for ASP.NET Core](sso-oauth2.md)
* [Single Sign-on with OpenID Connect for ASP.NET Core](sso-open-id.md)
* [Resource protection with JWT Bearer tokens for ASP.NET Core](jwt-authentication.md)
* [Resource protection using Mutual TLS (Certificate Authorization) in ASP.NET Core](mtls.md)

In addition to authentication and authorization providers, Steeltoe security offers:

* [A security provider for using Redis/Valkey on Cloud Foundry with ASP.NET Core Data Protection Key Ring storage](redis-key-storage-provider.md)
* [A CredHub API Client for .NET applications to perform credential storage, retrieval, and generation](credhub-api-client.md)
