---
layout: post
title: RSpec / Capybara notes
category: posts
---

I didn't pay much attention to testing in rails until recently. I thought that it takes too much time to finish an app and sometimes feels boring.

But recently I decided to make a small [task manager web-app](https://github.com/railsr/tm-assignment) and try to fully test it by using RSpec and Capybara.
I know about Minitest but don't like it for some reasons :)

Here's what I got in the end:

![codecoverage](/public/post_files/codecoverage.png)

## Setup and problems...

So the first this was installing gems

{% highlight ruby %}

group :development, :test do
  gem 'rspec-rails', '~> 3.5'
  gem 'capybara'
  gem 'selenium-webdriver' # to test pages that require js execution
  gem 'rails-controller-testing'
  gem 'factory_girl_rails'
  gem 'shoulda-matchers'
  gem 'database_cleaner'
  gem 'simplecov', :require => false
end

group :development do
  gem 'guard-rspec', require: false # to run tests when I make changes in code
end

{% endhighlight %}

Then, after `rails generate rspec:install` I had to configure the generated rspec config files:

[spec/spec_helper.rb](https://github.com/railsr/tm-assignment/blob/master/spec/rails_helper.rb)
Here I only added `expectations.syntax = [:should, :expect]`

Most of the configuration goes in [spec/rails_helper.rb](https://github.com/railsr/tm-assignment/blob/master/spec/rails_helper.rb) And in most parts it can just be copied from project to project.

---

I came across a problem where Selenum couldn't login to the app when I used `js: true`. As it turned out the problem was with configuration of Database cleaner.

![selenium](/public/post_files/selenium.gif)

The following configuration worked just fine.

{% highlight ruby %}

config.before :suite do
  DatabaseCleaner.clean_with(:truncation)
end

config.before(:each) do
  DatabaseCleaner.strategy = :transaction
end

config.before(:each, :js => true) do
  DatabaseCleaner.strategy = :truncation
end

config.before(:each) do
  DatabaseCleaner.start
end

config.append_after(:each) do
  DatabaseCleaner.clean
end

{% endhighlight %}

Also set `config.use_transactional_fixtures = false`
[More info](https://relishapp.com/rspec/rspec-rails/docs/transactions)

---

## Testing

At first it was confusing what exactly should I test.
But there's a list of what can actually be tested with different kind of specs at relishapp.com

controller specs, model specs, ...

[here's an example for controllers](https://relishapp.com/rspec/rspec-rails/v/3-5/docs/controller-specs)

Since most of the crud controller specs look the same, here's an example of how controller spec should look like. If it's not just a crud, it's still gonna be something similar.

[tasks_controller_spec.rb](https://github.com/railsr/tm-assignment/blob/master/spec/controllers/web/admin/dashboard/tasks_controller_spec.rb)

And here're the other specs of this app: [/spec](https://github.com/railsr/tm-assignment/tree/master/spec)

## Helpful resources:

- [https://www.relishapp.com/rspec/rspec-rails/docs/](https://www.relishapp.com/rspec/rspec-rails/docs/)
- Instant Rspec Test Driven Development How-To (68 pages book)
