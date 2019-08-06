---
layout: post
title: Automating secure payments with Amazon Connect and Stripe
summary: In this post we look at using Amazon Connect cloud contact centre to take credit card details and send them to Stripe
tags: [aws, connect, lambda, systems-manager]
---

## *Diclaimer*
We are using Stripe [API Direct](https://stripe.com/docs/security) method, which still requires the solution to be SAQ D under PCI, see [here](https://www.pcisecuritystandards.org/documents/SAQ_D_v3_Merchant.pdf) for more details, but note this solution is a proof of concept and not a production ready deployment.

### Analog meets digital

I was given the opportunity to work on Amazon Connect, which is a cloud based contact centre. The requirements was around integrating with a payment provider to be able to take secure payments over the phone.

My solution extends the great work already posted on the AWS Blog around creating a [secure IVR](https://aws.amazon.com/blogs/contact-center/creating-a-secure-ivr-solution-with-amazon-connect/).

So I am going to focus on the Stripe integration using Lambda. So I grabbed the API Keys from my Stripe account and added an additional function to the code from the blog post which takes the decrypted card number, creates a token using the Stripe API and then creates a payment using this token. So we are not storing the number in our solution.

The `payment.js` file looks like this

```javascript
const stripe = require("stripe")("Get your Stripe Token from SSM as well");

const Payment = {
    async createToken(card, exp_month, exp_year, cvc) {
        const token = await stripe.tokens.create({
            card: {
                number: card,
                exp_month: exp_month,
                exp_year: exp_year,
                cvc: cvc
            }
        });
        return token;
    },
    async createCharge(tokenId, amount, phoneNumber) {
        console.log(tokenId)
        const charge = await stripe.charges.create({
            amount: parseInt(amount, 10),
            currency: "nzd",
            source: tokenId,
            description: "Charge from Amazon Connect, Phone Number: " + phoneNumber
        });
        console.log(charge);
        return charge;
    }
}
```

This was a great way to get a proof of concept up and running, we are now looking at using [AWS Certificate Manager](https://docs.aws.amazon.com/acm/index.html) to store the certificates so we can automate the rotation.