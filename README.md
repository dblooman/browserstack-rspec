browserstack-rspec
==================

Setup for browser stacks using rspec and capybara

At time of writing, browser stack doesn't give a lot of help for those who don't want to use cucumber, this guide aims to solve that.

Most of the code for the cucumber setup is reusable, so we'll start there.  

## Rakefile

The `Rakefile` has a few minor changes with gems and tasks.  I have set the default nodes to 2 if you decide not to pass in a command line option.

```ruby
require 'rspec/core/rake_task'
require 'parallel'
require 'json'

task :default => [:stacks]

@browsers = JSON.load(open('browsers.json'))
@parallel_limit = ENV["nodes"] || 5
@parallel_limit = @parallel_limit.to_i

task :stacks do
  Parallel.each(@browsers, :in_processes => @parallel_limit) do |browser|
    begin
      puts "Running with: #{browser.inspect}"
      ENV['SELENIUM_BROWSER'] = browser['browser']
      ENV['SELENIUM_VERSION'] = browser['browser_version']
      ENV['BS_AUTOMATE_OS'] = browser['os']
      ENV['BS_AUTOMATE_OS_VERSION'] = browser['os_version']
      Rake::Task[:spec].execute()
    rescue RuntimeError => e
      puts "Error while running task"
    end
  end
end

RSpec::Core::RakeTask.new(:spec)
```

## Spec_Helper

You can think of `env.rb` as your `spec_helper.rb`, therefor the caps go in there.  I like to just use `Rake` in the command line, so I have removed the `ENV` option for user and authkey.  If you want to use the command line to pass credentials, refer to cucumber docs and remove replace the url variable contents.
You will see support/helpers, this assumes you have a support folder with a file called `helpers.rb`.  Place your helpers in there, these helpers are then called in the Rspec setup with ```ruby config.include Helpers```
```ruby
require 'rspec'
require 'rspec/core'
require 'capybara'
require 'capybara/rspec'
require 'capybara/dsl'
require 'selenium/webdriver'
require 'support/helpers'

# Use credentials here
url = "http://USERNAME:AUTHKEY@hub.browserstack.com/wd/hub"

Capybara.register_driver :selenium do |app|
  capabilities = {
    os: ENV['BS_AUTOMATE_OS'],
    os_version: ENV['BS_AUTOMATE_OS_VERSION'],
    browser:  ENV['SELENIUM_BROWSER'],
    browser_version: ENV['SELENIUM_VERSION'],
    :"browserstack.debug" => "true",
    project: "News",
    build: "#{ENV['BS_AUTOMATE_BUILD'] if ENV['BS_AUTOMATE_BUILD']}"
  }

  Capybara::Selenium::Driver.new(app, {
    :browser => :remote,
    :url => url,
    :desired_capabilities => Selenium::WebDriver::Remote::Capabilities.new(capabilities)
  })
end

Capybara.default_driver = :selenium

# Make Rspec matchers available to specs
RSpec.configure do |config|
  config.include Capybara::DSL
  config.include Capybara::RSpecMatchers
  config.include Helpers

  config.after do
    Capybara.reset_sessions!
    Capybara.use_default_driver
  end
end

describe "browsers" do
  before(:each) do
    @browser = :selenium
  end
end
```
