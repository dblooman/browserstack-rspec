Browserstack-rspec
==================

Setup for [Browserstack](http://www.browserstack.com/) using rspec and capybara

At time of writing, browser stack doesn't give a lot of help for those who don't want to use cucumber, this guide aims to solve that.

Most of the code for the cucumber setup is reusable, so we'll start there.  

## Rakefile

The `Rakefile` has a few minor changes with gems and tasks.  I have set the default nodes to 2 if you decide not to pass in a command line option.

```ruby
require 'rspec/core/rake_task'
require 'parallel'
require 'json'
require 'dotenv'
require 'dotenv/tasks'

Dotenv.load(ENV['CONFIG'])

task :default => ENV['TASK']

@browsers = JSON.load(open('browsers.json'))
@parallel_limit = ENV["nodes"] || 8
@parallel_limit = @parallel_limit.to_i
@combinations_failed = []

task :mobile_stacks do
  @browsers = JSON.load(open('mobile.json'))
  Rake::Task["stacks"].invoke
end

task :stacks => :dotenv do
  begin
    Parallel.each(@browsers, :in_threads => @parallel_limit) do |browser|
      begin
        puts "Running with: #{browser.inspect}"
        ENV['SELENIUM_BROWSER'] = browser['browser']
        ENV['SELENIUM_VERSION'] = browser['browser_version']
        ENV['BS_AUTOMATE_OS'] = browser['os']
        ENV['BS_AUTOMATE_OS_VERSION'] = browser['os_version']
        ENV['BS_AUTOMATE_PLATFORM'] = browser['platform']
        ENV['BS_AUTOMATE_DEVICE'] = browser['device']
        ENV['BS_AUTOMATE_BROWSER_NAME'] = browser['browserName']
        ENV['BS_AUTOMATE_PROJECT'] = browser['project']
        ENV['BS_AUTOMATE_BUILD'] = browser['build']

        Rake::Task[:spec].execute()
      rescue RuntimeError => e
        puts "Error while running task"
      rescue Exception => e
        @combinations_failed << browser
      end
    end
  ensure
    puts "$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$"
    puts "Your test failed for following combinations"
    @combinations_failed.each do |ele|
      puts "Your test failed for >>>> #{ele.inspect}"
      exit 1
      # You can redirect this output to a log file to make it more clear
    end
  end
end

RSpec::Core::RakeTask.new(:spec) do |t|
  t.rspec_opts = ENV['TAG']
end

task :local => :dotenv do
  begin
    Rake::Task[:spec].execute()
  rescue RuntimeError => e
    puts "Error while running task"
  end
end
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
require 'capybara/poltergeist'
require 'support/helpers'
require 'selendroid'

# Use credentials here
url = "http://USERNAME:AUTHKEY@hub.browserstack.com/wd/hub"

# For local testing
Capybara.register_driver :poltergeist do |app|
  driver = Capybara::Poltergeist::Driver.new(app, {
    :timeout => 60,
    :js_errors => false,
  })

  driver
end

# For local testing
Capybara.register_driver :chrome do |app|
  Capybara::Selenium::Driver.new(app, :browser => :chrome)
end

# For local testing
Capybara.register_driver :firefox do |app|
  Capybara::Selenium::Driver.new(app, :browser => :firefox)
end

# For local testing
Capybara.register_driver :android do |app|
  Capybara::Selenium::Driver.new(app, :url => "http://localhost:4444/wd/hub", :browser => :android)
end

# For local testing
Capybara.register_driver :opera do |app|
  Capybara::Selenium::Driver.new(app, :browser => :opera)
end

# For Browserstack
Capybara.register_driver :selenium do |app|
  capabilities = {
    os: ENV['BS_AUTOMATE_OS'],
    os_version: ENV['BS_AUTOMATE_OS_VERSION'],
    browser:  ENV['SELENIUM_BROWSER'],
    browser_version: ENV['SELENIUM_VERSION'],
    :"browserstack.debug" => "true",
    :"browserstack.local" => "true",
    project: "Responsive News",
    name: "News Tests",
    build: "0"
  }

  Capybara::Selenium::Driver.new(app, {
    :browser => :remote,
    :url => url,
    :desired_capabilities => Selenium::WebDriver::Remote::Capabilities.new(capabilities)
  })
end

# Set host for URLs
Capybara.app_host = ENV['HOST']

# Capybara setting
driver = ENV['DRIVER'].to_sym
Capybara.default_driver = driver
Capybara.run_server = false

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
```

## ENV file
Here is an example of a .env file that we use, we have many for different environments and for local testing.  

### Filename would .live.env

```yaml
TASK=stacks
SITE=live
HOST=http://www.live.bbc.co.uk
TAG='--tag standard --tag video'
DRIVER=selenium
```

## Running

`CONFIG=.live.env rake`
