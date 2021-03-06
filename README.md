# About Winston

Winston is a AB/split testing library which utilizes Redis and basic machine learning. At it's core, Winston is a configurable roll-your-own A/B testing tool. Winston comes with several flavors of AB testing out of the box based on user-defined configuration options.

## Features


* Supports machine learning so you can set and forget variations and let Winston determine the best one.
* Supports split testing (multiple tests per page at once)
* Supports client side javascript events as well as code-triggered events
* Minimal bloat to your codebase
* Highly configurable
* Composer based

#### Winston and Machine Learning
You can optionally tell Winston whether you'd like to enable machine learning algorithm or not. If it's enabled, Winston performs the following:

1. It first checks if a test is reliably a favorite via confidence intervals. 
2. If a test has no clear favorite, Winston decides on one of the following execution paths:
   1. It falls back to picking a random test variation a certain percentage of the time *(defaults to 10%)* 
   2. It picks the current top performing variant the remaining percentage of the time *(defaults to 90%)*. 
   3. If no test variations have any data collected, Winston will always pick at random. 

The goal of Winston is to take the guess work out of displaying a top performing test variation. It's a set and forget operation on your part and Winston takes care of the rest. The default random percentages are entirely configurable.

## Usage

Winston installation and setup is comprised of three core parts. Below is an overview of the three parts followed by details on configuring each of the parts.

1. *Configuration:*  
   You need to setup your Winston configuration file settings to your liking and create tests and variations. You can store your configuration settings however you want (array, JSON, Yaml, database, key/val store) so long as you can convert the settings to a properly formatted array when initializing Winston.
2. *Client Side Code:*  
   You need to add code to your client facing frontend website to display variations and track performance.
3. *Server Side Code:*  
   You need to create a new server side file (a controller route and action if you use MVC) which you grant Winston access to POST data to. There are two specific API endpoints you'll need for Winston which ultimately load an instance of Winston and call the following: 
   1. `$winston->recordEvent($_POST);`
   2. `$winston->recordPageview($_POST);`

#### Configuration

Winston requires a fairly lengthy configuration array of settings, tests, and test variations. For a full picture of what a configuration array looks like, check out the basic example config file:

https://github.com/popdotco/winston/blob/master/examples/config.php

#### Client Side Code

This example uses short tags, but you don't have to if they aren't enabled. In this example, we're checking to see if varying the page's headline/tagline has any affect on the frequency of clicking a button directly below it.

```php
<?php
// include the composer autoloader
require_once 'vendor/autoloader.php';

// load your configuration array from a file
$config = include('/path/to/config.php');

// load the client library
$winston = new \Pop\Winston($config);
?>

<html>
<head><title>Sample</title></head>
<body>
<!-- show a test variation -->
<h1><?= $winston->test('my-first-headline-test'); ?></h1>

<!-- add an event separate from the test that also triggers a success -->
<button type="button" <?= $winston->event('my-first-headline-test', 'click'); ?>>Sample Button</button>

<!-- load the footer javascript (echo) -->
<?= $winston->javascript(); ?>

<!-- include the required jquery lib -->
<script type="text/javascript" src="//ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
<script type="text/javascript" src="/path/to/jquery.winston.js"></script>
</body>
</html>
```

#### Server Side Code

This implementation is only an example with minimal routing support to give you an idea for how to tie in the endpoints from the config file.

```php
<?php
// include the composer autoloader
require_once 'vendor/autoloader.php';

// load your configuration array from a file
$config = include('/path/to/config.php');

// load the client library
$winston = new \Pop\Winston($config);

// we want the post data
$data = $_POST;

// determine which endpoint we're requesting
$uri = getenv('REQUEST_URI');
if ($uri == 'winston/event') {
    $winston->recordEvent($data);
} else if ($uri == 'winston/pageview') {
    $winston->recordPageview($data);
}
```

## Advanced Usage: Adding Events Within Variations via Templating

With Winston, you can add event bindings directly within your variation text/html. In each variation, you can use the syntax `{{EVENT_NAME}}` where `EVENT_NAME` is one of the supported client events found in the section below. Winston will internally find and replace these matching template strings with DOM event handlers. If the JavaScript event is triggered, the currently selected variation will trigger successfully and an AJAX request will fire to your backend indicating the success.

Here's an example of a test you can setup in your configuration file which utilizes the basic template engine:

```php
<?php
$config = array(
    'tests' => array(
        'signup-submit-button-test' => array(
            'description' => 'A sample test',
            'variations' => array(
                array(
                    'id'    => 'submit-default',
                    'text'  => '<button type="submit" {{click}}>Submit</button>'
                ),
                array(
                    'id'    => 'submit-signup-now',
                    'text'  => '<button type="submit" {{click}}>Signup Now</button>'
                ),
            )
        )
    )
);
```

## Supported Client Side Events

Winston supports triggering variation successes for all of the popular DOM events, however we suggest steering clear of mouse movement events given how frequently they trigger. The full list of supported events is `click`, `submit`, `focus`, `blur`, `change`, `mouseover`, `mouseout`, `mousedown`, `mouseup`, `keypress`, `keydown`, and `keyup`. Note that we do not handle preventing default event actions or stopping propagation. If you need that, add your own additional event bindings to the element(s).

To trigger an event in your client side code, simply call: `$winston->event('name-of-your-test', EVENT_TYPE)` where `EVENT_TYPE` is one of the events mentioned above. This method will then generate and return a DOM event string for you to output directly in your HTML, i.e.

```php
// returns: onclick="POP.Winston.event('name-of-your-test', 'SELECTED_VARIATION_ID', 'click');"
// where SELECTED_VARIATION_ID is the variation id found to be optimal/randomized by Winston
$winston->event('name-of-your-test', 'click');
```

Let's now bind a form submission event directly to a form as an example which will get attributed to the chosen variation. The order in which you call `event()` and `variation()` doesn't matter.

```php
<form <?= $winston->event('name-of-your-test', 'submit'); ?>>
    <?= $winston->variation('name-of-your-test'); ?>
    <button type="submit">Submit Form</button>
</form>
```

## Requirements

  1. PHP 5.3+
  2. Redis must be installed and accessible.
  3. Composer is required for loading dependencies such as Predis, a popular PHP Redis client.
  4. You must create server side API endpoints in your framework or custom rolled application for Winston to be able to interact with the server side Winston library. These endpoints will need to take in `POST` data, load the Winston library, and pass in the `POST` data to the Winston library. More documentation to come.
  
## Suggested Setup ##

#### Improve Redis persistence

Redis is an in-memory key/value store. It's default configuration is to save snapshots of your data every **60 seconds** or every **1000 keys changed**. Because of this, you risk data loss if any of the following were to occur:

  1. Redis fails/stops
  2. A power outage occurs without a UPC
  3. The machine crashes/restarts

If you can't tolerate losses of this magnitude and are willing to sacrifice a bit write speed, you'll want to enable **Append-only file** data persistence in your redis configuration file:

```bash
# enable append-only file
appendonly yes

# enable fsync'ing every second (up to 1 second data loss)
# for no data loss, use 'appendfsync always'
appendfsync everysec

# disable snapshotting (RDB)
save ""
```

Before updating your `redis.conf` file, you'll want to first read the guide below to backup your existing Redis database via an RDB dump to ensure no data is lost during the transition.

[You can read more about Redis persistence and configuration options here](http://redis.io/topics/persistence).
  
#### Secure Redis

You will likely want to increase the default security measures/precautions of your Redis install.

  1. Set up firewall rules (i.e. IPTables) to only allow certain machines to access the Redis port, i.e. `127.0.0.1` for the local machine or `192.168.1.X` for a machine within your same subnet. Likewise, you'll want to edit your `redis.conf` file and add `bind XXX.XXX.XXX.XXX` with your allowed IP or IPs. If you need remote access, you can use `bind 0.0.0.0` so long as you also create firewall rules to whitelist machines and grant them access to port `6379`. 
  2. Enabling password authentication is highly recommended as Redis defaults to no password with full access to all commands. Winston supports Redis authentication in the configuration file by adding `auth = 'yourredispassword'`.

```bash
# firewall using UFW on Ubuntu
sudo ufw allow from xx.xx.xx.x1 to any port 6379

# firewall using iptables 
sudo iptables -A INPUT -s XXX.XXX.XXX -p tcp -m tcp --dport 6379 -j ACCEPT 
sudo bash -c 'iptables-save > /etc/sysconfig/iptables'
```

[You can read more about Redis security and configuration options here](http://redis.io/topics/security).


## Status

Winston is *beta* and in *active development*. You can contribute, but it's up to you to determine if Winston is ready for showtime.

## Future Improvements

1. Add method(s) to retrieve current test and variation stats/performance.
2. Create a complementary graphical summary page to check on stats/performance.
