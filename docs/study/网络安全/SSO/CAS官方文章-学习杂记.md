# CAS官方文章-学习杂记

> - [CAS | Apereo](https://www.apereo.org/projects/cas)
> - [CAS - Home (apereo.github.io)](https://apereo.github.io/cas/6.3.x/index.html)    <=   目前按照当前最新的6.3.X版本进行学习(**以下内容皆出自于此**)

# CAS Enterprise Single Sign-On

Welcome to the home of the Apereo Central Authentication Service project, more commonly referred to as CAS. CAS is an enterprise multilingual single sign-on solution for the web and attempts to be a comprehensive platform for your authentication and authorization needs.

CAS is an open and well-documented authentication protocol. The primary implementation of the protocol is an open-source Java server component by the same name hosted here, with support for a plethora of additional authentication protocols and features.

The following items include a summary of features and technologies presented by the CAS project:

- [Spring Webflow](https://apereo.github.io/cas/6.3.x/webflow/Webflow-Customization.html)/Spring Boot [Java server component](https://apereo.github.io/cas/6.3.x/planning/Architecture.html).
- [Pluggable authentication support](https://apereo.github.io/cas/6.3.x/installation/Configuring-Authentication-Components.html) ([LDAP](https://apereo.github.io/cas/6.3.x/installation/LDAP-Authentication.html), [Database](https://apereo.github.io/cas/6.3.x/installation/Database-Authentication.html), [X.509](https://apereo.github.io/cas/6.3.x/installation/X509-Authentication.html), [SPNEGO](https://apereo.github.io/cas/6.3.x/installation/SPNEGO-Authentication.html), [JAAS](https://apereo.github.io/cas/6.3.x/installation/JAAS-Authentication.html), [JWT](https://apereo.github.io/cas/6.3.x/installation/JWT-Authentication.html), [RADIUS](https://apereo.github.io/cas/6.3.x/mfa/RADIUS-Authentication.html), [MongoDb](https://apereo.github.io/cas/6.3.x/installation/MongoDb-Authentication.html), etc)
- Support for multiple protocols ([CAS](https://apereo.github.io/cas/6.3.x/protocol/CAS-Protocol.html), [SAML](https://apereo.github.io/cas/6.3.x/protocol/SAML-Protocol.html), [WS-Federation](https://apereo.github.io/cas/6.3.x/protocol/WS-Federation-Protocol.html), [OAuth2](https://apereo.github.io/cas/6.3.x/protocol/OAuth-Protocol.html), [OpenID](https://apereo.github.io/cas/6.3.x/protocol/OpenID-Protocol.html), [OpenID Connect](https://apereo.github.io/cas/6.3.x/protocol/OIDC-Protocol.html), [REST](https://apereo.github.io/cas/6.3.x/protocol/REST-Protocol.html))
- Support for [multifactor authentication](https://apereo.github.io/cas/6.3.x/mfa/Configuring-Multifactor-Authentication.html) via a variety of providers ([Duo Security](https://apereo.github.io/cas/6.3.x/mfa/DuoSecurity-Authentication.html), [FIDO U2F](https://apereo.github.io/cas/6.3.x/mfa/FIDO-U2F-Authentication.html), [YubiKey](https://apereo.github.io/cas/6.3.x/mfa/YubiKey-Authentication.html), [Google Authenticator](https://apereo.github.io/cas/6.3.x/mfa/GoogleAuthenticator-Authentication.html), [Authy](https://apereo.github.io/cas/6.3.x/mfa/AuthyAuthenticator-Authentication.html), [Acceptto](https://apereo.github.io/cas/6.3.x/mfa/Acceptto-Authentication.html), etc.)
- Support for [delegated authentication](https://apereo.github.io/cas/6.3.x/integration/Delegate-Authentication.html) to external providers such as [ADFS](https://apereo.github.io/cas/6.3.x/integration/ADFS-Integration.html), Facebook, Twitter, SAML2 IdPs, etc.
- Built-in support for [password management](https://apereo.github.io/cas/6.3.x/password_management/Password-Management.html), [notifications](https://apereo.github.io/cas/6.3.x/webflow/Webflow-Customization-Interrupt.html), [terms of use](https://apereo.github.io/cas/6.3.x/webflow/Webflow-Customization-AUP.html) and [impersonation](https://apereo.github.io/cas/6.3.x/installation/Surrogate-Authentication.html).
- Support for [attribute release](https://apereo.github.io/cas/6.3.x/integration/Attribute-Release.html) including [user consent](https://apereo.github.io/cas/6.3.x/integration/Attribute-Release-Consent.html).
- [Monitor and track](https://apereo.github.io/cas/6.3.x/monitoring/Monitoring-Statistics.html) application behavior, statistics and logs in real time.
- Manage and register [client applications and services](https://apereo.github.io/cas/6.3.x/services/Service-Management.html) with specific authentication policies.
- [Cross-platform client support](https://apereo.github.io/cas/6.3.x/integration/CAS-Clients.html) (Java, .Net, PHP, Perl, Apache, etc).
- Integrations with [InCommon, Box, Office365, ServiceNow, Salesforce, Workday, WebAdvisor](https://apereo.github.io/cas/6.3.x/integration/Configuring-SAML-SP-Integrations.html), Drupal, Blackboard, Moodle, [Google Apps](https://apereo.github.io/cas/6.3.x/integration/Google-Apps-Integration.html), etc.

## Getting Started

We recommend reading the following documentation in order to plan and execute a CAS deployment.

- [Architecture](https://apereo.github.io/cas/6.3.x/planning/Architecture.html)
- [Getting Started](https://apereo.github.io/cas/6.3.x/planning/Getting-Started.html)
- [Installation Requirements](https://apereo.github.io/cas/6.3.x/planning/Installation-Requirements.html)
- [Installation](https://apereo.github.io/cas/6.3.x/installation/WAR-Overlay-Installation.html)
- [Blog](https://apereo.github.io/)

## 1. Architecture

![CAS Architecture Diagram](https://apereo.github.io/cas/6.3.x/images/cas_architecture.png)

![img](https://upload-images.jianshu.io/upload_images/12540413-041b3228c5e865e8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

### 1.1 System Components

The CAS server and clients comprise the two physical components of the CAS system architecture that communicate by means of various protocols.

### 1.2 CAS Server

The CAS server is Java servlet built on the Spring Framework whose primary responsibility is to authenticate users and grant access to CAS-enabled services, commonly called CAS clients, by issuing and validating tickets. An SSO session is created when the server issues a ticket-granting ticket (TGT) to the user upon successful login. A service ticket (ST) is issued to a service at the user’s request via browser redirects using the TGT as a token. The ST is subsequently validated at the CAS server via back-channel communication. These interactions are described in great detail in the CAS Protocol document.

### 1.3 CAS Clients

The term “CAS client” has two distinct meanings in its common use. A CAS client is any CAS-enabled application that can communicate with the server via a supported protocol. A CAS client is also a software package that can be integrated with various software platforms and applications in order to communicate with the CAS server via some authentication protocol (e.g. CAS, SAML, OAuth). CAS clients supporting a number of software platforms and products have been developed.

Platforms:

- Apache httpd Server ([mod_auth_cas module](https://github.com/Jasig/mod_auth_cas))
- Java ([Java CAS Client](https://github.com/apereo/java-cas-client))
- .NET ([.NET CAS Client](https://github.com/apereo/dotnet-cas-client))
- PHP ([phpCAS](https://github.com/Jasig/phpCAS))
- Perl (PerlCAS)
- Python (pycas)
- Ruby (rubycas-client)

Applications:

- Canvas
- Atlassian Confluence
- Atlassian JIRA
- Drupal
- Liferay
- uPortal
- …

When the term “CAS client” appears in this manual without further qualification, it refers to the integration components such as the Java CAS Client rather than to the application relying upon (a client of) the CAS server.

### 1.4 Supported Protocols

Clients communicate with the server by any of several supported protocols. All the supported protocols are conceptually similar, yet some have features or characteristics that make them desirable for particular applications or use cases. For example, the CAS protocol supports delegated (proxy) authentication, and the SAML protocol supports attribute release and single sign-out.

Supported protocols:

- [CAS (versions 1, 2, and 3)](https://apereo.github.io/cas/6.3.x/protocol/CAS-Protocol.html)
- [SAML 1.1 and 2](https://apereo.github.io/cas/6.3.x/protocol/SAML-Protocol.html)
- [OpenID Connect](https://apereo.github.io/cas/6.3.x/protocol/OIDC-Protocol.html)
- [OpenID](https://apereo.github.io/cas/6.3.x/protocol/OpenID-Protocol.html)
- [OAuth 2.0](https://apereo.github.io/cas/6.3.x/protocol/OAuth-Protocol.html)
- [WS Federation](https://apereo.github.io/cas/6.3.x/protocol/WS-Federation-Protocol.html)

### 1.5 Software Components

It is helpful to describe the CAS server in terms of three layered subsystems:

- Web (Spring MVC/Spring Webflow)
- [Ticketing](https://apereo.github.io/cas/6.3.x/ticketing/Configuring-Ticketing-Components.html)
- [Authentication](https://apereo.github.io/cas/6.3.x/installation/Configuring-Authentication-Components.html)

Almost all deployment considerations and component configuration involve those three subsystems. The Web tier is the endpoint for communication with all external systems including CAS clients. The Web tier delegates to the ticketing subsystem to generate tickets for CAS client access. The SSO session begins with the issuance of a ticket-granting ticket on successful authentication, thus the ticketing subsystem frequently delegates to the authentication subsystem.

The authentication system is typically only processing requests at the start of the SSO session, though there are other cases when it can be invoked (e.g. forced authentication).

#### Spring Framework

CAS uses the many aspects of the Spring Framework; most notably, [Spring MVC](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/mvc.html) and [Spring Webflow](https://projects.spring.io/spring-webflow). Spring provides a complete and extensible framework for the core CAS codebase as well as for deployers; it’s straightforward to customize or extend CAS behavior by hooking CAS and Spring API extension points. General knowledge of Spring is beneficial to understanding the interplay among some framework components, but it’s not strictly required.

#### Spring Boot

CAS is also heavily based on [Spring Boot](http://projects.spring.io/spring-boot/), which allows it to take an opinionated view of the Spring platform and third-party libraries to create a stand-alone web application without the hassle of XML configuration as much as possible. Spring Boot allows CAS to hide much of the internal complexity of its components and their configuration and instead provides auto-configuration modules that
and automatically configure the running application context without much manual interference.

## 2. Getting Started

This document provides a high-level guide on how to get started with a CAS server deployment. The sole focus of the guide is describe the process that must be followed and adopted by CAS deployers in order to arrive at a successful and sustainable architecture and deployment.

### 2.1 Collect Use Cases

It is very important that you document, catalog and analyze your desired use cases and requirements prior to the deployment. Once you have a few ideas, please discuss and share those with the [CAS community](https://apereo.github.io/cas/Support.html) to learn about common trends, practices and patterns that may already have solved the same issues you face today.

> In general, avoid designing and/or adopting use cases and workflows that heavily alter the CAS internal components, induce a heavy burden on your management and maintenance of the configuration or re-invent the CAS software and its supported protocols. All options add to maintenance cost and headache.

### 2.2 Study Architecture

Understand what CAS is and can do. This will help you develop a foundation to realize which of your use cases and requirements may already be possible with CAS. Take a look at the fundamentals of the [CAS architecture](https://apereo.github.io/cas/6.3.x/planning/Architecture.html) to see what options and choices might be available for deployments and application integrations.

Likewise, it’s equally important that you study the list of CAS [supported protocols and specifications](https://apereo.github.io/cas/6.3.x/protocol/Protocol-Overview.html).

### 2.3 Review Blog

From time to time, blog posts appears on the [Apereo Blog](https://apereo.github.io/) that might become useful as you are thinking about requirements and evaluating features. It is generally recommended that you follow the blog and keep up with project news and announcements as much as possible, and do not shy away from writing and contributing your own blog posts, experiences and updates throughout your CAS deployment.

### 2.4 Prepare Environment

Study the [installation requirements](https://apereo.github.io/cas/6.3.x/planning/Installation-Requirements.html) for the deployment environment.

### 2.5 Deploy CAS

It is recommended to build and deploy CAS locally using the [WAR Overlay method](https://apereo.github.io/cas/6.3.x/installation/WAR-Overlay-Installation.html). This approach does not require the adopter to *explicitly* download any version of CAS, but rather utilizes the overlay mechanism to combine CAS original artifacts and local customizations to further ease future upgrades and maintenance.

**Note**: Do NOT clone or download the CAS codebase directly. That is ONLY required if you wish to contribute to the development of the project.

It is **VERY IMPORTANT** that you try to get a functional baseline working before doing anything else. Avoid making ad-hoc changes right away to customize the deployment. Stick with the CAS-provided defaults and settings and make changes **one step at a time**. Keep track of process and applied changes in source control and tag changes as you make progress.

### 2.6 Customize

This is where use cases get mapped to CAS features. Browse the documentation to find the closest match and apply. Again, it is important that you stick with the CAS baseline as much as possible:

- Avoid making ad-hoc changes to the software internals.
- Avoid making manual changes to core configuration components such as Spring and Spring Webflow.
- Avoid making one-off bug fixes to the deployment, should you encounter an issue.

As noted previously, all such strategies lead to headache and cost.

Instead, try to warm up to the following suggestions:

- Bug fixes and small improvements belong to the core CAS software. Not your deployment. Make every attempt to report issues, contribute fixes and patches and work with the CAS community to solve issues once and for all.
- Certain number of internal CAS components are made difficult to augment and modify. In most cases, this approach is done on purpose to steer you away from dangerous and needlessly complicated changes. If you come across a need and have a feature or use case in mind whose configuration and implementation requires modifications to the core internals of the software, discuss that with the CAS community and attempt to build the enhancement directly into the CAS software, rather than treating it as a snowflake.

To summarize, only make changes to the deployment configuration if they are truly and completely specific to your needs. Otherwise, try to generalize and contribute back to keep maintenance costs down. Repeatedly, failure to comply with this strategy will likely lead to disastrous results in the long run.

### 2.7 Troubleshooting

The [troubleshooting guide](https://apereo.github.io/cas/6.3.x/installation/Troubleshooting-Guide.html) might have some answers for issues you may have run into and it generally tries to describe a strategy useful for troubleshooting and diagnostics. You may also seek assistance from the [CAS community](https://apereo.github.io/cas/Mailing-Lists.html).

## 3. Installation Requirements

Depending on choice of configuration components, there may be additional requirements such as LDAP directory, database, and caching infrastructure. In most cases, however, requirements should be self evident to deployers who choose components with clear hardware and software dependencies. In any case where additional requirements are not obvious, the discussion of component configuration should mention system, software, hardware, and other requirements.

### 3.1 Java

CAS at its heart is a Java-based web application. Prior to deployment, you will need to have [JDK](https://openjdk.java.net/projects/jdk/11/) `11` installed.

> **Oracle JDK License**
>
> Oracle has updated the license terms on which Oracle JDK is offered. The new Oracle Technology Network License Agreement for Oracle Java SE is substantially different from the licenses under which previous versions of the JDK were offered. **Please review** the new terms carefully before downloading and using this product.

The key part of the license is as follows:

> You may not: use the Programs for any data processing or any commercial, production, or internal business purposes other than developing, testing, prototyping, and demonstrating your Application.

Do **NOT** download or use the Oracle JDK unless you intend to pay for it. **Use an OpenJDK build instead.**

### 3.2 Servlet Containers

There is no officially supported servlet container for CAS, but [Apache Tomcat](http://tomcat.apache.org/) is the most commonly used. Support for a particular servlet container depends on the expertise of community members.

See [this guide](https://apereo.github.io/cas/6.3.x/installation/Configuring-Servlet-Container.html) for more info.

### 3.3 Build Tools

WAR overlays are [provided](https://apereo.github.io/cas/6.3.x/installation/WAR-Overlay-Installation.html) to allow for a straightforward and flexible deployment solution. While it admittedly requires a high up-front cost in learning, it reaps numerous benefits in the long run.

> **Do Less**
>
> You **DO NOT** need to have Gradle installed prior to the installation. It is provided to you automatically.

### 3.3 Git (Optional)

While not strictly a requirement, it’s HIGHLY recommended that you have [Git](https://git-scm.com/downloads) installed for your CAS deployment, and manage all CAS artifacts, configuration files, build scripts and setting inside a source control repository.

### 3.4 OS

No particular preference on the operating system, though Linux-based installs are typically more common than Windows.

### 3.5 Internet Connectivity

Internet connectivity is generally required for the build phase of any Maven/Gradle based project, including the recommended WAR overlays used to install CAS. The build process resolves dependencies by searching online repositories containing artifacts (jar files in most cases) that are downloaded and installed locally.

### 3.6 Hardware

Anecdotal community evidence seems to suggest that CAS deployments would perform well on a dual-core 3.00Ghz processor with 8GB of memory, at a minimum. Enough disk space (preferably SSD) is also needed to house CAS-generated logs, if logs are kept on the server itself.

Remember that the above requirements are *suggestions*. You may get by perfectly fine with more or less, depending on your deployment and request volume. Start with the bare minimum and be prepared to adjust and strengthen capacity on demand if needed.

## 4. WAR Overlay Installation

CAS installation is a fundamentally source-oriented process, and we recommend a WAR overlay (1) project to organize customizations such as component configuration and UI design. The output of a WAR overlay build is a `cas.war` file that can be deployed to a servlet container like [Apache Tomcat](https://apereo.github.io/cas/6.3.x/installation/Configuring-Servlet-Container.html).

### 4.1 Requirements

[See this guide](https://apereo.github.io/cas/6.3.x/planning/Installation-Requirements.html) to learn more.

### 4.2 What is a WAR Overlay?

Overlays are a strategy to combat repetitive code and/or resources. Rather than downloading the CAS codebase and building from source, overlays allow you to download a pre-built vanilla CAS web application server provided by the project itself and override/insert specific behavior into it. At build time, the build installation process will attempt to download the provided binary artifact first. Then the tool will locate your configuration files and settings made available inside the same project directory and will merge those into the downloaded artifact in order to produce one wholesome archive (i.e. `cas.war`) . Overridden artifacts may include resources, java classes, images, CSS and javascript files. In order for the merge process to successfully execute, the location and names of the overridden artifacts locally must **EXACTLY** match that of those provided by the project inside the originally downloaded archive. Java code in the overlay project’s `src/main/java` folder and resources in `src/main/resources` will end up in the `WEB-INF\classes` folder of cas.war and they will be loaded by the classloader instead of resources with the same names in jar files inside `WEB-INF\lib`.

It goes without saying that while up-front ramp-up time could be slightly complicated, there are significant advantages to this approach:

1. There is no need to download/build from the source.
2. Upgrades are tremendously easier in most cases by simply adjusting the build script to download the newer CAS release.
3. Rather than hosting the entire software source code, as the deployer you **ONLY** keep your own local customizations which makes change tracking much easier.
4. Tracking changes inside a source control repository is very lightweight, again simply because only relevant changes (and not the entire software) is managed.

### 4.3 Managing Overlays

Most if not all aspects of CAS can be controlled by adding, removing, or modifying files in the overlay; it’s also possible and indeed common to customize the behavior of CAS by adding third-party components that implement CAS APIs as Java source files or dependency references.

The process of working with an overlay can be summarized in the following steps:

- Start with and build the provided basic vanilla build/deployment.
- Identify the artifacts from the produced build that need changes. These artifacts are generally produced by the build in the `build` directory for Gradle. Use the gradle `explodeWar` task.
- Copy the identified artifacts from the identified above directories over to the`src/main/resources`directory.
  1. Create the `src/main/resources` directories, if they don’t already exist.
  2. Copied paths and file names **MUST EXACTLY MATCH** their build counterparts, or the change won’t take effect. See the table below to understand how to map folders and files from the build to `src`.
- After changes, rebuild and repeat the process as many times as possible.
- Double check your changes inside the built binary artifact to make sure the overlay process is working.

> **Be Exact**
>
> Do NOT copy everything produced by the build. Attempt to keep changes and customizations to a minimum and only grab what you actually need. Make sure the deployment environment is kept clean and precise, or you incur the risk of terrible upgrade issues and painful headaches.

### 4.4 CAS WAR Overlays

CAS WAR overlay projects described below are provided for reference and study.

### 4.5 CAS Overlay Initializr

Apereo CAS Initializr is a relatively new addition to the Apereo CAS ecosystem that allows you as the deployer to generate CAS WAR Overlay projects on the fly with just what you need to start quickly.

To learn more about the initializr, please [review this guide](https://apereo.github.io/cas/6.3.x/installation/WAR-Overlay-Initializr.html).

### 4.6 CAS Overlay Template

You can download or clone the repositories below to get started with a CAS overlay template.

> **Review Branch!**
>
> The below repositories point you towards their `master` branch. You should always make sure the branch you are on matches the version of CAS you wish to configure and deploy. The `master` branch typically points to the latest stable release of the CAS server. Check the build configuration and if inappropriate, use `git branch -a` to see available branches, and then `git checkout [branch-name]` to switch if necessary.

| Project                                                      | Build Directory           | Source Directory     |
| ------------------------------------------------------------ | ------------------------- | -------------------- |
| [CAS WAR Overlay](https://github.com/apereo/cas-overlay-template) | `cas/build/cas-resources` | `src/main/resources` |

The `cas/build/cas-resources` files are unzipped from `cas.war!WEB-INF\lib\cas-server-webapp-resources-<version>.jar` via `gradle explodeWar` in the overlay.

To construct the overlay project, you need to copy directories and files *that you need to customize* in the build directory over to the source directory.

The WAR overlay also provides additional tasks to explode the binary artifact first before re-assembling it again. You may need to do that step manually yourself to learn what files/directories need to be copied over to the source directory.

Note: Do **NOT** ever make changes in the above-noted build directory. The changeset will be cleaned out and set back to defaults every time you do a build. Put overlaid components inside the source directory and/or other instructed locations to avoid surprises.

### 4.7 CAS Configuration Server Overlay

See this [Maven WAR overlay](https://github.com/apereo/cas-configserver-overlay) for more details.

To learn more about the configuration server, please [review this guide](https://apereo.github.io/cas/6.3.x/configuration/Configuration-Server-Management.html).

### 4.8 Dockerized Deployment

See [this guide](https://apereo.github.io/cas/6.3.x/installation/Docker-Installation.html) for more info.

### 4.9 Servlet Container

CAS can be deployed to a number of servlet containers. See [this guide](https://apereo.github.io/cas/6.3.x/installation/Configuring-Servlet-Container.html) for more info.

### 4.10 Custom and Third-Party Source

It is common to customize or extend the functionality of CAS by developing Java components that implement CAS APIs or to include third-party source by dependency references. Including third-party source is trivial; simply include the relevant dependency in the overlay `build.gradle` file.

> **Stop Coding**
>
> Overlaying or modifying CAS internal components and classes, *unless ABSOLUTELY required*, should be a last resort and is generally considered a misguided malpractice. Where possible, avoid making custom changes to carry the maintenance burden solely on your own. Avoid carrying . You will risk the stability and security of your deployment. If the enhancement case is attractive or modest, contribute back to the project. Stop writing code, or rite it where it belongs.

In order to include custom Java source, it should be included under a `src/main/java` directory in the overlay project source tree.

```shell
├── src
│   ├── main
│   │   ├── java
│   │   │   └── edu
│   │   │       └── sso
│   │   │           └── middleware
│   │   │               └── cas
│   │   │                   ├── audit
│   │   │                   │   ├── CompactSlf4jAuditTrailManager.java
│   │   │                   ├── authentication
│   │   │                   │   └── principal
│   │   │                   │       └── UsernamePasswordCredentialsToPrincipalResolver.java
│   │   │                   ├── services
│   │   │                   │   └── JsonServiceRegistryDao.java
│   │   │                   ├── util
│   │   │                   │   └── X509Helper.java
│   │   │                   └── web
│   │   │                       ├── HelpController.java
│   │   │                       ├── flow
│   │   │                       │   ├── AbstractForgottenCredentialAction.java
│   │   │                       └── util
│   │   │                           ├── ProtocolParameterAuthority.java
```

### 4.11 Dependency Management

Each release of CAS provides a curated list of dependencies it supports. In practice, you do not need to provide a version for any of these dependencies in your build configuration as the CAS distribution is managing that for you. When you upgrade CAS itself, these dependencies will be upgraded as well in a consistent way.

The curated list of dependencies contains a refined list of third party libraries. The list is available as a standard Bills of Materials (BOM). Not everyone likes inheriting from the BOM. You may have your own corporate standard parent that you need to use, or you may just prefer to explicitly declare all your configuration.

To take advantage of the CAS BOM at `org.apereo.cas:cas-server-support-bom`, via Gradle, please [use this guide](https://plugins.gradle.org/plugin/io.spring.dependency-management) and configure the Gradle build accordingly.

(1) [WAR Overlays](http://maven.apache.org/plugins/maven-war-plugin/overlays.html)

## 5. Blog

> [Apereo Community Blog](https://apereo.github.io/)

