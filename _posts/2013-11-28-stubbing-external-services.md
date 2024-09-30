---
title:  "Stubbing external services"
categories: testing ruby rails
---

This post takes a little detour into (IMHO) the best way of stubbing an
external resource like a payment gateway.

## Testing with VCR

Recently I was implementing a subscription service into a product and I
decided to use [Braintree][braintree]. They provide a nice ruby wrapper
library and a sandbox to test your functionality before you go live. Testing
the implementation is a whole another story. One witch I'm gonna go into in this post.

Testing external services should be done by not directly stubbing the
calls to the library. Instead it should be accomplished by either
stubbing the http (or any other network protocol) calls or recording the
intercactions and playing them back. Avdi Grimm has a good freebee
episode of [Ruby Tapas][ruby-tapas-end-of-mocking] on the matter.

I went ahead and started using the most popular record-playback stubbing
library around [VCR][vcr-github]. This meant we could write our tests
and after they ran successfully for the first time, they would run like
that forever. There are a couple of edge cases that cannot be reasonably
dealt with just by using the record-playback technique.

## Trouble testing

The most common case I have come across where the record-playback
technique does not work is when you need to get an external system into
a state which is not easily achievable. E.g. you want to retry
a subscription payment when the payment is past due.
More on Braintree payments can be found
[here][braintree-doc-subscriptions].

``` ruby
authorized_transaction = Braintree::Subscription.retry_charge(
  subscription.id
).transaction
result = Braintree::Transaction.submit_for_settlement(
  authorized_transaction.id
)
result.success?
#=> true
```

The sandbox does not include an option to change this property of a
payment (or to simulate a recurring transaction) which means the only
way is to setup a subscription that will fail for the next the day, let
it fail and run the test that retries the transaction on the next day.
This is both cumbersome and error-prone.


## fake_braintree

During development I read a blog post on the [Toughtbot blog][howto-stub-external-services-in-tests]
which got me thingking and eventually rolling my own little braintree
sandbox stub. This got me thinking again - someone else must have encountered
this problem and solve it somehow (perhaps much better than me). This
lead me to to discover the [Fake Braintree][fake-braintree-github] gem.
The gem encapsulated everything I did with my little solution and some
more. Despite being exactly what I needed, there were some problems
getting it to do what I needed. I would like to note those here for
my (and perhaps others) future reference.

## fake_braintree documentation

Since the documentation is mostly in the form of a github readme I was
left on my own go through the codebase. Not a problem, here is what I
found.

*Creating a braintree customer*

The first order of business is creating a customer for us to be
interacting with. The easiest way I found was to create is manually
within fake_braintree:

``` ruby
customer = FakeBraintree::Customer.new({'credit_card' =>
                                         {
                                           'number' => '4111111111111111',
                                           'expiration_month' => '12',
                                           'expiration_year' => '2015'
                                         }
                                       },
                                       {id: '42'})
customer.create

#=> {"Content-Encoding"=>"gzip"}, "..."]
```
This creates a customer along with a default credit card and an id we
controll. Note that the return value is the same as if we hit the real
braintree endpoint. This means that the following call will return the customer we
just created.

``` ruby
customer = Braintree::Customer.find('42')
#=> #<Braintree::Customer id: "42", company: nil, email: nil, fax: nil, first_name: nil, last_name: nil, phone: nil, website: nil, created_at: nil, updated_at: nil, addresses: [], credit_cards: [#<Braintree::CreditCard token: "5910f4ea0062a0e29afd3dccc741e3ce", billing_address: nil, bin: "411111", card_type: nil, cardholder_name: nil, created_at: nil, customer_id: "42", expiration_month: "12", expiration_year: "2015", last_4: "1111", updated_at: nil, prepaid: nil, payroll: nil, commercial: nil, debit: nil, durbin_regulated: nil, healthcare: nil, country_of_issuance: nil, issuing_bank: nil, image_url: nil>]>

customer.default_credit_card
#=> #<Braintree::CreditCard token: "5910f4ea0062a0e29afd3dccc741e3ce", billing_address: nil, bin: "411111", card_type: nil, cardholder_name: nil, created_at: nil, customer_id: "42", expiration_month: "12", expiration_year: "2015", last_4: "1111", updated_at: nil, prepaid: nil, payroll: nil, commercial: nil, debit: nil, durbin_regulated: nil, healthcare: nil, country_of_issuance: nil, issuing_bank: nil, image_url: nil>
```

*Creating a subscription*

Creating subscriptions is a little more tricky. You need a payment
method token (which can be obtained from a customer's credit card) and a
plan id. Continuing from the last example

``` ruby
subscription = Braintree::Subscription.create(payment_method_token: customer.default_credit_card.token,
                               plan_id: 'my_plan_id').subscription
```

Now you can run the Braintree retry_charge method on the subscription as
if the subscription was past due.
If you tried to retry_charge on a subscription in the Braintree sandbox,
you would get an error.

``` ruby
authorized_transaction = Braintree::Subscription.retry_charge(
  subscription_id,
  price
).transaction

# Run the code with fake_braintree
#=> #<Braintree::Transaction id: "2a55837dd3f2a63a64c23107afecdd95", type: "sale", amount: "10.0", status: "authorized", created_at: nil, credit_card_details: #<token: nil, bin: nil, last_4: nil, card_type: nil, expiration_date: "/", cardholder_name: nil, customer_location: nil, prepaid: nil, healthcare: nil, durbin_regulated: nil, debit: nil, commercial: nil, payroll: nil, country_of_issuance: nil, issuing_bank: nil>, customer_details: #<id: nil, first_name: nil, last_name: nil, email: nil, company: nil, website: nil, phone: nil, fax: nil>, subscription_details: #<Braintree::Transaction::SubscriptionDetails:0x007fd892886ae0>, updated_at: nil>

# Run the code with the real endpoint
#=> #<Braintree::ErrorResult params:{...} errors:<transaction:[(91531) Subscription status must be Past Due in order to retry.]>>


authorized_transaction = Braintree::Subscription.retry_charge(subscription_id, 42.0).transaction
result = Braintree::Transaction.submit_for_settlement(
  authorized_transaction.id
)
#=> true
```

To be fair, you cannot retry_charge with the current released version of
the fake_braintree gem. I needed this to work, so I created a fork where
I implemented the functionality [https://github.com/m1k3/fake_braintree][m1k3-fake_braintree-repo].
I may get around to submitting a pull-request, but in the meantime use
my forked repo instead of the official gem.

## fake_braintree gotchas

So far I only found one gotcha. I may get around to fixing it (and
submitting a pull-request), but in
the mean time one has to work around the problem. In the examples above
we created a subscription. In a real Braintree endpoint this subscription
would get added to the credit cards subscriptions. This means that after
creating the subscription and searching for customer information again
we should not get an empty array of subscriptions [see BT
documentation][braintree-doc-creditcard-subscriptions].

``` ruby
customer = Braintree::Customer.find('42')
customer.default_credit_card.subscriptions
#=> []
# this should be
#[<Braintree::CreditCard token: "5910f4ea0062a0e29afd3dccc741e3ce", billing_address: nil, bin: "411111", card_type: nil, cardholder_name: nil, created_at: nil, customer_id: "42", expiration_month: "12", expiration_year: "2015", last_4: "1111", updated_at: nil, prepaid: nil, payroll: nil, commercial: nil, debit: nil, durbin_regulated: nil, healthcare: nil, country_of_issuance: nil, issuing_bank: nil, image_url: nil>]
```

[braintree]: https://braintreepayments.com
[braintree-doc-subscriptions]: https://www.braintreepayments.com/docs/ruby/subscriptions/overview
[braintree-doc-creditcard-subscriptions]: https://www.braintreepayments.com/docs/ruby/credit_cards/details#associations
[ruby-tapas-end-of-mocking]: https://graceful.dev/courses/the-freebies/modules/testing/topic/episode-052-the-end-of-mocking
[vcr-github]: https://github.com/vcr/vcr
[fake-braintree-github]: https://github.com/thoughtbot/fake_braintree
[howto-stub-external-services-in-tests]: https://robots.thoughtbot.com/how-to-stub-external-services-in-tests/
[m1k3-fake_braintree-repo]: https://github.com/m1k3/fake_braintree
