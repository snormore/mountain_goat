---
layout: post
title: 3 possible reasons why your Rspec tests are flaky
---

You pushed a commit for the new feature, waited 10 minutes for CI to finish run (maybe you went to slack off at Reddit or Hacker News while waiting), then you received an email from CI saying two of your test specs failed!  “It passed the last few times! Why did it fail?! I didn’t change anything that affect these tests!” You then think that it might be false negative on the CI side, and click “rerun” , and hope that this time the test can pass, so that your boss can review your pull request and merge it as the feature is quite urgent. 



The test might have passed this time, but you **know** that something is wrong with the spec, as the feature works as intended (or is it?) on the production. As your workload gotten lighter, you decided to sit down and try to figure out why the flaky test occur.



Usually when your test is flaky, ie. sometimes it pass and sometimes it fail (it doesn’t pass all the time and doesn’t fail all the time), it means there are some variables that does not follow a fixed value, this usually happen when you use factory (eg: FactoryBot gem) to set up a variable in test.



From my experience on debugging Flaky test, this largely falls into 3 categories.

1. [Time dependent variable or functionality](#time)
2. [Faker Randomness](#faker)
3. [Dependent on third party API response](#third)



<span id="time"></span>

## Time dependent variable or functionality

Let's say your Rails app has a "get today orders" functionality which allow user to retrieve orders which have been created today, then you might have a spec to check if the 'get today orders' functionality perform correctly (check if the number of orders returned is correct, don't include yesterday orders etc).



Then you have a factory that create orders and set their creation time to `2.hours.ago` , you might assume that order created “2 hours ago” will be on the same day that you list the order. Then you ran the spec at 12.30am and expect there should be 1 order created today, which there is not because 2 hours ago was 10.30pm, which was yesterday!



```ruby
# FactoryBot gem

let!(:today_order) do
  create(:order, created_at: 2.hours.ago)
end

#... capybara script here to fill in form and click 'search' button

expect(number_of_today_orders).to eq(1)
# this will fail if you run the spec between 12am - 1:59am
```

Instead of using a hardcoded `2.hours.ago`,  we can use **Date.today.beginning_of_day** which will guarantee the order to be created in the same day as today.

```ruby
let!(:today_order) do
  create(:order, created_at: Date.today.beginning_of_day)
end
```



Another possible cause for timing issue is **different timezone**. I have encountered this a few times, which the CI server timezone isn’t the same as my machine, causing the time returned from `Time.now` from CI to be different than my local machine one. And also parsing  date / time from string without Timezone might cause this too.



You might have an input field which accept date time input from user, eg: params[:sign_up_time] = '24 Oct 2019 8:00 PM' , which doesn't include the timezone.

```ruby
# In my local machine, which machine timezone is set at UTC +8
'24 Oct 2019 8:00 PM'.to_time
=> "2019-10-24T20:00:00.000+08:00"
# 8pm in UTC+8 is 12pm in UTC+0

# In CI server, which machine timezone is set at UTC +0
'24 Oct 2019 8:00 PM'.to_time
=> "2019-10-24T20:00:00.000+00:00"
# 8pm in UTC+0!
```


In Rails's ActiveSupport [date handling code](https://github.com/rails/rails/blob/b9ca94caea2ca6a6cc09abaffaad67b447134079/activesupport/lib/active_support/core_ext/date/conversions.rb#L79-L84), there's a comment mentioning that if no argument is provided, **to_time** will use the **Ruby's process timezone** (most likely the timezone of your server's operating system), instead of the timezone you have set in Rails configuration.

```ruby
# NOTE: The :local timezone is Ruby's *process* timezone, i.e. ENV['TZ'].
#       If the *application's* timezone is needed, then use +in_time_zone+ instead.
def to_time(form = :local)
  raise ArgumentError, "Expected :local or :utc, got #{form.inspect}." unless [:local, :utc].include?(form)
  ::Time.send(form, year, month, day)
end
```

 



Which means the following timezone in Rails config will not be used : 

```ruby
# config/environments/production.rb

# to_time won't use this timezone
Rails.application.configure do
  config.time_zone = 'Kuala Lumpur'
end
```

 



The fix for this is to use **in_time_zone** instead of to_time, which will use the timezone in Rails config to parse the string. [Source code](https://github.com/rails/rails/blob/b9ca94caea2ca6a6cc09abaffaad67b447134079/activesupport/lib/active_support/core_ext/string/zones.rb#L9-L14)

```ruby
# In my local machine, which machine timezone is set at UTC +8
# Rails config timezone is set to UTC +8
'24 Oct 2019 8:00 PM'.in_time_zone
=> Thu, 24 Oct 2019 20:00:00 +08 +08:00
# 8pm in UTC+8

# In CI server, which machine timezone is set at UTC +0
# Rails config timezone is set to UTC +8
'24 Oct 2019 8:00 PM'.in_time_zone
=> Thu, 24 Oct 2019 20:00:00 +08 +08:00
# 8pm in UTC+8
```

 



If your spec or model factory is dependent on time / timezone and is flaky, it might be a good idea to take a look into it.



<span id="faker"></span>

## Faker Randomness

It’s common to use Faker to generate random data for factory variable in your spec, each test run will have different random faker string output, and sometimes they cause problem.



The following example is a stupid mistake of mine, I've used URL parameters to pass the model name from one controller action to another action, eg: **http://localhost:3000/controller/action?car_name=toyota&customer_name=axel** . For the factory of my spec, I used something like this : 



```ruby
let!(car_name) { Faker::Vehicle.manufacture }
```

 



and sometimes the Faker will generate an output like "BAY EQUIPMENT & REPAIR" , and this output will be passed to URL params without proper encoding : **http://localhost:3000/controller/action?car_name=BAY%20EQUIPMENT%20&%20REPAIR&customer_name=axel**


It is passing the following parameters : 

1. car\_name=BAY EQUIPMENT
2. REPAIR=
3. customer\_name=axel



There was an extra “REPAIR” parameter due to the “&” symbol not being encoded properly. I used the javascript function `encodeURI()` to encode the parameter before passing it to the URL, but the correct javascript function is **encodeURIComponent()** , lesson learnt! 😂



Your use case of Faker might be different than mine, but if you have Flaky test on spec that uses Faker, it might be a good idea to inspect if any of the string generated might cause problem, especially symbols characters.



<span id="third"></span>

## Dependent on third party API response

In some occassion, your server might integrate with third party API and communicate via HTTP request / response. You might have called the third party API in your test spec like this : 

```ruby
require 'net/http'

it 'third party API should return valid http response' do
  response = Net::HTTP.get_response('example.com', '/')

  # this will fail if example.com is down when the test is run
  expect(response.code).to eq(200)
end

```

 



If the third party server is down or they change their response unexpectedly when the test is running, it will fail. (Looking at you, Quickbook)



Your Rails app should not be responsible for the uptime of others server, one solution for this is to record down the valid response returned when the first request is made, save this response and return this saved response on subsequent request.



There's a gem for this, [VCR](https://github.com/vcr/vcr) , this gem will record and save the response retrieved if there is no previously saved response for a certain request. Then on subsequent requests, it will return the saved response.



Taking a sample code from their README.md ,

```ruby
require 'rubygems'
require 'test/unit'
require 'vcr'

VCR.configure do |config|
  # the recorded response will be saved in this folder
  config.cassette_library_dir = "fixtures/vcr_cassettes"
  config.hook_into :webmock
end

class VCRTest < Test::Unit::TestCase
  def test_example_dot_com
    # VCR will first check if fixtures/vcr_cassettes/sypnosis.yml exist
    # if it exist it will return the response recorded in the yml file
    # if it doesn't exist, an actual HTTP request will be made to the server,
    # and the response will be saved into the .yml file
    VCR.use_cassette("synopsis") do
      response = Net::HTTP.get_response(URI('http://www.iana.org/domains/reserved'))
      assert_match /Example domains/, response.body
    end
  end
end
```

 



By using cached response,  your test suite results won't be affected if the third party API is having downtime during the test run. (But your actual Rails app might be affected, you should handle this with care)

<script async data-uid="4776ba93ea" src="https://rubyyagi.ck.page/4776ba93ea/index.js"></script>


