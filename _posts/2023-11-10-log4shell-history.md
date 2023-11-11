---
layout: post
title: "Retrospective on CVE-2021-44228 (Log4Shell)"
date: 2023-11-10 17:00:00 -0500
categories:
- log4j
- java 
---
[Apache Log4j 2][log4j2] is an open source Java logging library that I've been contributing to and maintaining since 2014 as a member of the Apache Logging Services Project Management Committee (aka [PMC][pmc]).
Up until late 2021, Log4j 2 was a fairly low profile project in contrast with its widespread usage across the industry.
On 2021-11-24, the Logging PMC received a vulnerability report that would put Log4j 2 on the global stage just a couple of weeks later.
That email was just one link in the chain of events leading to Log4Shell, one of the most impactful security vulnerabilities in over a decade.
In this post, we'll reflect on the history leading up to [CVE-2021-44228][cve] and [CVE-2021-45046][cve2], how the [Apache Way][way] of open source software development prevailed in rapidly responding to these vulnerabilities, and what else we're doing in Log4j 2 to help prevent any similar issue from happening in the future.
Most of the information here was documented over a couple of weeks after the incident, and as we come close to the second anniversary of Log4Shell, I wanted to publish this publicly with some updates.

## Background

The Log4Shell vulnerability works through a combination of features added over time that ultimately exploits design flaws in the [Java Naming and Directory Interface][jndi] (JNDI) API—a standard component of Java—to allow an attacker to execute remote code fairly easily.
Dating back to the first release of Log4j 2 version 2.0-alpha1 while the project was still an experiment, a [commit was added][str-subst] that changed how substituting property placeholders in configuration files work to support a similar configuration feature from Log4j version 1.2.x.
Unfortunately, the way this was implemented applied property substitution to both configuration files and log messages when it was only intended to apply to configuration data.
As this functionality was introduced during the experimental period of development, there aren't any Jira issues to refer to for further historical context.
While this functionality was not initially dangerous as the types of property lookup plugins available then were fairly benign, this was certainly unexpected behavior which wasn't discovered until long after version 2.0 was released.
The Log4j 2 team has always taken backwards compatibility seriously, and the scope of this compatibility included both the logging API and the default logging configuration behavior.
There are other configuration options we'd love to make the default behavior—particularly around performance options—but broad changes like that will have to wait for version 3.0.

After several more pre-2.0 releases, a [new feature was contributed][jndi-lookup] to support property substitution for strings from the JNDI API.
The provided use case was to support a configuration where log messages would be appended to different files depending on which web application context the logs were coming from.
Specifying a common logging config for an entire Java EE cluster was a great way to reduce toil involved in updating logging configurations across a broad deployment.
Being July 2013, the exploitability of the JNDI API was still unknown, so this feature was accepted and released as part of version 2.0-beta9.
Combined with the log message property substitution behavior, the foundation was laid for making the default logging configuration susceptible to Log4Shell.

Later that year, I discovered the Log4j 2 project while researching what logging library to use in Java applications and began contributing to the project.
In early 2014, I was invited to become a committer, and in later 2014, I was invited to join the PMC.
It was that year when we finally released version 2.0 for general availability and set the stable compatibility point for the API and default configuration.
Near the end of 2014, an [issue was filed][date-bug] that discovered the log message property substitution behavior while debugging strange errors encountered from Apache Camel.
The details of this issue weren't made apparent until 2016 when the root cause was discovered and partially fixed in version 2.7.

One year later, another [issue was filed][no-message-lookup] and implemented to add a global option to disable message property substitution.
This was released in version 2.10.0, and this flag was one of the early mitigations available for Log4Shell five years later.
The flag was initially justified for performance reasons as a quadratic lookup function was being executed in a hot loop which didn't seem to be necessary, though details of the exploitability of JNDI were already publicly known by the security community ever since a [Blackhat 2016 talk][jndi-blackhat] published details around an earlier zero-day vulnerability exploited during [Operation Pawn Storm][pawn-storm].
This zero-day was a vulnerability that was being exploited by attackers in the wild before Oracle were ever informed about the issue.

## The First Vulnerability Report

On 2021-11-24, one day before the American holiday Thanksgiving Day, the Logging PMC received an email with the first vulnerability report.
Included were exploitation details and an attached proof of concept project to demonstrate the exploit.
Over the next 24 hours, the PMC discussed the implications of the vulnerability, potential solutions to patch the issue, and further details on how the exploit worked.
On 2021-11-25, the PMC acknowledged the vulnerability with the reporter and continued internal discussions on how to address the issue over the next few days.
During that time, the original reporter followed up with confirmations that Apache Flink, Apache Druid, and Apache Struts 2 were all impacted by this vulnerability.
By the 29th of November, the PMC had settled on a root cause of the problem and solution, so a [public issue was filed][disable-message-lookups] along with a pull request which made no reference to a security vulnerability, though astute observers may have deduced the seriousness of this issue before the CVE was later published.
During the following week, we began preparing the codebase for a 2.15.0 release.

On 2021-12-05, the proposed fix for the vulnerability was merged, and version 2.15.0-rc1 was cut the following day.
While the release candidate was being reviewed by the PMC during a standard 72-hour voting period, the original reporter emailed us on the 8th to warn that they discovered public discussions among security teams on social media about a Log4j RCE vulnerability, though they weren't sure if they knew the full extent of the problem.
Given the lack of evidence for anyone having exploited the issue before they had first reported the issue back in November, this would disqualify the use of "zero-day" vulnerability, though it did lead to zero-day vulnerabilities in tons of existing software.
On the 9th, the reporter emailed us again to show that the proposed fix was insufficient, so the release candidate vote was cancelled.
Later that day, a new release candidate was cut with a more comprehensive fix for the issue along with an expedited release vote process.
During this release vote, we received more vulnerability reports related to the failed rc1 vote, though these were all the same problem identified by the original reporter earlier that day.
The vote passed, and Log4j 2.15.0 was officially released with many items in the changelog, most notably a fix for a CVE rated at 10/10 in severity.
The following day, I woke up to a bursting inbox and a handful of text messages from various journalists looking for details on the breaking story that eventually gained the moniker of Log4Shell.

## The Second Vulnerability Report

Over the next few days, we received two independent vulnerability reports that the fixes provided for CVE-2021-44228 were insufficient to protect from exploitation in certain non-default configurations.
During this time, the Logging PMC gathered in a long-running private video chat while the impact of Log4Shell began to spread throughout the world.
It was discovered that the property substitution was still being applied to a couple other Pattern Layout components which could contain attacker-controlled input.
Methods to bypass the newly introduced filtering for the JNDI lookup were demonstrated, and other places where JNDI could potentially be abused were identified and hardened.
We quickly addressed these issues and created versions 2.16.0 (for Java 8+), 2.12.2 (for Java 7), and 2.3.1 (for Java 6) to publish the fix along with a backport of the two CVEs respectively.
CVE-2021-45046 was initially published with a lower severity than it currently has at 9.0, and additional details were later reported to us independently by four separate sources that demonstrated how CVE-2021-45046 was a remote code execution vulnerability.
Various other reporters informed us on more variants of this later on, but they were all either caused by the same issue or were less severe than the RCE aspect.

Of course, this wasn't the final security vulnerability we fixed in December 2021, but these two issues were the only ones that led to serious problems like remote code execution.
Since then, an issue related to denial of service and another related to unexpected behavior when using JNDI for configuring other components were fixed in CVE-2021-45105 and CVE-2021-44832 respectively.
For more information about these CVEs, see [the Log4j security page][log4j-security].

## Future of Log4j

These security vulnerabilities raised a lot of questions and laid bare the consequences of the industry not keeping inventory of the software they use let alone the more detailed inventory of subcomponents that make up each piece of software.
The [security process established at the Apache Software Foundation][sec] worked great at delivering security updates for the public as quickly as possible, though it turned out that many of our users were unaware they were even users of Log4j 2.
When I first wrote this for a limited audience, 35% of recent downloads of Log4j 2 from Maven central were for vulnerable releases.
Nearly two years later, this has reduced to [23% of recent downloads][updates] which is an improvement.
In the Log4j project, we have a couple ideas we're working on that will help prevent similar problems in the future.

A primary cause of Log4Shell was a packaging philosophy shared by Java itself until Java 9 finally broke itself up into well-defined modules.
To simplify upstream packaging and downstream use, many optional features were included in the standard distribution.
During the early 2010s when Log4j 2 was initially developed, build tools like Apache Maven and Gradle were not as dominant while tools like Apache Ant or even Make were used with manually checked-in copies of third party library jar files.
The previous major version of Log4j was packaged as a single jar; while convenient for the even more primitive build tooling of the late '90s and early 2000s, this also had the design flaw of mixing the external and internal APIs together.
Version 2.0 was initially packaged into two main jars: the frontend `log4j-api` and the backend `log4j-core`.
The API has strong compatibility support from version to version, and the core backend contains all the implementations available as plugins for use in a logging configuration.
Many optional plugins are included here which can only be used when their required dependencies are present on the classpath.
This packaging format was chosen to maximize simplicity of deployment, though even this split-jar setup has caused its own large amounts of confusion to Java developers of all experience levels; some users are first learning what jars are while other users don't even know the difference between `log4j-api` (the frontend) and `log4j-core` (the backend).

For version 3.0, [we are planning][modularization] to break up `log4j-core` into several additional modules while leaving a slim, secure kernel.
We've published one alpha release of 3.0 so far, and we want to make a beta release before 3.0.0.
While the updated `log4j-core` will still contain many common plugins, this reduces the required module dependencies to a base requirement of `java.base`.
Users will have to opt in to fancier features that rely on dependencies outside the `java.base` module provided by Java 11 as those plugins will be packaged in their own modules.
We are also adding basic feature flags to select plugins so that they can require explicit opt-in even when the module is present at runtime to further protect from abuse of potentially insecure plugins.

## Conclusion

In the time since these issues were fixed, the project has received various levels of sponsorship and donations.
This has helped a lot as we've been able to dedicate more time to working on improvements to our release process, security metadata, code and test quality, and several other aspects of the project.
As world governments debate how to regulate software engineering, we continue to lead by example both in the Logging Services PMC and the Apache Software Foundation more broadly.

Log4j 2 continues to improve over time with a healthy community and a fresh wave of widespread interest in reviewing the project for any sign of potential security issues.
As always, we encourage others to contribute to the project and other projects at Apache; community is the power behind the Apache Way.
It's important that everyone using the project make sure they're using the latest version.
Users still on Log4j 1 should upgrade to Log4j 2 or use our [Log4j 1 compatibility features][migration] which are supported by the PMC.

[log4j2]: https://logging.apache.org/log4j/2.x/
[way]: https://www.apache.org/theapacheway/
[cve]: https://logging.apache.org/log4j/2.x/security.html#CVE-2021-44228
[cve2]: https://logging.apache.org/log4j/2.x/security.html#CVE-2021-45046
[str-subst]: https://s.apache.org/log4shell-origins-1
[jndi-lookup]: https://s.apache.org/log4shell-origins-2
[date-bug]: https://s.apache.org/log4shell-origins-3
[no-message-lookup]: https://s.apache.org/log4shell-origins-4
[jndi-blackhat]: https://www.blackhat.com/docs/us-16/materials/us-16-Munoz-A-Journey-From-JNDI-LDAP-Manipulation-To-RCE.pdf
[pawn-storm]: https://www.trendmicro.com/vinfo/us/security/news/cyber-attacks/operation-pawn-storm-fast-facts
[disable-message-lookups]: https://s.apache.org/log4shell-origins-5
[sec]: https://www.apache.org/security/committers.html
[updates]: https://www.sonatype.com/resources/log4j-vulnerability-resource-center
[modularization]: https://issues.apache.org/jira/browse/LOG4J2-2226
[migration]: https://logging.apache.org/log4j/2.x/manual/migration.html
[pmc]: https://www.apache.org/dev/pmc.html
[jndi]: https://docs.oracle.com/javase/jndi/tutorial/getStarted/overview/index.html
[log4j-security]: https://logging.apache.org/log4j/2.x/security.html
