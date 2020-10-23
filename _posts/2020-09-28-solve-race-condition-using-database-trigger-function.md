---
layout: post
title: Solve race condition using database trigger function
---

For this article, we will be using two ActiveRecord models like this : 

![ERD](https://rubyyagi.s3.amazonaws.com/5-race-condition-trigger/erd.png)



As example, we have a **Budget** model, which have allocation_cents as the amount allocated for the budget, which can be used to buy the items proposed in the budget.



A budget can have many_items, and we need make sure that the sum of the items' price (price_cents) does not exceed the budget's allocation amount (allocation_cents).



Here's the **budget.rb** model file :

```ruby
class Budget < ApplicationRecord
  has_many :items, dependent: :delete_all
end
```



And here's the **item.rb** model file, with validation that check if the sum of items' price has exceeded budget allocation : 

```ruby
class Item < ApplicationRecord
  belongs_to :budget

  validate :total_price_of_items, if: proc { |t| t.budget.present? }

  def total_price_of_items
    used_budget = 0
    # get the sum of the existing items price in the budget, if they exist
    used_budget = budget.items.map(&:price_cents).reduce(:+) if budget.items.count.positive?

    # add the sum with the current item's price (which will be added into the budget)
    # if the sum is more than the budget allocation, raise error
    if used_budget + price_cents > budget.allocation_cents
      errors.add(:base, 'Items total price has exceeded budget allocation')
    end
  end
end
```



For example, if we have a budget of 10000 cents, and we keep adding an item that cost 3000 cents into the budget, it would raise an error on the 4th item (4 x 3000 = 12000 , which is larger than 10000).



<script id="asciicast-QDjl7dS03E2NVLH4NiSXK3jud" src="https://asciinema.org/a/QDjl7dS03E2NVLH4NiSXK3jud.js" data-speed="2" async></script>



This validation works well... until you have multiple users adding items to the budget at almost the same time.



When multiple users add item to the budget at (almost) the same time, the model validation passes as the validations are done at the almost same time, which the items of every user is not added in yet.



**Timeline** : 

1. User 1 check if the items total price is larger than budget allocation, validation passes.
2. User 2 check if the items total price is larger than budget allocation, validation passes.
3. User 3 check if the items total price is larger than budget allocation, validation passes.
4. User 1 add the item into the budget
5. User 2 add the item into the budget
6. User 3 add the item into the budget



We can simulate this concurrent add items by multiple users scenario, by creating multiple threads and join them to run multple add items statement at the same time :

```ruby
threads = concurrency_level.times.map do
  Thread.new do
   # dont start execution until we allow it to
   true while wait_for_it
   # the thread will keep looping the above line until we change 'wait_for_it' to false  
     
   budget.items.create(name: 'beers', price_cents: 4000)
  end
end

wait_for_it = false
# this will create 4 beer item, simultaneously
threads.each(&:join)
```



After executing this, the budget will have 4 beers item, with the total price of 12000, which exceed the 10000 allocation!



To guard against this, we can utilize database function, which validate right before a record is created in the database.



## Database function and trigger for validation

As there's no ActiveRecord function for creating SQL function / trigger, we have to write the raw SQL ourselves.

`rails g migration CreateTriggerItemTotalCheck`



Then in the migration file : 

```ruby
class CreateTriggerItemTotalCheck < ActiveRecord::Migration[6.0]
  def up
    execute <<-SQL
      CREATE OR REPLACE FUNCTION check_item_total()
        RETURNS TRIGGER 
      AS $func$
      DECLARE
        allowed_total BIGINT;
        new_total     BIGINT;
      BEGIN
        SELECT INTO allowed_total allocation_cents
        FROM budgets
        WHERE id = NEW.budget_id;
       
        SELECT INTO new_total SUM(price_cents)
        FROM items
        WHERE budget_id = NEW.budget_id;
       
        IF new_total > allowed_total
        THEN
          RAISE EXCEPTION 'Items total price [%] is larger than budget allocation [%]',
          new_total,
          allowed_total;
        END IF;
        RETURN NEW;
      END;
      $func$ 
      LANGUAGE plpgsql;

      CREATE TRIGGER item_total_trigger
      AFTER INSERT OR UPDATE ON items
          FOR EACH ROW EXECUTE PROCEDURE check_item_total();
    SQL
  end

  def down
    execute <<-SQL
      DROP TRIGGER item_total_trigger ON items;
    SQL
  end
end
```

 



**CREATE_AND_AND_REPLACE_FUNCTION** will create a function with the name **check_item_total()** or replace it if it already exists. The function accepts no parameter and return a trigger type (RETURNS TRIGGER), you can think of trigger like ActiveRecord callback, which we can set it to execute after a model object (row) is created (inserted).



Between **AS $func$** and **$func$** is the actual SQL function, we can think of **AS $func$** and **$func$** as the delimiter for multiline strings in Ruby, similar like the **<<-SQL** and **SQL**  part.



In the **DECLARE** section, we can declare two variables **allowed_total** and **new_total**, they both have the type BIGINT.



The actual function is located between the **BEGIN** and **END** statement.



The "**NEW**" in the function refers to the new row we want to insert into the **items** table, or the existing row we want to update in the items table. 

![new row](https://rubyyagi.s3.amazonaws.com/5-race-condition-trigger/new_row.png)



```SQL
SELECT INTO allowed_total allocation_cents
FROM budgets
WHERE id = NEW.budget_id;
```

 

The above statement will select the "allocation_cents" value from the budget which the new item row belongs to, and save it into the variable **allowed_total**.



```sql
SELECT INTO new_total SUM(price_cents)
FROM items
WHERE budget_id = NEW.budget_id;
```

 

The above statement will select the sum of the price_cents of all items that belongs to the budget, and save in into the variable **new_total**. As the function (trigger) is run after the new item is inserted, this new total includes the price of the new row.



```sql
IF new_total > allowed_total
THEN
  RAISE EXCEPTION 'Items total price [%] is larger than budget allocation [%]',
  new_total,
  allowed_total;
END IF;
```

 

If the new total (sum of all items price) is larger than allowed total (allocation of budgets), raise an exception, which will rollback the insertion of the new item row.



We end the function by returning the new row (RETURN NEW) if there is no exception, ie. it created successfully.



After the function, we create a trigger that will be executed every time a new row inserted into the items table, and when an existing row in the items table is updated : 

```sql
CREATE TRIGGER item_total_trigger
AFTER INSERT OR UPDATE ON items
    FOR EACH ROW EXECUTE PROCEDURE check_item_total();
```

 

After each insert or update of row in items table, the **check_item_total** function will be run.



For the **down** migration, we can choose to delete the trigger as reversal.



Now when we attempt to add more item when the budget allocation is exceeded, it will roll back and a "**ActiveRecord::StatementInvalid**" exception will be thrown.



In your controller action, you can rescue this exception and show an error message :

```ruby
# budgets_controller.rb

def update
  # ...
rescue ActiveRecord::StatementInvalid => e
  flash[:error] = "Items total exceeded budget allocation, please try again"
  render 'edit'
end
```





## Adding trigger to schema.rb using fx gem

If you open **schema.rb** after the migration of the SQL trigger, you would notice that there's no information about the trigger function in it. This would cause problem when you are running test using the test database, as `rake db:migrate RAILS_ENV=test` will just copy the schema.rb structure to the database (using rake db:schema:load), without going through all the migrations file one-by-one.



We can solve this by using **[fx gem](https://github.com/teoljungberg/fx)** , this gem will include the SQL function into schema.rb after the migration.



Before continuing, make sure to rollback migration to before adding the SQL function, and delete the migration file of the SQL function, to ensure no conflict for the migration created by the fx gem.



Include the 'fx' gem in your **Gemfile** :

`gem 'fx'`



then run **bundle install**.



After installing the fx gem, we can use its generator to create a function, in terminal, run : 

`rails generate fx:function check_item_total`



This will generate a new migration file and a blank sql file in **db/functions/check_item_total_v01.sql** .



We can then move the SQL function (excluding the trigger) from the previous migration to this file.



![sql_file](https://rubyyagi.s3.amazonaws.com/5-race-condition-trigger/function_sql.png)





Next we will create a trigger for this function using this command : 

`rails generate fx:trigger item_total_trigger table_name:items`



This will generate a new migration file and a blank sql file in **db/triggers/item_total_trigger_v01.sql**.

We can then move the SQL trigger from the previous migration to this file.

![trigger sql](https://rubyyagi.s3.amazonaws.com/5-race-condition-trigger/trigger_sql.png)



Now we can run `rake db:migrate` to add the function and trigger into the schema and also database.



This time the SQL function and trigger appears in the schema.rb! 



## Testing race condition

Arkency has written a good article on [how to test race condition here](https://blog.arkency.com/2015/09/testing-race-conditions/). We can reference it to simulate race condition to add multiple items at the same time by using multiple threads (using rspec) : 



```ruby
require 'rails_helper'

describe 'Create multiple items for the budget at the same time' do
  context 'items price more than allocation' do

    let!(:budget) { create(:budget, allocation_cents: 10_000) }

    it 'should fail' do
      # https://blog.arkency.com/2015/09/testing-race-conditions/
      expect(budget.allocation_cents).to eq(10_000)

      fail_occurred = false
      wait_for_it = true

      # create multiple threads to create the same payment at the same time
      threads = 4.times.map do
       Thread.new do
         # halt execution until we allow it to
         true while wait_for_it
         begin
           budget.items.create(name: 'beers', price_cents: 3000)
         rescue ActiveRecord::StatementInvalid => e
           # SQL exception will be thrown, then ActiveRecord will throw statement invalid
           # and we will catch it here
           fail_occurred = true
         end
       end
      end
      wait_for_it = false
      threads.each(&:join)

      # Add delay in case the previous database transaction rollback is not finished yet
      sleep 2
      
      # only 3 items should pass through, 3 x 3000 = 9000
      expect(budget.items.count).to eq(3)
      
      # sum of budget items price should be equal or smaller than budget allocation
      expect(budget.items.map(&:price_cents).reduce(:+)).to be <= budget.allocation_cents
      
      # ActiveRecord should throw an error
      expect(fail_occurred).to eq(true)
    end
  end
end
```

 



After adding the database trigger and function, this spec should pass. However in some older Postgresql version (12.2 and older), I have encountered a bug where the schema exports  "FOR EACH ROW EXECUTE **FUNCTION**" instead of "FOR EACH ROW EXECUTE **PROCEDURE**" after running rake db:migrate, despite us typing "FOR EACH ROW EXECUTE PROCEDURE" in the SQL file. This might cause the database function not being called!



One simple fix for this is to manually change the "FUNCTION" into "PROCEDURE" in the schema.rb file, and run **rake db:migrate RAILS_ENV=test** to ask the test database to copy the schema again.



<script id="asciicast-361577" src="https://asciinema.org/a/361577.js" async></script>



With database trigger , we can guard against race condition better as the validation is done on the database level instead of application level.


<script async data-uid="d862c2871b" src="https://rubyyagi.ck.page/d862c2871b/index.js"></script>