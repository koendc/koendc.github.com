---
layout: post_page
title:  "How to test your Logstash configuration"
date:   2013-10-11 14:32:03
---

When pushing more and more types of events to [Logstash], the configuration easily became quite complex and hard to manage.
It's easy to make both *syntax* and *logical* errors. So, testing your logstash configuration is good thing.

### Step 1: Syntax check

We start with a quick win: see if the configuration file you created is valid. Logstash has a built-in command option for it:

{% highlight bash %}
java -jar logstash.jar agent --configtest --config <yourconfigfile>
{% endhighlight %}

It's there since [1.1.10][configtest]. To avoid mistakes, you can add it to your init script.


### Step 2: logstash rspec

For logical errors, `logstash rspec` comes to help:

{% highlight bash %}
java -jar logstash.jar rspec <your spec file>
{% endhighlight %}

Cool to see find out those nifty, slightly hidden features of Logstash!

How do you write those tests? Let's take an example:

{% highlight ruby %}
require "test_utils"

describe "haproxy logs" do
  extend LogStash::RSpec

  config <<-CONFIG
  filter {
    grok {
       type            => "haproxy"
       add_tag         => [ "HTTP_REQUEST" ]
       pattern         => "%{HAPROXYHTTP}"
    }
    date {
       type            => 'haproxy'
       match           => [ 'accept_date', 'dd/MMM/yyyy:HH:mm:ss.SSS' ]
    }
  }
  CONFIG

  sample({'@message' => '<150>Oct 8 08:46:47 syslog.host.net haproxy[13262]: 10.0.1.2:44799 [08/Oct/2013:08:46:44.256] frontend-name backend-name/server.host.net 0/0/0/1/2 404 1147 - - ---- 0/0/0/0/0 0/0 {client.host.net||||Apache-HttpClient/4.1.2 (java 1. 5)} {text/html;charset=utf-8|||} "GET /app/status HTTP/1.1"',
          '@source_host' => '127.0.0.1',
          '@type' => 'haproxy',
          '@source' => 'tcp://127.0.0.1:60207/',

      }) do

    insist { subject["@fields"]["backend_name"] } == [ "backend-name" ]
    insist { subject["@fields"]["http_request"] } == [ "/app/status" ]
    insist { subject["@timestamp"] } == "2013-10-08T06:46:44.256Z"
    reject { subject["@timestamp"] } == "2013-10-08T06:46:47Z"
  end
end

{% endhighlight %}

You can see the pattern:

1. Specify filter configuration. Filters are the most important (and easiest) to test.
2. Add some sample input. This is what the filter would get from the input side.
3. Write expectations for the sample input.

When you run the test, this is the output:

{% highlight bash %}
$ java -jar logstash.jar rspec --color example_spec.rb 
.

Finished in 0.028 seconds
1 example, 0 failures
{% endhighlight %}

Cool! Of course you can test a whole configuration file instead of a snippet in the spec file:

{% highlight ruby %}
describe "haproxy logs" do
  extend LogStash::RSpec

  file = File.open("path/to/etc/logstash_filters.conf", "rb")
  contents = file.read
  file.close
  config  contents

  # sample block goes here
end
{% endhighlight %}

Make sure you leave the `input { }` and `output { }` parts out of the configuration files.

### Bonus points: test your puppet-templated Logstash configuration file


[Logstash]:   http://logstash.net
[configtest]: https://logstash.jira.com/browse/LOGSTASH-345
