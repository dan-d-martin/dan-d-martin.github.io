---
layout: post
title:  "Logger Aware Services"
date:   2014-11-11 21:29:00
categories: 
---

Like any good developer, when building Symfony applications I aim to minimise the amount of code residing in controllers by moving it into appropriate service classes with only the required dependencies injected in.

Inside these services I very often find that I would like to log something. The most correct way to do this is by injecting the Logger service but to do this for every service adds a chunk of extra code to each which reduces clarity. Futhermore, logging is generally not a mandatory part of the service's functionality, so the service should be able to function without a logger.

To get around this, I have created a LoggerAware trait which I can add to any classes needing the logger. It uses setter injection rather than the constructor to inject the logger again because we want logger support to be optional and largely invisible to the service class, so we don't want to have to modify the constructor.

Our basic LoggerAwareS trait looks like this:

{% highlight php %}
use Psr\Log\LoggerInterface as PsrLogger;
trait LoggerAware
{
	protected $logger;

	public function setLogger(PsrLogger $l)
	{
		$this->logger = $l;
	}

	public function getLogger()
	{
		return $this->logger;
	}
}
{% endhighlight %}

So, if we use it in a service class from it we can do something like this:

{% highlight php %}
class ServiceClass
{
    use LoggerAware; 

	public function someOperation()
	{
		$this->getLogger()->info('About to perform some operation');

		// do our actual function

		$this->getLogger()->info('Finished performing some operation');
	}
}

{% endhighlight %}

Now the class need concern itself only with its own operations, and aside from the actual logging lines themselves is uncluttered by code related to the logger.

However, we now have a situation where injecting the logger is mandatory if we want to do any logging. i.e. if we attempt to perform logging calls without having called setLogger() an error will result. We don't want that because the lack of a logger shouldn't prevent the service from performing its function. To get around this, we create a NullLogger class which implements the Psr LoggerInterface but performs no actual logging:

{% highlight php %}
use Psr\Log\LoggerInterface as PsrLogger;

class NullLogger implements PsrLogger
{
    public function emergency($message, array $context = array())
    {
        return null;
    }
    public function alert($message, array $context = array())
    {
        return null;
    }
	public function critical($message, array $context = array())
    {
        return null;
    }
	public function error($message, array $context = array())
    {
        return null;
    }
	public function warning($message, array $context = array()) 
    {
        return null;
    }
	public function notice($message, array $context = array()) 
    {
        return null;
    }
    public function info($message, array $context = array()) 
    {
        return null;
    }
    public function debug($message, array $context = array()) 
    {
        return null;
    }
    public function log($level, $message, array $context = array()) 
    {
        return null;
    }
}

{% endhighlight %}

Then in our LoggerAware trait we add the following constructor:

{% highlight php %}

public function __construct() 
{
	$this->logger = new NullLogger();	
}

{% endhighlight %}

Now when we use this trait we can call getLogger() and assume that a logger will definitely be returned. By default this will be a NullLogger which will just throw the messages away, but if we simply inject a functioning logger, all our calls will be handled. By doing this our class can focus on its own behaviour, it can log messages if a logger is available but it won't fail without a logger.

The code for the LoggerAware trait can be found at GitHub: [https://github.com/dan-d-martin/logger-aware](https://github.com/dan-d-martin/logger-aware)