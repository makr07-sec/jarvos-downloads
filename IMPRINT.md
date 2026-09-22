# Imprint / Impressum

> **Not legal advice.** This is the operator-identity disclosure for the JarvOS download/landing
> page, drafted at statute level against Swiss UWG and the EU e-Commerce Directive. The entity
> details below are final; the framing questions (commercial vs. non-commercial) are noted at the
> end.

## Where this goes

**Not in the app.** JarvOS ships with an in-app privacy statement (Settings → About) and
`PRIVACY.md`/`docs/PRIVACY.md` covering the zero-egress data model — that's a *product*
disclosure. This imprint is an *operator-identity* disclosure and belongs in the **footer
of the public download/landing page** (`site/index.html`), not inside the desktop app itself. The app has no accounts, no server, and no
persistent UI surface where a statutory imprint would normally live — the download site is
the only public-facing surface subject to this duty.

The landing page lives on `helveticlabs.org`, which already carries the canonical imprint for
every product of the same operator: <https://helveticlabs.org/imprint>. The block below is the
JarvOS-specific copy of it; keep the two in sync.

## Why this applies

- **Switzerland (UWG Art. 3 para. 1 lit. s):** operators of websites offering goods or
  services must make identifiable — name/legal form, address, and an email or other means
  of prompt contact — reachable without disproportionate effort. The operator runs other
  commercial products under the same trade name, so the duty is treated as triggered rather
  than argued about.
- **Switzerland (DSG):** if the landing page processes any personal data (contact form,
  analytics, cookies), the controller identity + contact must be disclosed — the block below
  covers it, plus a data-protection contact line.
- **EU (e-Commerce Directive Art. 5 / member-state transposition, e.g. § 5 DDG in Germany,
  § 5 ECG / § 25 MedienG in Austria):** applies to any publicly accessible site with commercial
  character, including free apps distributed for reputation/adoption purposes.
- **EU (GDPR Art. 13):** if the page collects personal data (even a mailing-list field, or
  hosting analytics), the controller-identity block below satisfies the "identity and contact
  details of the controller" requirement.

## Block to publish

```
Imprint / Impressum

Information pursuant to Art. 3 lit. s UWG (CH), § 5 DDG (DE),
§ 5 ECG / § 25 MedienG (AT).

JarvOS is operated by HelveticLabs, which also operates our other products.
The operator details below apply to this service.

Operator:
Marvin Krieg
HelveticLabs (sole proprietorship)
c/o SwissMailBox 170
12 Rue Le Corbusier
1208 Genève
Switzerland

Contact: contact@helveticlabs.org

Commercial register: not registered.
VAT identification number: not VAT-registered.

Responsible for content: Marvin Krieg

Other products by the same operator: https://helveticlabs.org/imprint

Consumer dispute resolution
The EU online dispute resolution (ODR) platform ceased operation on
20 July 2025. We are neither obliged nor willing to participate in dispute
resolution proceedings before a consumer arbitration board.

Liability for content
The contents of this site have been created with the utmost care. However,
we cannot guarantee the accuracy, completeness, or timeliness of the content.
As a service provider, we are responsible for our own content on these pages
in accordance with general laws. We are not, however, obliged to monitor
transmitted or stored third-party information or to investigate circumstances
that indicate illegal activity.

Liability for links
Our site contains links to external websites of third parties, over whose
contents we have no influence. Therefore, we cannot assume any liability for
these external contents. The respective provider or operator of the linked
pages is always responsible for their content.

Copyright
The content and works on these pages created by the site operator are subject
to copyright law. Duplication, processing, distribution, or any form of
commercialization of such material beyond the scope of copyright law requires
the prior written consent of its respective author or creator. The application
source code is released separately under its own licence and may be used on
those terms.

JarvOS itself is free, local-only software with no accounts, no telemetry, and
no data collection by the operator — see PRIVACY.md for the product's data
handling statement. This imprint concerns only the operator of this download
page, not JarvOS's runtime behavior.
```

## Remaining judgment calls

- Phone number: not published. Some German commentary reads § 5 DDG as requiring a phone
  number; the prevailing view after *Deutsche Bahn* (CJEU C-649/17) is that a prompt,
  direct electronic channel suffices. Email is that channel here.
- Commercial vs. non-commercial framing for the CH UWG threshold: published as commercial,
  which is the conservative choice and costs nothing.
- Counsel review before the landing page goes live is still worth having — this is a
  statute-level pass by an engineer, not sign-off.
