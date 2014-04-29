---
layout: post
title:  "Using a time service to aid testing"
date:   2014-04-29 19:40:00
categories: php, testing, dependency injection, symfony
---

A problem which has cropped up numerous times in projects I've worked on is that of testing code which uses the current time, or some variant thereof, such as an audit logger for example.

Consider the following code:

{% highlight php %}
class Logger
{
	function log($message) {
		$logRecord = new LogRecord();
		$logRecord->setTime(new \DateTime());
		$logRecord->setMessage($message);
		return $logRecord;
	}
}
{% endhighlight %}

Clearly, this creates a new LogRecord object, adds the message, stamps it with the current time and then returns it. Great. But how do we test it? We can validate that the message got set easily enough, but what about the time? We could do this:

{% highlight php %}
$logger = new Logger();
$record = $logger->log('hello');
$this->assertEquals(new \DateTime(), $log->getTime());
{% endhighlight %}

But this is bad. How can we be certain that the time at the point when the record was created is the same as the time when we test it? In this case it probably will be, as the two lines are right next to each other, but we don't know how long log() will take to return and there's a chance that the times won't match, leading to a test failure when there's nothing wrong with the code.

We need a better way of getting the time in. We could, of course, pass the time in:
{% highlight php %}
function log($message, $time) {
	$logRecord = new LogRecord();
	$logRecord->setTime($time);
	$logRecord->setMessage($message);
	return $logRecord;
}
{% endhighlight %}

But now we need to specify another parameter and we'll just move the problem somewhere else in our code.

We could make the time parameter optional:
{% highlight php %}
function log($message, $time = null) { 
	if(is_null($time))
		$time = new \DateTime();
	$logRecord = new LogRecord();
	$logRecord->setTime($time);
	$logRecord->setMessage($message);
	return $logRecord;
}
{% endhighlight %}

But now we can't test the pathway where we don't pass a time, as we're back to the original problem again.

The solution is to use dependency injection and create a time service. This will provide a method that will give us the current time and can be mocked in a test:
{% highlight php %}
class TimeService
{
	function now() {
		return new \DateTime();
	}
}
{% endhighlight %}

And in our code:
{% highlight php %}
class Logger
{
	$timeService = null;
	
	function __construct(TimeService $ts) {
		$this->timeService = $ts;
	}

	function log($message) {
		$logRecord = new LogRecord();
		$logRecord->setTime($this->timeService->now());	// now use the time service to get the time
		$logRecord->setMessage($message);
		return $logRecord;
	}
}
{% endhighlight %}

Now when we come to test, we can inject a mock time service and have complete control over the time:
{% highlight php %}
$ts = $this->getMock('TimeService');
$ts->expects($this->any())
	->method('now')
	->will($this->returnValue(new \DateTime('2014-04-29 14:00:00')));

$logger = new Logger($ts);
$record = $logger->log('hello');
$this->assertEquals(new \DateTime('2014-04-29 14:00:00'), $log->getTime());
{% endhighlight %}

Now our test will always pass because we never use the actual time but always use a known time. Our code can modify that date in any way it wishes, and it can take as long as it wants; we can always test the correct result because it starts from a known point.
