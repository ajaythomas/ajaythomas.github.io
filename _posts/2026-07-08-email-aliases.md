---
title: Cloudflare email forwarding to mask your personal email and DomainConnect
categories:
- Tech
feature_image: "https://picsum.photos/2560/600?image=872"
---

I recently migrated my personal site ([this one](https://thomasthoughts.com/)) to Cloudflare from Squarespace (that Google had originally migrated my domain to). Their [domain migration guide](https://developers.cloudflare.com/registrar/get-started/transfer-domain-to-cloudflare/) is pretty straightforward. However, during that process, I stumbled on the DNS records for email that lets you create an email for your domain (but without a mailbox). As of July 2026, you can setup up to 100 email aliases on your paid domain (E.g: `ajay@thomasthoughts.com`).

This is especially useful to mask your personal Gmail so you can use this domain email address for sites.

Once you have setup relevant TXT and MX records for your domain email address, go to the email tab on your Cloudflare dashboard to create a routing rule for each of these email address to a destination mailbox like your personal Gmail.
At any time, any of these email addresses gets spammed, you can just kill the routing rule and your personal Gmail mailbox stays protected.

## Domain Connect

Separately, I have a personal app where I use Resend API to send emails. Resend is an email sending/receiving service with a neat API that you can use for your apps (similar to Twilio's Sendgrid). Resend requires you to setup an email address on a domain owned by you to send out emails using their API.
This is where I came across the open standard "DomainConnect" - similar to OAuth. It is a very useful standard that removes the pain for users to manually add DNS records to prove domain ownership and setting up email DNS records on your domain registrar.

There are 2 parties:

1. DNS provider (E.g: Cloudflare where I have my domain)
1. Service provider (Resend - a SaaS product which provides the service attached to the above domain)

On Resend (Service Provider), in their domains tab, I would add my Cloudflare domain (`thomasthoughts.com`) and choose "Auto Configure". This triggers the DomainConnect flow  I would initiate the DomainConnect Synchronous flow that opens up a pop up with my logged in Cloudflare session. 

This does 2 things:

1. Verifies that I own the domain on CloudFlare
1. Adds the MX and TXT records needed on Cloudflare DNS records to support Resend using that

As you can see in the screenshot below, it is a one time authorization granted to Resend to add these DNS records. Pretty cool! I love seeing rubber hitting the road open standards that solve a very clear user problem. 

{% include figure.html image="/assets/blog/email-aliases/domain_connect.png" %}

## Useful links

1. [Cloudflare Email DNS records primer](https://developers.cloudflare.com/dns/manage-dns-records/how-to/email-records/)
1. DomainConnect [primer](https://www.domainconnect.org/) and [synchronous flow](https://www.domainconnect.org/getting-started/)