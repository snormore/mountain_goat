---
layout: post
title: Run specific tests on Rspec using tag
---

Sometimes when you are modifying a feature on a large Rails app, it would be faster to just run specs related to that feature instead of running the full test spec (eg: 1000+ feature specs that takes 20 minutes to run).



You can use tag for this, say we want to tag a few test cases related to checkout flow, with the tag "**:checkout**"  on the second parameter of **it** / **describe** / **context** .



```ruby
# checkout_discount.spec
# ...
it 'should discount price when promo code applied', :checkout do
  # your test code here
end
```



You can use the same tag on different spec file :

```ruby
# checkout_referral.spec
# ...
describe 'referral code applied', :checkout do
  it 'should give bonus to referral account' do
  # your test code here ...
  end
end
```



Now you can specify rspec to only run tests which are tagged with :checkout like this :

`rspec spec --tag checkout`



This command means that rspec will check test files located inside the "spec" folder, and only run blocks (**it** / **describe** / **context**) which are tagged with 'checkout'. You can change this tag name to any name you like.

