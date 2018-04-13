---
title: "Certificate spelunking for fun and profit"
layout: "post"
---

Google and others contribute TLS certificates
to Certificate Transparency logs
as they crawl the web,
but there is less systematic effort
devoted to discovering certificates
from secure, non-HTTPS protocols.
I searched the
[Rapid7 "More SSL" scan log](https://github.com/rapid7/sonar/wiki/More-SSL-Certificates)
for novel certificates and contributed them to CT logs.

[Certificate transparency](https://www.certificate-transparency.org/)
is an effort to publicly log
all publicly trusted TLS certificates.
Publishing certificates has
[a number of benefits](https://www.certificate-transparency.org/what-is-ct),
including allowing site owners to detect
fraudulent certificates issued in their name,
and allowing more comprehensive oversight
of CA operations.

Google Chrome will begin
[requiring CT records](https://groups.google.com/a/chromium.org/d/msg/ct-policy/sz_3W_xKBNY/6jq2ghJXBAAJ)
to trust newly-issued certificates, beginning in the next month,
but historical certificates may not have been logged,
and site operators may prefer not to log certificates
which are not intended for web browsers.

TLS certificates protecting HTTPS connections
are collected by Google's spider
and added to CT logs.
I hypothesized that TLS-protected, non-HTTPS services
might be an additional source of novel certificates for logging,
and could reveal unidentified misissuance.

Rapid7's [Project Sonar](https://github.com/rapid7/sonar/wiki)
scans selected services on the entire IPv4 internet at regular intervals
and produces reports containing,
among other things,
any TLS certificates provided by the scanned hosts.
This work is based on
[the `moressl` dataset](https://github.com/rapid7/sonar/wiki/More-SSL-Certificates),
which contains certificates discovered
while crawling services including SMTP, IMAP, and FTP
in both TLS-wrapped and STARTSSL modes.

## Discovered certificates

This data set did indeed reveal many certificates
which had not been previously logged.

* The `moressl` dataset contains 10,341,552 certificates as of Mar 22, 2018
* 2,172,408 certificates (21%) had valid signatures
  and chained to roots included in the Mozilla trust store
* 56,249 trusted certificates (3.6% of trusted certificates) were novel to
  [crt.sh](https://crt.sh/), which has views of many public CT logs
* 22,893 trusted certificates (40% of novel certificates) were unexpired
* 2,279 unexpired, trusted certificates were novel but contained a SCT proof,
  meaning that a precertificate had already been disclosed to CT,
  but the final, issued certificate had not been logged

This validates the idea that non-HTTPS TLS services
are an important source of undisclosed certificates,
and confirms the novelty of this analysis. :)

Although CT signature embedding will become more common,
which requires that CAs submit precertificates to logs
before certificate issuance,
some CAs do not have a practice of subsequently logging
the final issued certificate
containing the embedded CT signature proofs.
Although the precertificates are sufficient
to identify most cases of misissuance,
understanding which proofs
are actually embedded in certificates
helps assess the impact of decertifying logs,
so collecting these issued certificates is still worthwhile
even in the presence of a logged precertificate.

## cablint results

Running [cablint](https://github.com/awslabs/certlint),
which audits certificates to the CA/Browser Forum Baseline Requirements,
over the unexpired certificates
reveals a variety of badly encoded or otherwise non-compliant certificates,
presented here in approximate order of interest.

### Noncompliance possibly revealed by this work

These certificates were unrevoked at the time of discovery.
I believe these issues with these issuers have not been previously discussed,
or were considered resolved,
or minimally that they were the only results I saw for these CAs on
the [misissued.com cablint page](https://misissued.com/cablint/).
Please let me know if you think I shouldn't take credit for them. :)

I have _not_ applied cablint to all certificates in CT;
it is completely possible that other certificates with these problems,
from these issuers or others, have already been publicly disclosed.
But these were the ones that you couldn't have seen before yesterday!

| Issuer | Last issued | Certificate | Details |
|--------|------|-------------|---------|
| GoDaddy | 2013 | [1](https://crt.sh/?id=370273130&opt=cablint,ocsp) | &ge; 60 month validity period |
| Digicert | 2016 | [1](https://crt.sh/?q=a44620f1c703393ddd6c90db12a47e7b2074fc17aa34ca10c864363011986ff3&opt=ocsp,cablint) | IP address encoded as a DNS SAN |
| ABB (Digicert) | 2016 | [1](https://crt.sh/?id=370240502&opt=ocsp,cablint) | CN not in SAN * |
| NetLock | 2017 | [1](https://crt.sh/?id=370179087&opt=ocsp,cablint), [2](https://crt.sh/?id=370239913), [3](https://crt.sh/?id=370210461&opt=ocsp,cablint) | Invalid EKU (KeyAgreement for RSA key) |
| KPN (Logius/PKIoverheid) | 2016 | [1](https://crt.sh/?id=370246293&opt=ocsp,cablint), [2](https://crt.sh/?id=370245612&opt=ocsp,cablint), [3](https://crt.sh/?id=370179725&opt=ocsp,cablint) | Invalid EKU (KeyAgreement for RSA key) |
| ECCE 001 (Digicert) | 2018  | [1](https://crt.sh/?id=370256565&opt=ocsp,cablint), [2](https://crt.sh/?id=370210890&opt=ocsp,cablint) | Invalid string encoding  |
| Tenera (Digicert) | 2017 | [1](https://crt.sh/?id=370246110&opt=ocsp,cablint), [2](https://crt.sh/?id=370184296&opt=cablint&opt=ocsp,cablint) | Invalid string lengths |
| Entrust | 2015  | [1](https://crt.sh/?id=370213436&opt=ocsp,cablint) | Invalid string encoding |


\* A [certificate](https://crt.sh/?id=16963475&opt=ocsp,cablint) with the same problem from the same issuer was revoked after it was disclosed [last August](https://groups.google.com/forum/#!msg/mozilla.dev.security.policy/K3sk5ZMv2DE/4oVzlN1xBgAJ).

### Noncompliance already visible from previously logged certificates

* ~~The [HydrantID SSL ICA G2](https://crt.sh/?caid=1483&opt=cablint,zlint)
  CA is trusted by Mozilla (via QuoVadis) for TLS authentication, but issues
  certs intended for IPSEC and which lack serverAuth and clientAuth EKU values,
  which are not BR-compliant (7.1.2.3.f).~~ These certificates are compliant;
  zlint was [reporting them incorrectly](https://github.com/zmap/zlint/pull/218).
  I regret the error.
* Digicert has issued certificates with [underscores in DNS names](https://groups.google.com/d/msg/mozilla.dev.security.policy/Cyyyjdf_t2Q/rBRBrLYLEQAJ)
* Microsoft's CA software inserts trailing NULs in the CPSuri field
  and fails to flag the KeyUsage extension as critical.

### Otherwise notable

* Comodo issued certificates in [2017](https://crt.sh/?id=370265846)
  and [2014](https://crt.sh/?id=370172550&opt=ocsp,cablint)
  where the CN is a U-label and only the matching A-label is present as a SAN;
  this situation, but not this certificate, was
  [discussed](https://groups.google.com/d/msg/mozilla.dev.security.policy/K3sk5ZMv2DE/4oVzlN1xBgAJ)
  on m.d.s.p
* Thawte very recently revoked two certificates containing only metadata in the
  subject OU identifier; [1](https://crt.sh/?id=370208210&opt=ocsp,cablint), [2](https://crt.sh/?id=370235442&opt=ocsp,cablint)

## Discussion

This work shows that non-HTTPS servers present a meaningful volume of
undisclosed browser-trusted TLS certificates, and that these certificates
contain interesting examples of BR violations.

It is not obvious to me that the rate or distribution of problems
in this data set is different
than they would be
from a random sample
of certificates already disclosed to CT,
which describes an interesting follow-on experiment.
The [misissued.com cablint page](https://misissued.com/cablint/)
monitors recent, but not historical, disclosures, and unknown horrors
may lurk in CT's merkled depths. _Cthulhu fhtagn_!

Happily, systematic evaluations of the quality of historical logged certificates have been undertaken recently, including work presented [a few days ago](https://www.net.in.tum.de/fileadmin/bibtex/publications/papers/pam18ctlog-slides.pdf) by Oliver Gasser at TU Munich, and an upcoming [conference paper](https://kumarde.com/papers/misissuance.pdf) by Deepak Kumar at UIUC and colleagues. Their work on logged certificates provides a useful baseline for future comparisons to discovered certificates.

The source code for retrieving and processing scan data is available at
[https://github.com/tdsmith/tattle](https://github.com/tdsmith/tattle).

Other outcomes of this work included
[a bug report against pyca/cryptography](https://github.com/pyca/cryptography/issues/4175).

## Acknowledgements

Thanks to:

* [Alex Gaynor](https://alexgaynor.net/) for helpful discussion and advice.
  I also used Alex's [ct-tools](https://github.com/alex/ct-tools) to submit
  certificates to CT logs, and his [x509-validator](https://github.com/alex/x509-validator)
  informed my chain-building strategy
* [Alex Gaynor](https://alexgaynor.net/) and [Paul Kehrer](https://twitter.com/reaperhulk)
  for their work maintaining [pyca/cryptography](https://cryptography.io/en/latest/)
* Comodo and [Rob Stradling](https://twitter.com/_robstr) for operating [crt.sh](crt.sh)
* Amazon and Peter Bowen for [cablint](https://github.com/awslabs/certlint)
* Rapid7 and their collaborators for publishing scan data

Any errors are, of course, mine alone.
