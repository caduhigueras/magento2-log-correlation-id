# magento2-log-correlation-id

[![Build Status](https://app.travis-ci.com/AmpersandHQ/magento2-log-correlation-id.svg?token=4DzjEueYNQwZuk3ywXjG&branch=main)](https://app.travis-ci.com/AmpersandHQ/magento2-log-correlation-id)

Magento 2 log correlation id for PHP requests/processes and magento logs.

This is useful when debugging issues on a production site as high amounts of traffic can cause many logs to be written and identifying which logs belong to a specific failing request can sometimes be difficult.

With this you should easily be able to find all associated logs for a given web request or CLI process.
- From a web request you can look at the `X-Log-Correlation-Id` header then search all your magento log files for the logs corresponding to only that request.
- You can also find the log correlation identifier attached to New Relic transactions.

## Install

Composer install the module.
```bash
composer require ampersand/magento2-log-correlation-id
```

Add the additional dependency injection config necessary to boot early in the application flow
```bash
mkdir app/etc/ampersand_magento2_log_correlation
cp vendor/ampersand/magento2-log-correlation-id/dev/ampersand_magento2_log_correlation/di.xml app/etc/ampersand_magento2_log_correlation/
```

Run module installation
```bash
php bin/magento setup:upgrade
```

## Uninstall

```bash
rm app/etc/ampersand_magento2_log_correlation/di.xml
php bin/magento module:disable Ampersand_LogCorrelationId
```

## How it works

###  Entry point

This module creates a new cache decorator (`src/CacheDecorator/CorrelationIdDecorator.php`). 

It needs to be here so that it's constructed immediately after `Magento\Framework\Cache\Frontend\Decorator\Logger` which is the class responsible for instantiating `Magento\Framework\App\Request\Http` and the Logger triggered after. 

This is the earliest point in the magento stack where we can get any existing traceId from a request header (for example `cf-request-id`) and have it attached to any logs produced.

This cache decorator initialises the identifier which is immutable for the remainder of the request.

### Exit points
- The correlation ID is attached to web responses as `X-Log-Correlation-Id` in `src/HttpResponse/HeaderProvider/LogCorrelationIdHeader.php`
    - REST API requests work a bit differently in Magento and attach the header using `src/Plugin/AddToWebApiResponse.php`
- Monolog files have the correlation ID added into their context section under the key `amp_correlation_id` via `src/Processor/MonologCorrelationId.php`
- Magento database logs have this identifier added by `src/Plugin/AddToDatabaseLogs.php`
- New Relic has this added as a custom parameter under the key `amp_correlation_id`

## Example usage

Firstly you need to expose the header in your logs, this is an example for apache logs

```apacheconf
Header always note X-Log-Correlation-Id amp_correlation_id
LogFormat "%t %U %{amp_correlation_id}n" examplelogformat
CustomLog "/path/to/var/log/httpd/access_log" examplelogformat
```

If you are using Nginx, that's how you can add the correlation id to the access logs

```nginx
log_format examplelogformat '$time_local  $request $sent_http_x_log_correlation_id'
```

The above configuration would give log output like the following when viewing a page

```shell
[13/Jan/2021:11:34:37 +0000] /some-cms-page/ cid-61e04d741bf78
```

You could then search for all magento logs pertaining to that request 

```shell
$ grep -ri 61e04d741bf78 ./var/log
./var/log/system.log:[2022-01-13 16:04:14] main.INFO: some_log_entry_61e04d7e2ed03 {"amp_correlation_id":"cid-61e04d741bf78","some":"context"} []
```

If the request was long-running, or had an error it may also be flagged in new relic with the custom parameter `amp_correlation_id`
 
## Configuration and Customisation

### Change the key name from `amp_correlation_id`

You can change the monolog/new relic key from `amp_correlation_id` using `app/etc/ampersand_magento2_log_correlation/di.xml`

```xml
<type name="Ampersand\LogCorrelationId\Service\CorrelationIdentifier">
    <arguments>
        <argument name="identifierKey" xsi:type="string">your_key_name_here</argument>
    </arguments>
</type>
```

### Use existing correlation id from request header

If you want to use an upstream correlation/trace ID you can define one `app/etc/ampersand_magento2_log_correlation/di.xml`

```xml
<type name="Ampersand\LogCorrelationId\Service\CorrelationIdentifier">
    <arguments>
        <argument name="headerInput" xsi:type="string">X-Your-Header-Here</argument>
    </arguments>
</type>
```

If this is present on the request magento will use that value for `X-Log-Correlation-Id`, the monolog context, and the New Relic parameter. Otherwise magento will generate one.

For example 
```shell
$ 2>&1 curl  -H 'X-Your-Header-Here: abc123' https://your-magento-site.example.com/ -vvv | grep "X-Log-Correlation-Id"
< X-Log-Correlation-Id: abc123
```

```shell
$ 2>&1 curl https://your-magento-site.example.com/ -vvv | grep "X-Log-Correlation-Id"
< X-Log-Correlation-Id: cid-61e4194d1bea5
```

### Custom Loggers

By default this module hooks into all vanilla magento loggers.

However third party modules may define additional loggers to write to custom files. If you want the correlation ID added to those logs as well you will need to create a module that depends on both `Ampersand_LogCorrelation_Id` and the module with the custom logger, you will then have to add the log handler in `di.xml` like so

```xml
<type name="This\Is\Some\Logger">
    <arguments>
        <argument name="processors" xsi:type="array">
            <item name="correlationIdProcessor" xsi:type="array">
                <item name="0" xsi:type="object">Ampersand\LogCorrelationId\Processor\MonologCorrelationId</item>
                <item name="1" xsi:type="string">addCorrelationId</item>
            </item>
        </argument>
    </arguments>
</type>
```

This module provides a command to try to help you keep track of the custom loggers in your system

```shell
$ php bin/magento ampersand:log-correlation-id:list-custom-loggers
Scanning for classes which extend Monolog\Logger
- You need to run 'composer dump-autoload --optimize' for this command to work
- Use this on your local environment to configure your di.xml
- See vendor/ampersand/magento2-log-correlation-id/README.md
--------------------------------------------------------------------------------
Dotdigitalgroup\Email\Logger\Logger
StripeIntegration\Payments\Logger\WebhooksLogger
Yotpo\Yotpo\Model\Logger
--------------------------------------------------------------------------------
DONE
```
