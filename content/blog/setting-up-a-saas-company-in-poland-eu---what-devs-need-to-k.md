+++
title = "Setting Up a SaaS Company in Poland (EU) - What Devs Need to Know"
date = 2026-02-09
description = "JDG vs sp. z o.o., ZUS costs, VAT OSS, GDPR, Merchant of Record vs Stripe - a practical guide for developers who want to sell SaaS from Poland."

[taxonomies]
tags = ["saas", "business", "eu", "devops"]
+++

You built the thing. The SaaS works. People are signing up for the beta. And now you're staring at a government website trying to figure out what a "jednoosobowa dzialalnosc gospodarcza" is and whether you need one before you can legally charge someone 9 euros a month.

Polish bureaucracy isn't harder than other EU countries - it's actually simpler in some ways. But the information is scattered across accountant blogs, Reddit threads in Polish, and government PDFs that haven't been updated since 2023. This post puts everything in one place. No legal jargon, no "consult your lawyer" cop-outs on every paragraph. Just the concrete decisions you need to make, the numbers behind them, and the order to do things in.

<!-- more -->

## Step Zero: Pick Your Business Structure

You have two realistic options in Poland. Ignore everything else (spolka jawna, komandytowa, akcyjna) - they're for different situations.

### JDG (Jednoosobowa Dzialalnosc Gospodarcza) - Sole Proprietorship

This is the default choice for a solo developer starting out. You register online through [CEIDG](https://www.biznes.gov.pl/en/firma/doing-business-in-poland) in about 15 minutes. No notary, no capital requirement, no registration fee. You get your NIP (tax ID) and REGON (statistical number) automatically. You can start issuing invoices the same day.

The catch: you're personally liable for everything. If your SaaS somehow causes damages and you get sued, your personal assets are on the line. For a developer selling a $20/month project management tool, this risk is mostly theoretical. For a SaaS handling financial data or health records, it's worth thinking about.

### Sp. z o.o. (Spolka z Ograniczona Odpowiedzialnoscia) - Limited Liability Company

This is the Polish equivalent of a US LLC or German GmbH. Your personal assets are separated from the company's liabilities. You need 5,000 PLN (about 1,150 EUR) in share capital, and you can register online through [S24](https://ekrs.ms.gov.pl/s24/) for about 350 PLN in court and gazette fees. The traditional notary route costs more - around 600 PLN total.

The catch: double taxation. The company pays CIT (Corporate Income Tax) on profits, then you pay 19% PIT on dividends when you take money out. More on the actual numbers below.

### The Quick Comparison

| | **JDG** | **Sp. z o.o.** |
|---|---|---|
| Setup cost | 0 PLN | ~5,350 PLN (capital + fees) |
| Setup time | Same day | 1-3 days (S24) or 1-2 weeks (notary) |
| Personal liability | Unlimited | Limited to share capital |
| ZUS contributions | Required (1,600-1,927 PLN/mo) | Not required if you're on the board |
| Tax options | Ryczalt, liniowy, skala | CIT 9% or 19% |
| Accounting cost | 200-500 PLN/mo | 500-1,500 PLN/mo |
| Raising investment | Hard | Standard |

## The Real Cost of ZUS

ZUS (Zaklad Ubezpieczen Spolecznych) is Poland's social insurance system. If you run a JDG, you pay it every month regardless of whether you made any revenue. This is the single biggest complaint from Polish developers running side businesses, and it's the main reason many switch to sp. z o.o.

Here are the 2026 numbers:

**Ulga na start (first 6 months):** You only pay health insurance - about 432 PLN/month. No social contributions. This applies automatically when you register your first-ever JDG.

**Preferencyjne ZUS (months 7-30):** Reduced social contributions based on 30% of minimum wage. Total cost: approximately 456 PLN/month for social insurance, plus health insurance. You're looking at roughly 890 PLN/month all-in.

**Full ZUS (after 30 months):** This is where it hurts. The full contribution in 2026 is about 1,927 PLN/month (approximately 445 EUR). That's your fixed cost before you've earned a single zloty. The breakdown: retirement (697 PLN), disability (286 PLN), sickness (88 PLN), accident (60 PLN), labor fund (88 PLN), health insurance (about 700 PLN depending on your tax form).

**Maly ZUS Plus:** If your revenue in the previous year was under 120,000 PLN, you can pay reduced contributions based on your actual income. This brings the total down significantly - often to the 900-1,200 PLN range. You can use it for 36 months within every 60-month period. Apply for it in January through your ZUS PUE account.

Here's why this matters for sp. z o.o.: if you're the sole shareholder AND on the management board, you don't pay ZUS at all. The company pays CIT on profits, and you pay PIT on dividends, but the monthly ZUS drain disappears. For a SaaS that's generating inconsistent revenue in the early months, avoiding 1,927 PLN/month in fixed costs can be the difference between survival and running out of runway.

## Taxes: Picking the Right Option

### JDG Tax Forms

You have three choices, and the right one depends on your revenue and expenses:

**Ryczalt (lump-sum tax):** You pay tax on revenue, not profit. No deducting expenses. For IT services (PKWiU 62.01, 62.02), the rate is 12%. For some specific activities like technical support, it can be 8.5%. This works well if your expenses are low - which they usually are for a SaaS (a VPS, a domain, maybe some API costs). Available if your annual revenue is under 2 million EUR.

**Podatek liniowy (flat tax):** 19% on net profit (revenue minus expenses). You can deduct business expenses - servers, software, hardware, conference tickets, co-working space. Health insurance under liniowy is 4.9% of income. Makes sense when your deductible expenses are significant.

**Skala podatkowa (progressive tax):** 12% on income up to 120,000 PLN, then 32% above that. Includes a tax-free amount of 30,000 PLN. Health insurance is 9% of income and NOT tax-deductible. Rarely the best choice for a profitable SaaS, but works if your income is low and you want to use the tax-free threshold.

For most solo SaaS developers, **ryczalt at 12% is the winner**. Your main "expense" is your time, which you can't deduct anyway. A SaaS making 15,000 PLN/month with 500 PLN in hosting costs saves almost nothing from expense deductions - but pays 12% on revenue instead of 19% on profit.

Quick comparison on 180,000 PLN annual revenue with 10,000 PLN in expenses:

| | **Ryczalt 12%** | **Liniowy 19%** |
|---|---|---|
| Tax base | 180,000 PLN (revenue) | 170,000 PLN (profit) |
| Income tax | 21,600 PLN | 32,300 PLN |
| Health insurance | ~8,640 PLN (deductible 50%) | ~8,330 PLN (deductible) |
| **Effective rate** | ~12% | ~19% |

### IP Box - The 5% Rate

Poland offers a preferential 5% tax rate on income derived from intellectual property you created. This includes income from licensing software - which is what a SaaS subscription technically is. Both JDG and sp. z o.o. can use it.

The requirements are strict:
- You must be developing the software yourself (R&D activity)
- You need detailed documentation: project descriptions, development logs, time tracking
- You must separate IP income from service income in your books
- The nexus ratio formula determines what portion qualifies

In practice, IP Box works best for developers who write custom software for clients and retain IP rights, or for SaaS products where you can clearly demonstrate that the subscription revenue comes from licensing your code. You'll need an accountant who understands IP Box - most generalist bookkeepers don't. Budget 800-1,500 PLN/month for an accountant who handles IP Box correctly.

One important note: there were plans to restrict IP Box starting in 2026, requiring employers of at least 3 full-time workers or significant monthly R&D expenditure. As of early 2026, [those restrictions are on hold](https://latwy-start.pl/en/ip-box-and-r-d-support-in-poland/) and haven't been implemented. But they could come back. Don't build your entire tax strategy around IP Box without a backup plan.

### Sp. z o.o. Tax Math

The company pays CIT on profits:
- **9% CIT** if annual revenue is under 2 million EUR (this is the "small taxpayer" rate)
- **19% CIT** if above that threshold

Then when you take money out as a dividend, you pay 19% PIT on the dividend amount.

Combined effective rate with 9% CIT: **26.29%**
Combined effective rate with 19% CIT: **34.39%**

That looks worse than JDG ryczalt at 12%. And it is - for taking money out. But sp. z o.o. has a major advantage: money that stays in the company is only taxed once at 9%. If you're reinvesting profits into the business (hiring, marketing, infrastructure), sp. z o.o. is more tax-efficient than JDG because you're deploying pre-tax money.

There are also legal ways to extract money without dividends - paying yourself a board salary (subject to PIT but no ZUS if structured correctly), renting your personal equipment to the company, or using B2B contracts between your JDG and your sp. z o.o. These structures need proper legal and accounting advice. Getting them wrong triggers tax audits.

## VAT and VAT OSS

### Do You Need to Register for VAT?

If your annual revenue is under 200,000 PLN (increased to 240,000 PLN in 2026), you're exempt from VAT in Poland. You don't charge it, you don't file returns, and your life is simpler.

But here's the thing: for B2B SaaS sales, you probably want to register voluntarily. Why? Because you can reclaim VAT on your business expenses (servers, hardware, software subscriptions). And because EU business customers expect to see VAT on invoices - they reclaim it on their end. An invoice without VAT from a Polish company looks unprofessional to a German CFO.

For B2C sales within Poland, VAT exemption is straightforward. For B2C sales to other EU countries, you need VAT OSS.

### VAT OSS (One-Stop Shop)

When you sell digital services (SaaS subscriptions, downloadable software, e-books) to consumers (B2C) in other EU countries, you need to charge VAT at the rate of the customer's country. Germany is 19%, France is 20%, Ireland is 23%, Luxembourg is 17%.

Before OSS existed, you'd have to register for VAT in every EU country where you had customers. Now you register once in Poland through the [VIU-R form](https://www.podatki.gov.pl/vat/e-commerce-vat/pakiet-vat-e-commerce/) and file a single quarterly return (VIU-DO). Poland's tax office collects everything and distributes it to other member states.

The EUR 10,000 threshold: if your total B2C cross-border sales to all EU countries combined are under 10,000 EUR per year, you can charge Polish VAT rates instead. Once you cross that threshold, you must switch to destination-country rates.

Practical setup:
1. Register as a VAT payer in Poland (if not already)
2. Submit the VIU-R registration form electronically
3. Implement country detection in your payment flow (IP geolocation + billing address)
4. Charge the correct VAT rate per country
5. File VIU-DO quarterly (by the end of the month following the quarter)
6. Pay in PLN - the tax office handles currency conversion

If this sounds like a lot of work, keep reading. Merchant of Record services handle all of it for you.

## GDPR Compliance - The Practical Checklist

GDPR applies to you the moment you collect data from anyone in the EU. You don't need a Data Protection Officer if you're a small company and data processing isn't your core activity. But you do need these things:

### The Non-Negotiable Items

**Privacy Policy.** Explain what data you collect, why, how long you keep it, and who you share it with. Write it in plain language. Link to it from your signup page and footer. Cover these points: data controller identity (your company name and address), what data you collect, legal basis for processing (consent, contract performance, legitimate interest), data retention periods, third-party processors (hosting, analytics, payment), user rights (access, deletion, portability), and how to contact you.

**Cookie Consent.** If you use cookies beyond strictly necessary ones (session cookies, auth tokens), you need a consent banner that blocks tracking scripts until the user opts in. Google Analytics, Hotjar, Intercom chat widgets - all need consent first. "By continuing to browse this site you agree to cookies" is NOT valid consent under GDPR. The user must actively click "Accept."

Tools that handle this properly: [Cookiebot](https://www.cookiebot.com/), [Osano](https://www.osano.com/), or the open-source [Klaro](https://github.com/klaro-org/klaro-js).

**Data Processing Agreements (DPAs).** Every third-party service that processes your users' data needs a DPA. Most SaaS tools have them ready - check their legal/privacy page. You need DPAs with your hosting provider, email service, analytics tool, payment processor, and any other service that touches user data. Download them, sign them, keep them on file.

**Data Subject Requests.** Users can ask you to: show them all data you have on them, export their data in a machine-readable format, delete their account and all associated data, and restrict processing. You need to respond within 30 days. Build these capabilities into your app from day one. A "Delete my account" button isn't optional - it's a legal requirement.

**Breach Notification.** If you discover a data breach, you have 72 hours to notify your supervisory authority (in Poland, that's [UODO](https://uodo.gov.pl/)). If the breach poses a high risk to users, you must notify them directly too. Have an incident response plan ready. Know who to contact and what information to include.

### The Technical Checklist

```
[ ] HTTPS everywhere (no mixed content)
[ ] Passwords hashed with bcrypt/argon2 (never MD5/SHA1)
[ ] Database encrypted at rest
[ ] Backups encrypted and access-controlled
[ ] Logging does NOT contain personal data (no emails in logs)
[ ] Session tokens are secure (HttpOnly, Secure, SameSite)
[ ] Rate limiting on auth endpoints
[ ] Input validation and SQL injection prevention
[ ] Audit trail for data access/modification
[ ] Data retention policy implemented (auto-delete old data)
```

### What You Can Skip (For Now)

- **Data Protection Officer:** Not required unless data processing is a core activity or you handle special categories (health, biometric, criminal records)
- **Data Protection Impact Assessment:** Only required for high-risk processing (large-scale profiling, systematic monitoring)
- **ISO 27001 / SOC 2:** Nice to have for enterprise sales, not legally required

The penalty for GDPR violations can reach 4% of annual global turnover or 20 million EUR, whichever is higher. For a small SaaS, the real risk isn't the maximum fine - it's the reputational damage and the cost of dealing with a regulatory inquiry.

## Payments: Merchant of Record vs Stripe

This is the decision that will save you the most headaches. Or cause them.

### Stripe: You Are the Seller

[Stripe](https://stripe.com/) is a payment processor. You are the merchant. You handle:
- VAT calculation and collection for every EU country
- VAT filing (or VAT OSS registration and quarterly returns)
- Sales tax for US states (if you sell there)
- Invoice generation that meets each country's legal requirements
- Refunds and chargebacks
- Currency conversion decisions

Stripe's base fee: **1.5% + 0.25 EUR** for European cards, **3.25% + 0.25 EUR** for international cards. Add [Stripe Tax](https://stripe.com/tax) for automatic VAT calculation at **0.5%** per transaction. Add [Stripe Billing](https://stripe.com/billing) for subscription management at **0.7%** per transaction on the paid tier. Add currency conversion at **2%**.

Total effective cost for an international SaaS with subscriptions: **4.5-6%** depending on your customer mix.

### Merchant of Record: They Are the Seller

A Merchant of Record (MoR) legally sells your product on your behalf. The customer's invoice comes from the MoR, not from your company. The MoR handles all tax compliance, invoicing, chargebacks, and refunds. They pay you the net amount after fees.

**[LemonSqueezy](https://www.lemonsqueezy.com/):** 5% + $0.50 per transaction. No add-on fees. Tax compliance included. Clean checkout UI. Good for products under ~$100k MRR. Now owned by Stripe, but operates independently.

**[Paddle](https://www.paddle.com/):** 5% + $0.50 per transaction. More enterprise-oriented. Better analytics and retention tools. Supports B2B sales with proper VAT handling. Good for products over $50k MRR.

### When MoR Wins

For a solo developer or small team selling a SaaS from Poland, a Merchant of Record is almost always the right choice when starting out. Here's why:

1. **No VAT OSS registration needed.** The MoR is the seller. They handle VAT in every country. You receive a single payment from the MoR, and you invoice the MoR as a B2B service. One invoice, one tax jurisdiction.

2. **No per-country invoice requirements.** French invoices need specific fields. German invoices need Steuernummer. Italian invoices need SDI codes. The MoR handles all of it.

3. **Chargeback protection.** The MoR deals with disputes. You don't wake up to a Stripe email saying someone disputed a $49 charge and you owe $15 in dispute fees.

4. **Time savings.** The hours you'd spend on tax compliance, invoice formatting, and chargeback disputes are hours you're not spending on your product.

The math: let's say you're at $10,000 MRR with 50% international customers.

| | **Stripe + Tax + Billing** | **LemonSqueezy** |
|---|---|---|
| Transaction fees | ~$300/mo | ~$500 + ~$100 (per-txn) |
| Tax compliance | Your time + accountant | Included |
| Invoice generation | Your time or paid tool | Included |
| Chargeback handling | Your time + $15/dispute | Included |
| Total monthly cost | ~$300 + your time | ~$600 |

Stripe looks cheaper on paper. But "your time" handling tax compliance across 27 EU countries is not free. An accountant who handles international VAT will charge 500-1,500 PLN/month extra. Add the engineering time to build compliant invoicing, and LemonSqueezy is cheaper until you're well past $50k MRR.

### When to Switch to Stripe

Consider moving to Stripe (or Stripe + a tax automation tool) when:
- Your MRR exceeds $50-100k and the percentage-based fees become significant
- You need custom billing logic (usage-based pricing, complex plans)
- You're selling to enterprises that require invoices from your company specifically
- You want full control over the checkout experience

The switch isn't binary. Some companies use a MoR for self-serve customers and Stripe for enterprise contracts.

## When to Switch from JDG to Sp. z o.o.

There's no magic revenue number, but here are the triggers:

**ZUS becomes unbearable.** After your preferential period ends (30 months), you're paying ~1,927 PLN/month in ZUS regardless of revenue. If your SaaS isn't consistently generating enough to cover this plus taxes plus your living expenses, sp. z o.o. eliminates the ZUS burden.

**You want to reinvest.** With JDG, all profit is your personal income and gets taxed immediately. With sp. z o.o. at 9% CIT, you can keep money in the company for hiring, marketing, or infrastructure and only pay 9% until you take it out. If you're reinvesting more than half your profits, the math favors sp. z o.o.

**You're taking on risk.** The moment your SaaS handles sensitive data, processes payments, or signs contracts with larger companies, limited liability matters. One bad contract dispute shouldn't put your apartment at risk.

**You want to raise money.** Investors invest in companies, not sole proprietorships. If you're even thinking about fundraising, set up the sp. z o.o. first.

**You want co-founders.** JDG is for one person. The moment you add a partner, you need a company structure.

A common pattern: start with JDG during the 6-month ulga na start (health insurance only), use the 24-month preferencyjne ZUS period to validate your product, and switch to sp. z o.o. when your preferential ZUS period ends - around the 30-month mark. By then you know if the SaaS is working, and you avoid paying full ZUS while figuring that out.

## The Checklist

Here's the order of operations for a developer in Poland going from "I have a working SaaS" to "I'm legally selling it":

### Phase 1: Before You Charge Anyone

```
[ ] Decide JDG or sp. z o.o. (JDG for most people starting out)
[ ] Register your business (CEIDG for JDG, S24 for sp. z o.o.)
[ ] Open a business bank account (mBank, ING, or Nest Bank
    have free accounts for new businesses)
[ ] Pick your tax form (ryczalt 12% for most SaaS developers)
[ ] Register for VAT if selling B2B (optional under 240k PLN 
    revenue, but recommended)
[ ] Find an accountant (300-500 PLN/mo for JDG, budget more
    for sp. z o.o. or IP Box)
```

### Phase 2: Before Launch

```
[ ] Choose payment provider (LemonSqueezy or Paddle for 
    simplicity, Stripe if you need control)
[ ] Write privacy policy (cover all GDPR requirements)
[ ] Implement cookie consent (if using analytics/tracking)
[ ] Add "Delete my account" functionality
[ ] Set up HTTPS, secure headers, proper password hashing
[ ] Sign DPAs with all third-party processors
[ ] Create terms of service
[ ] Set up invoicing (your MoR handles this, or use 
    Fakturownia/inFakt for Stripe)
```

### Phase 3: After First Revenue

```
[ ] Track revenue against VAT exemption threshold (240k PLN)
[ ] Track cross-border B2C sales against OSS threshold 
    (10k EUR) - skip if using MoR
[ ] Register for VAT OSS if needed - skip if using MoR
[ ] File monthly/quarterly tax returns (your accountant 
    handles this)
[ ] Pay ZUS by the 20th of each month (JDG only)
[ ] Keep receipts for all business expenses
[ ] Review your GDPR compliance quarterly
```

### Phase 4: Scaling

```
[ ] Evaluate JDG -> sp. z o.o. switch when approaching
    end of preferential ZUS
[ ] Consider IP Box if you qualify and the savings justify
    the accounting costs
[ ] Look into Stripe if MoR fees exceed $2-3k/month
[ ] Get proper business insurance (OC dzialalnosci)
[ ] Consider KSeF (National e-Invoice System) integration
    - mandatory for VAT payers starting February 2026
```

## Resources

- [CEIDG - Register JDG](https://www.biznes.gov.pl/en/firma/doing-business-in-poland) - official government portal
- [S24 - Register Sp. z o.o.](https://ekrs.ms.gov.pl/s24/) - online company registration
- [ZUS PUE](https://www.zus.pl/portal/logowanie.npi) - manage ZUS contributions online
- [VAT OSS Registration](https://www.podatki.gov.pl/vat/e-commerce-vat/pakiet-vat-e-commerce/) - Polish tax authority
- [UODO](https://uodo.gov.pl/) - Polish data protection authority
- [LemonSqueezy](https://www.lemonsqueezy.com/) - Merchant of Record
- [Paddle](https://www.paddle.com/) - Merchant of Record
- [podatki.wtf](https://podatki.wtf/) - B2B tax calculator for Poland (community-built, great for comparing tax forms)
- [inFakt](https://www.infakt.pl/) - accounting + invoicing platform popular with Polish JDGs

## Final Thoughts

The bureaucracy looks intimidating from the outside, but the actual steps are straightforward. Register, pick a tax form, get an accountant, use a Merchant of Record, and focus on building your product.

Poland is genuinely one of the better places in the EU to run a small SaaS. The ryczalt at 12% is lower than most European tax rates for sole traders. ZUS is annoying but manageable with preferential periods and Maly ZUS Plus. And the Merchant of Record model means you can sell to all 27 EU countries without touching VAT compliance yourself.

The biggest mistake I see developers make isn't picking the wrong tax form or business structure - it's spending three months researching the "perfect" setup instead of shipping the product and charging for it. Start with JDG + ryczalt + LemonSqueezy. You can optimize later. The tax savings from switching to IP Box or sp. z o.o. don't matter if you don't have revenue yet.

Ship first. Optimize the legal structure when it actually costs you money not to.
