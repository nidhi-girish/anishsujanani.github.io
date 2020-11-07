Regardless of the size of an organization, a successful security program requires well rounded control implementation across multiple domains and business units. This article aims to provide context on the same.

## Topics:
- [Asset Management](#asset-management)
- [System Security](#system-security)
- [Network Security](#network-security)
- [Log Management & Monitoring](#log-management-and-monitoring)
- [Modelling & Risk Assessment](#modelling-and-risk-assessment)
- [Vulnerability Management and Patching](#vulnerability-management-and-patching)
- [Privacy](#privacy)
- [Awareness](#awareness)
- [Additional Resources](#additional-resources)
- [Beyond the Basics](#beyond-the-basics)

## Asset Management
### What?
- Corresponds to maintaining an up-to-date repository of assets within your organization. 
- Assets could include processes, infrastructure and software whose details such as versions, date of purchase, license expiry/renewal information are typically recorded.
- Grouping assets by tags such as ‘network device’, ‘operating system’, ‘machine software’, ‘user-installed software’, etc. could help in understanding allocation, placement of systems and information flow across your environment.
- The added advantage is the capability to detect the use of unauthorized or third-party assets, which directly affects your risk score.

### Why?
- In terms of audit and awareness, you should be able to produce documentation of what you own, the corresponding details and terms of ownership.
- Having such a database would help in detecting the unauthorized installation of machines, network devices and software. Typically, this would be coupled with NAC/IDS services for detection and remediation.
- Having a dashboard of assets would speed up processes such as risk assessments, data classification and compliance scoping, by showing you how information flows between your systems.
- System policies can be set in accordance with asset management solutions to whitelist or blacklist software and services on host systems, controlling the technology risk.
- This practice aids availability as it provides a record of what versions to roll-back to in case of failure during patching or upgrades.

### How?
- Relatively smaller environments might be able to get away with manually recording information, however, would run into problems in continuous monitoring of existing and growing assets.
- Open-source solutions such as osQuery are widely recommended, though it might require manual tweaking for your environment.
- There are a wide variety of solutions that have a free tier with a limited feature set, that can be licensed based on the storage space, feature-set and vendor support requirements. Commonly stated products include AssetTiger, Rumble and SnipeIT, among many others.
- Cloud service providers provide automated dashboards of the services you are leasing from them. Eg. AWS Systems Manager Inventory.

## System Security
### What?
- System security refers to the collection of processes and technologies that enable practices such as system hardening, configuration baselining, integrity monitoring and endpoint protection.

### Why?
- Host-based protection is critical within networks and systems that store sensitive data or have access to the machines that do. These controls provide preventive and detective measures against host-based compromise.
- **System-hardening:** involves the removal of unnecessary services, software and modules. This reduces the attack surface on the machine and therefore reduces the risk of a bad guy compromising it and pivoting further into your network. This is especially important on public-facing machines — for example, a public-facing web server running unnecessary SSH, Print/File Sharing services, FTP services, etc. is a recipe for disaster, more so if the services are outdated, vulnerable or misconfigured.
- **Baselining and File Integrity Monitoring:** Now that you have hardened your system, keeping only the required packages and services, it would be a good idea to record this safe state. The goal is to be able to detect changes to this safe state, which may be due to an internal process (eg. updates or new builds) or may point to malicious activity. Sensitive system configuration files such as DNS records, user listing and log/registry data should be monitored for access and modification. This practice can be coupled with Asset Management to whitelist and blacklist allowed services and software on systems.
- **Anti-Virus (AV) and Host-based Intrusion Detection Systems (HIDS):** Contrasted to traditional AV, HIDS solutions provide extensive detection capability by combining signature-based detection, anomaly-detection and behavioural pattern changes. These systems monitor processes and network activity and typically integrate with log management / SIEM solutions to provide holistic logging, correlation of events and alerting.

### How?
- **System hardening:** now that a range of benchmarking and automation solutions are available, manual hardening is a thing of the past. Ansible and Chef are arguably the most popular solutions for automating infrastructure builds. CIS also puts out comprehensive documentation on hardening practices for a variety of operating systems. Automated evaluation solutions are also available, OpenSCAP is recommended.
- **Baselining and File Integrity Monitoring:** Open source products include AIDE for Linux and the OSSEC suite. Most enterprise HIDS solutions include a FIM component.
- **HIDS solutions:** Open source solutions include OSSEC and osQuery. There are a wide variety of enterprise solutions to choose from if you need vendor implementation and support.

## Network Security
### What?
- The collection of processes and technologies that goes into securing network devices and network communications across systems. This is perhaps the most critical control as it is responsible for securing data in transit between systems and across the internet; a data path that you do not necessarily have control over.

### Why?
- Network security is a _very_ broad field usually differentiated based on the location of devices and type of networks. For example, processes that go into securing wireless networks are totally different from wired networks. Protecting assets at the perimeter of your organization (the edge between the internet and your network) is vastly different from protecting assets on the inside of your network.
- Listed below are basic measures that should be taken to secure your networks. You should definitely consult with your engineers to implement more specific controls within this domain.

### How? (The basic measures, at least)
- **Segment your networks:** Absolutely critical. Put your public-facing machines on a different network/subnet from your internal machines. This can be done in many ways depending on your set-up, but it typically comes down to implementing multiple VLANs on your switches OR by setting up different virtual subnets through your cloud service provider. Ensure that your routing tables and ACLs follow the principle of least privilege.
- **Firewall Management:** If you are running with a cloud vendor, you would not have to manage a firewall as such. Instead, you would be focused on setting up access rules to machines. Connections should not be allowed in or out of your network unless it is absolutely required. Traditional firewalls come in multiple flavours, deployed as hardware or software solutions with machine learning and API extensions built-in.
- **Network Intrusion Detection Systems (NIDS):** usually corresponds to systems that sit behind your edge routers that listen to all the traffic going within and outside of your organization. These engines are capable of picking up on patterns and anomalies within the traffic to indicate external and internal attacks. Common open-source NIDS engines include BroIDS (now renamed to Zeek), Snort and Suricata.
- **Application Protocol Security:** Ensure that applications communicating over your network are doing so via secure means. This implies services such as FTP, IMAP/SMTP, HTTP, etc. over TLS (previously SSL) channels.

## Log Management and Monitoring
### What?
- Given that you now have a considerable number of systems and software to secure, your staff cannot monitor individual systems efficiently. Coupled with the alerts from your System Security and Network Security solutions, it is necessary to aggregate these logs to a centralised system — a SIEM system (Security Information and Event Management).

### Why?
- It is simply not possible to monitor individual systems once you get up to even small-scale environments.
- Centralised SIEM systems provide visibility into the security posture of your entire environment through multiple layers of software and network stacks.
- Intelligence can be built on top of these systems through correlation of logs. For example, a log entry from HIDS and NIDS may look benign individually, but in tandem might signify a more advanced vector.
- Real-time analytics and forecasting will enable security decisions and capacity planning.
- Automated alerts can be integrated based on the frequency of events, specific patterns and anomalies to indicate a security incident.
- Centralised logging goes further than just incident analysis as it is often a key requirement in compliance programs.

### How?
- Possibly the most competitive vertical within security software, Log Management/SIEM solutions are aplenty.
- A wide array of open-source solutions are available, but usually require strong technicality and time to set up and operationalize. Common free-to-use solutions include the AlienVault SIEM, SecurityOnion and Wazuh.
- You also have enterprise-scale fully-fledged solutions, aided by extensive machine learning, threat intelligence and automation provided by companies such as IBM, Exabeam and McAfee among many others.
- Cloud Service Providers provide their own console interfaces for log analysis and security alerts, however, their scope might be limited to their own leased infrastructure alone. See, AWS CloudTrail.
- One of the most comprehensive articles I have come across in explaining what an SIEM is, is available here.

## Modelling and Risk Assessment
### What?
- Modelling might be less technical in terms of technology, but is a crucial architectural process. It usually involves a highly experienced security professional who looks at your business process as a whole, identifies the systems and technologies you are using, and all the possible ways things could go wrong. In formal terms, a ‘risk assessment’ is done, either in terms of qualitative or quantitative risk, and recommendations are made to reduce said risk.

### Why?
- This might seem like an obvious process to implement, however, its benefits are usually understated. Without thorough threat modelling and assessment, there is no plan. For example, analysing your software stack which might consist of message queues and load-balanced web servers ( quite a generic set-up for web applications ) — Web-app Firewalls, DoS/DDoS protection, Connection Throttling and a whole bunch of other technical controls come into the picture. These processes are highly situation-specific and require expertise within specific domains to recommend and implement.

### How?
- SMBs usually do not have on-prem security folk. You could always reach out to organizations that carry out risk assessments and negotiate your scope and terms. Threat Modeling: Designing for Security — Adam Shostack explains the frameworks behind these processes brilliantly.


## Vulnerability Management and Patching
### What?
- **Vulnerability Management** is the process of scanning a system and the services running on it to check for known misconfigurations and common vulnerabilities.
- **Patch Management** is more of a process. The idea is to come up with a mechanism for periodically checking for updates, backing up current systems, applying updates and testing for stability.

### Why?
- **Vulnerability scanners** may provide outsider and insider insight into the security configuration of your systems. Such scanners are often available in 2 flavours — Non-credentialed scans, which emulate network scanning and service testing as an outsider system and credentialed scans, which emulate a scan from the system onto itself.
- **Patch management** as a process is essential to a successful program. Patches released by vendors usually address security, performance and configuration issues on a periodic basis and should be applied to keep your systems up to date.

### How?
- Tenable Nessus and the open-source OpenVAS are arguably the most well-known scanners out there. Personally, I have used the free distribution of Nessus for periodically scanning my home network and have been more than happy with it. Both solutions offer a wide variety of scanning options, credentialed and non-credentialed, with updated information stores on the latest version details and patterns to look for across products, services and operating systems. Reporting options and extensibility through the API are offered by both.
- With regards to patch management, you should be aware of possible system outages and downtimes that a patch failure could cause, for which you would need to be able to quickly roll-back or distribute workloads to a backup system.

## Privacy
### What?
- In order to grow your business reputation and maintain quality relationships with your customer base, you should be focused on securing their data, most importantly their PII: Privately Identifiable Information.
- The advent of GDPR and CCPA marks the legal requirement for companies to set up controls to protect user data. The coming of even more regional privacy laws around the world is highly likely and should be considered a priority going forward.

### Why?
- Data breaches are the bane of the enterprise. Companies have suffered significant blows to revenue and reputation due to private customer data being made public.
- This data goes beyond password leakage — and spans the domain of PII. PII includes names, addresses, phone numbers, email, blood groups, gender, birthdates, etc. A specific list can be found here.

### How?
- Large companies are having a tough time introducing these controls into their software stacks. These components were written at a time when customer privacy data was not a mainstream business requirement for success. Neither was it a legal requirement. Over the years, software evolved into highly complex systems and have formed legacy backbones. - - Patching software that runs so deep is not easy.
- The obvious solution for new software and small-scale systems is to integrate privacy controls right from the start of design and development. You may not have customers in regions currently implementing privacy laws, however, it is a good idea to build these controls into your systems early, as different regional laws will have an overlap of requirements.

## Awareness
### What?
- Despite the above control categories significantly reducing risk to your business, it might be interesting to note that your users, both employees and customers pose the largest attack surface. Awareness is a key factor in security programs.

### Why?
- The majority of incidents occur due to end-user unawareness.
- Over 90% of cyber incidents occur due to user unawareness through phishing.

### How?
- Not only is it important to safeguard user data and build systems that handle communication across the internet safely, but educating your associates is crucial.
- Technical staff need to be trained on secure development practices and building info-sec into systems right from the start.
- Even non-technical staff need to be trained on basic security practices, password policy and privacy.
- There are a range of companies that carry out these training programs. They also usually provide security incident simulation solutions within the company’s internal network to test associate awareness.

## Additional Resources
If you are a company with compliance standards to be in accordance with, you should definitely also check out the Secure Controls Framework which maps controls to multiple frameworks and makes achieving said compliance much easier.
The CIS Top 20 document is excellent in terms of splitting up control implementation and is generally regarded as a highly competent security program plan for any organization.


## Beyond the Basics
Other areas that are key to wider programs but might not be immediately suitable to SMBs include wireless security, NAC solutions, VPN concentrators, data loss prevention solutions and many others. These are usually far more involved and you may want to engage in further evaluation of your set-up.

------------------

I hope that the content in this article has helped you understand the need for these controls in your environment and that the links to resources/solutions help you achieve the implementation.
