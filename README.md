# Get-RBCD-Threaded
Tool to discover Resource-Based Constrained Delegation attack paths in Active Directory Environments

Based almost entirely on wonderful blog posts "Wagging the Dog: Abusing Resource-Based Constrained Delegation to Attack Active Directory" by [Elad Shamir](https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html) and "A Case Study in Wagging the Dog: Computer Takeover" by [harmj0y](https://www.harmj0y.net/blog/redteaming/another-word-on-delegation/). Read these two blog posts if you actually want to understand what is going on here. I honestly only half understand it all myself (and that's being generous).

I don't know how to C# well so I figured out how to communicate with a domain in C# by reading through the source code of [SharpSploit](https://github.com/cobbr/SharpSploit) and [SharpView](https://github.com/tevora-threat/SharpView).

## How it works
Get-RBCD-Thread will query all Active Directory users, groups (minus privileged groups like "Domain Admins" and "BUILTIN\Administrators"), and computer objects in your current domain and compile a list of their SIDs. Get-RBCD-Threaded will then query AD for all DACLs on the computer objects in the domain. Each ACE in the DACLs will be checked to see if one of the user/group/computer SIDS has either "GenericAll", "GenericWrite", or "WriteOwner" privileges on the computer object. If it does, then, well, you my friend are on your way to a Resource-Based Constrained Delegation attack!

## Usage
Compile in Visual Studio. This uses Parallel.ForEach to spead up searching through the DACL object, so .NET v4 is minimum required.

Tested in an environment with 20k+ uses, groups, and computers (over 60k total objects). Get-RBCD-Thread took ~60 seconds to complete. By comparison, my hacked together [PowerView](https://github.com/PowerShellMafia/PowerSploit/tree/dev) commands in this [gist](https://gist.github.com/FatRodzianko/e4cf3efc68a700dca7cedbfd5c05c99f) to perform a similar search ran for several hours and never completed.

Currently only works if you are authenticated to a domain, and only queries your current domain. I may add functionality later to allow for you to specify the domain to query and to pass your own domain credential.

## Detections
This tool does nothing more than query Active Directory using LDAP queries, which may not be easy to detect. Netflow could possibly be used to detect large numbers of LDAP queries / traffic to one system.

The other possible way to detect this is through honeypot accounts. The idea would be to create a computer object that some user / group has write privileges to. The RBCD attack relies on modifying a computer object and then delegating kerberos tickets to it. The possible points of detection for the honeypot computer object could be:
1. Monitor modifications to the honeypot computer object, specifically to the "msds-allowedtoactonbehalfofotheridentity" property
1. Monitor for kerberos tickets requested for services on the honeypot computer object, specifically any kerberos tickets for administrator users