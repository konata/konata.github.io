---
layout: post
title: 'Samsung’s ISVP: Betraying the Trust of Security Researchers'
published: true
date: 2025-05-16
---

As a security researcher passionate about improving mobile security, I’ve had two distinct experiences with Samsung’s security bug bounty program in 2024 and 2025. Unfortunately, both encounters left me frustrated due to **slow responses**, **lack of communication**, **opaque processes**, and **inconsistent reward evaluations**. This post details my journey, with the hope of shedding light on areas where Samsung’s program could improve to better support the security research community.

## 2024: LaunchAnywhere vulnerabilities and a frustrating experience

In 2024, I discovered and reported three high-severity launch anywhere vulnerabilities to Samsung’s security team. These vulnerabilities, which I believed posed significant risks, were eventually acknowledged as high-severity issues. However, the rewards $500, $300, and $300 were surprisingly low for vulnerabilities of this caliber. The entire process, from submission to reward, took nine months, with several pain points:

- Slow Response Times:

  The process dragged on for nearly nine months (2024-01-16 to 2023-09-04), with no clear timeline provided.

- Zero Communication:

  Despite reaching out via email and Samsung’s official reporting platform, I received no responses to any of my queries. It felt like shouting into a void.

> ![samsung-no-human-response-2024.png](/assets/images/samsung-isvp/samsung-no-human-response-2024.png.png)

- Opaque Standards:

  The evaluation and reward criteria were completely unclear. As a researcher, I was left in the dark, operating in what felt like a black-box system.

Frustrated by the low rewards and lack of engagement, I decided to pause my efforts with Samsung’s program. The experience was disheartening, and I shifted my focus to other platforms that offered better transparency and communication.

## 2025: A glimmer of hope with ISVP, but disappointment persists

In 2025, I came across Samsung’s [Important Scenario Vulnerability Program](https://security.samsungmobile.com/securityPostDetail.smsb/189) (ISVP), announced on their security portal (Samsung Mobile Security). The ISVP promised improvements, specifically addressing the issues I had faced in 2023. Two aspects of the program caught my attention:

- Clear Scenarios and Conditions:

  The ISVP outlined specific vulnerability scenarios and their criteria, leaving little room for ambiguity.

- Defined Reward Amounts:

  The program explicitly stated bounty amounts, suggesting that meeting the criteria would result in fair and predictable rewards.

Encouraged by these promises, I decided to give Samsung’s program another chance. I invested significant effort and successfully identified a vulnerability in the PackageInstaller component, enabling silent installation of arbitrary applications without user interaction, a serious security flaw. In my report, I clearly indicated that I was applying for the ISVP, adhering to all specified requirements.

Over the next four months, I followed up multiple times, asking for updates on the report’s status and whether it met ISVP criteria. I received no responses. Finally, just a few days ago, Samsung completed their review. To my dismay, the vulnerability was rated as Moderate (SVE-2025-0129, CVE-2025-20974), despite its potential for significant impact. The reward? A mere $500, far below the ~$50,000 specified in the ISVP rules for a local exploit enabling silent installation of arbitrary applications.

Here’s Samsung’s [public disclosure](https://security.samsungmobile.com/serviceWeb.smsb) and corresponding ISVP rules for reference:

> ```
> SVE-2025-0129 (CVE-2025-20974): Improper handling of insufficient permission in PackageInstallerCN
> Severity: Moderate
> Resolved version: 15.0.11.0
> Reported on: January 24, 2025
> Description: Improper handling of insufficient permission in PackageInstallerCN prior to version 15.0.11.0 allows local attacker to bypass user interaction for requested installation. The patch adds proper access control.
> ```

> **Arbitrary Application Install**
>
> | Target                        | Local      | Remote      |
> | ----------------------------- | ---------- | ----------- |
> | Application from Galaxy Store | ~ $ 30,000 | ~ $ 60,000  |
> | Arbitrary applications        | ~ $ 50,000 | ~ $ 100,000 |
>
> ※ Arbitrary application is an application from unofficial market place or attacker’s server.

When I reached out to inquire about the moderate rating and the reward decision, I was met with the same silence as before. No explanation, no dialogue, just a final decision handed down without justification.

> ![communication-via-email](/assets/images/samsung-isvp/communication-via-email.png)

> ![communication-via-portal](/assets/images/samsung-isvp/samsung-no-human-response-2025.png)

## A Pattern of Disappointment

My experiences in 2024 and 2025 reveal a troubling pattern in Samsung’s bug bounty program. Despite the introduction of the ISVP, which promised greater transparency and fairness, the core issues remain:

- Lack of Communication:

  Samsung’s failure to respond to researcher inquiries creates a frustrating and isolating experience. Whether through email or their reporting platform, researchers are left without guidance or feedback.

- Inconsistent Evaluations:

  The downgrade of a serious vulnerability to “Moderate” and the low reward amount suggest that Samsung’s evaluation process is still opaque. The ISVP’s promise of clear criteria and rewards have not been fulfilled in practice.

- Slow Processes:

  Waiting six to nine months for a resolution is unacceptable, especially when researchers invest significant time and effort into identifying vulnerabilities that protect Samsung’s users.

These issues not only discourage researchers but also undermine the security of Samsung’s ecosystem. A bug bounty program should be a partnership, where researchers and vendors collaborate to enhance security. Instead, Samsung’s approach feels one-sided, leaving researchers undervalued and unheard.

## Final Words

As a security researcher, I’m driven by a desire to make technology safer for everyone. My experiences with Samsung’s bug bounty program, however, have been profoundly disappointing. The lack of communication, opaque processes, and unfulfilled promises of the ISVP have eroded my trust in the program. I hope that by sharing my story, I can spark a conversation about the importance of treating researchers as valued partners in the security ecosystem.

To my fellow researchers: proceed with caution when engaging with Samsung’s program. To Samsung: please listen to the community and take meaningful steps to improve. The security of your devices and the trust of the research community depends on it.
