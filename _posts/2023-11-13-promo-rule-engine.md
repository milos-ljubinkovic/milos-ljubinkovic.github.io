---
title: "Rule engine for promotional codes and coupons"
date: 2023-11-01T15:34:30-04:00
categories:
    - blog
tags:
    - Architecture
---

# Rule engine for promotional codes and coupons

We wanted a more robust and flexible solution for giving promo codes and discounts and attaching them to Stripe invoices, as stripe solution had limited condition settings.

I designed a rule engine where every promo code has two sets of rules of different type. Conditions are rules that all have to be passed for the promo code to be successfully applied. Actions are rules that get applied directly to the invoice or in some way to the user object 

Attach and spend, use only spend if working with a checkout type system and both of using off-session payments. 

Db switchable in the repo layer

Make it a lib!

