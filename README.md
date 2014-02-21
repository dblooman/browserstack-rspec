browserstack-rspec
==================

Setup for browser stacks using rspec and capybara

At time of writing, browser stack doesn't give a lot of help for those who don't want to use cucumber, this guide aims to solve that.

Most of the code for the cucumber setup is reusable, so we'll start there.  

## Rakefile

The `Rakefile` has a few minor changes with gems and tasks.  I have set the default nodes to 2 if you decide not to pass in a command line option.

```ruby
require 'rspec/core/rake_task'
require 'parallel_tests'
require 'parallel'
require 'json'

task :default => [:stacks]

@browsers = JSON.load(open('browsers.json'))
@parallel_limit = ENV["nodes"] || 2
@parallel_limit = @parallel_limit.to_i

task :stacks do
  Parallel.map(@browsers, :in_processes => @parallel_limit) do |browser|
      puts "Running with: #{browser.inspect}"
      ENV['SELENIUM_BROWSER'] = browser['browser']
      ENV['SELENIUM_VERSION'] = browser['browser_version']
      ENV['BS_AUTOMATE_OS'] = browser['os']
      ENV['BS_AUTOMATE_OS_VERSION'] = browser['os_version']

      Rake::Task[:spec].execute()
  end
end


RSpec::Core::RakeTask.new(:spec)
```

## Spec_Helper

You can think of `env.rb` as your `spec_helper.rb`, therfor the caps go in there.  I like to just use `Rake` in the command line, so I have removed the `ENV` option for user and authkey.  If you want to use the command line to pass credentials, refer to cucumber docs and remove replace the url variable contents.

```ruby
require 'rspec'
require 'rspec/core'
require 'capybara'
require 'capybara/rspec'
require 'capybara/dsl'
require 'selenium/webdriver'

# Use credentials here
url = "http://USERNAME:AUTHKEY@hub.browserstack.com/wd/hub"

capabilities = Selenium::WebDriver::Remote::Capabilities.new
capabilities['os'] = ENV['BS_AUTOMATE_OS']
capabilities['os_version'] = ENV['BS_AUTOMATE_OS_VERSION']
capabilities['browser'] = ENV['SELENIUM_BROWSER']
capabilities['browser_version'] = ENV['SELENIUM_VERSION']
capabilities['browserstack.debug'] = "true"

capabilities['project'] = ENV['BS_AUTOMATE_PROJECT'] if ENV['BS_AUTOMATE_PROJECT']
capabilities['build'] = ENV['BS_AUTOMATE_BUILD'] if ENV['BS_AUTOMATE_BUILD']

Capybara.register_driver :selenium do |app|
  Capybara::Selenium::Driver.new(app, :browser => :remote, :url => url, 
                                                 :desired_capabilities => capabilities)
end

# Set default driver
Capybara.default_driver = :selenium

# Make Rspec matchers available to specs
RSpec.configure do |config|
  config.include Capybara::DSL
  config.include Capybara::RSpecMatchers
 
  config.after do
    Capybara.reset_sessions!
    Capybara.use_default_driver
  end
end

# Each spec runs through each browser
describe "browsers" do
  before(:each) do
    @browser = :selenium
  end
end
```