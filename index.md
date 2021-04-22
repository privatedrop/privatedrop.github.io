---
title: Breaking and Fixing Apple AirDrop
---

## Introduction

Apple AirDrop is a file-sharing service that allows users to send photos and other media over a direct Wi-Fi connection from one Apple device to another. As people typically want to share sensitive data exclusively with people they know, AirDrop only shows receiver devices from address book contacts by default. To determine whether the other party is a contact, AirDrop uses a mutual authentication mechanism that compares a user's phone number and email address with entries in the other user's address book.

## The Problem: Phone Number and Email Address Leakage

We discovered *two* severe privacy leaks in this authentication mechanism. In particular, we showed that it is possible to learn the phone numbers and email addresses of AirDrop users -- even as a complete stranger. An attacker just requires a Wi-Fi-capable device and physical proximity to a target.

The discovered problems are rooted in Apple's use of hash functions for "obfuscating" the exchanged contact identifiers, i.e., phone numbers and email addresses, during the discovery process. It is well-known in industry and academia that [hashing fails to provide privacy-preserving contact discovery](https://contact-discovery.github.io) since hash values of phone numbers can be quickly reversed using simple techniques such as brute-force attacks or database look-ups.

### Vulnerability #1: Sender Leakage

During the AirDrop authentication handshake, the sender always discloses their own (hashed) contact identifiers as part of an initial *discover* message. A malicious receiver can therefore learn all (hashed) contact identifiers of the sender without requiring any prior knowledge of their target. To obtain these identifiers, an attacker simply needs to wait (e.g., at a public hot spot) until a target device scans for AirDrop receivers, i.e., the user opens the sharing pane.

After collecting the (hashed) contact identifiers, the attacker can recover phone numbers and email addresses offline. As shown in [prior work](https://encrypto.de/papers/HWSDS21.pdf), recovering phone numbers is possible in the order of milliseconds. Recovering email addresses is less trivial but possible via dictionary attacks that check common email formats such as first.lastname@{gmail.com,yahoo.com,...}. Alternatively, an attacker could utilize [data breaches](https://www.businessinsider.com/stolen-data-of-533-million-facebook-users-leaked-online-2021-4) or use an [online lookup service for hashed email addresses](https://web.archive.org/web/20191211152224/https://datafinder.com/products/email-recovery).

This attack was also independently discovered and published by the [Apple Bleee](https://hexway.io/research/apple-bleee/) project in July 2019, shortly after our initial responsible disclosure to Apple in May 2019.

### Vulnerability #2: Receiver Leakage

AirDrop receivers present their (hashed) contact identifiers in response to the discover message if they know *any* of the sender's contact identifiers (e.g., if the receiver has stored the sender's email address). A malicious sender can thus learn *all* contact identifiers (including the receiver's phone number) without requiring any prior knowledge of the receiver -- if the receiver knows the sender.

Importantly, the malicious sender does not have to know the receiver: A popular person within a certain context (e.g., the manager of a company) can exploit this design flaw to learn all (private) contact identifiers of other people who have the popular person in their address book (e.g., employees of the company).

### Proof-of-Concept Attacks

We demonstrate attacks exploiting the two vulnerabilities with a proof-of-concept implementation that is publicly available on [GitHub](https://github.com/seemoo-lab/opendrop/blob/poc-phonenumber-leak/README.PoC.md). It combines the efforts of [OpenDrop](https://github.com/seemoo-lab/opendrop), an open-source AirDrop implementation, with [RainbowPhones](https://github.com/contact-discovery/rt_phone_numbers), an open-source hash cracking utility that is optimized for non-uniform input domains such as mobile phone numbers.

## Our Solution: PrivateDrop

We developed a solution named *PrivateDrop* to replace the flawed original AirDrop design. PrivateDrop is based on optimized cryptographic private set intersection protocols that can securely perform the contact discovery process between two users without exchanging vulnerable hash values. Our prototype implementation of PrivateDrop on iOS/macOS shows that our privacy-friendly mutual authentication approach is efficient enough to preserve AirDrop's exemplary user experience with an authentication delay well below one second.

The implementation of PrivateDrop is publicly available on [GitHub](https://github.com/seemoo-lab/privatedrop).

## Responsible Disclosure

We informed Apple about the privacy issues in May 2019 via responsible disclosure and shared our PrivateDrop solution in October 2020. As of April 20, 2021, Apple has not indicated that they are working on a solution.

This means **Apple users are still vulnerable to the outlined privacy attacks.** They can only protect themselves by disabling AirDrop discovery in the system settings and by refraining from opening the sharing pane.

## Publication

 * [HHSSW21] **_PrivateDrop: Practical Privacy-Preserving Authentication for Apple AirDrop_** by [Alexander Heinrich](https://www.informatik.tu-darmstadt.de/seemoo/team_seemoo/alexander_heinrich/alexander_heinrich.en.jsp), [Matthias Hollick](https://www.informatik.tu-darmstadt.de/seemoo/team_seemoo/matthias_hollick/index.en.jsp), [Thomas Schneider](https://encrypto.de/schneider), [Milan Stute](https://www.informatik.tu-darmstadt.de/seemoo/team_seemoo/milan_stute/milan_stute.en.jsp), and [Christian Weinert](https://encrypto.de/weinert) in [30th USENIX Security Symposium (USENIX Security'21)](https://www.usenix.org/conference/usenixsecurity21). Paper available as **[pre-print](https://www.usenix.org/system/files/sec21fall-heinrich.pdf)**. Implementation available on **[GitHub](https://github.com/seemoo-lab/privatedrop)**. Proof-of-concept attacks available on **[GitHub](https://github.com/seemoo-lab/opendrop/blob/poc-phonenumber-leak/README.PoC.md)**.
