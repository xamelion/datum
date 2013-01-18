# Datum

[![Build Status](https://secure.travis-ci.org/dandoescode/datum.png)](http://travis-ci.org/dandoescode/datum)

A simple API extension for DateTime with PHP 5.3+

```php
<?php

use Datum\Datum;

printf("Right now is %s", Datum::now()->toDateTimeString());
printf("Right now in Vancouver is %s", Datum::now('America/Vancouver'));  //implicit __toString()
$tomorrow = Datum::now()->addDay();
$lastWeek = Datum::now()->subWeek();
$nextSummerOlympics = Datum::createFromDate(2012)->addYears(4);

$officialDate = Datum::now()->toRFC2822String();

$howOldAmI = Datum::createFromDate(1975, 5, 21)->age;

$noonTodayLondonTime = Datum::createFromTime(12, 0, 0, 'Europe/London');

$worldWillEnd = Datum::createFromDate(2012, 12, 21, 'GMT');

// comparisons are always done in UTC
if (Datum::now()->gte($worldWillEnd)) {
   die();
}

if (Datum::now()->isWeekend()) {
   echo 'Party!';
}

echo Datum::now()->subMinutes(2)->diffForHumans(); // '2 minutes ago'

// ... but also does 'from now', 'after' and 'before'
// rolling up to seconds, minutes, hours, days, months, years

$daysSinceEpoch = Datum::createFromTimeStamp(0)->diffInDays();
```

## README Contents

* [Installation](#install)
    * [Requirements](#requirements)
    * [With composer](#install-composer)
    * [Without composer](#install-nocomposer)
* [API](#api)
    * [Instantiation](#api-instantiation)
    * [Getters](#api-getters)
    * [Setters](#api-setters)
    * [Fluent Setters](#api-settersfluent)
    * [IsSet](#api-isset)
    * [Formatting and Strings](#api-formatting)
    * [Common Formats](#api-commonformats)
    * [Comparison](#api-comparison)
    * [Addition and Subtraction](#api-addsub)
    * [Difference](#api-difference)
    * [Difference for Humans](#api-humandiff)
    * [Constants](#api-constants)
* [About](#about)
    * [Contributing](#about-contributing)
    * [Author](#about-author)
    * [License](#about-license)
    * [History](#about-history)
    * [Why the name Datum?](#about-whyname)

<a name="install"/>
## Installation

<a name="requirements"/>
### Requirements

- Any flavour of PHP 5.3+ should do
- [optional] PHPUnit to execute the test suite

<a name="install-composer"/>
### With Composer

The easiest way to install Datum is via [composer](http://getcomposer.org/). Create the following `composer.json` file and run the `php composer.phar install` command to install it.

```json
{
    "require": {
        "dandoescode/datum": "*"
    }
}
```

``` php
<?php
require 'vendor/autoload.php';

use Datum\Datum;

printf("Now: %s", Datum::now());
```

<a name="install-nocomposer"/>
### Without Composer

Why are you not using [composer](http://getcomposer.org/)? Download [Datum.php](https://github.com/briannesbitt/Datum/blob/master/Datum/Datum.php) from the repo and save the file into your project path somewhere.

```php
<?php
require 'path/to/Datum.php';

use Datum\Datum;

printf("Now: %s", Datum::now());
```

<a name="api"/>
## API

The Datum class is [inherited](http://php.net/manual/en/keyword.extends.php) from the PHP [DateTime](http://www.php.net/manual/en/class.datetime.php) class.

```php
<?php

namespace Datum;

class Datum extends \DateTime
{
    // code here
}
```

Datum has all of the functions inherited from the base DateTime class.  This approach allows you to access the base functionality if you see anything missing in Datum but is there in DateTime.

> **Note: I live in Ottawa, Ontario, Canada and if the timezone is not specified in the examples then the default of 'America/Toronto' is to be assumed.  Typically Ottawa is -0500 but when daylight savings time is on we are -0400.**

Special care has been taken to ensure timezones are handled correctly, and where appropriate are based on the underlying DateTime implementation.  For example all comparisons are done in UTC or in the timezone of the datetime being used.

```php
$dtToronto = Datum::createFromDate(2012, 1, 1, 'America/Toronto');
$dtVancouver = Datum::createFromDate(2012, 1, 1, 'America/Vancouver');

echo $dtVancouver->diffInHours($dtToronto); // 3
```

Also `is` comparisons are done in the timezone of the provided Datum instance.  For example my current timezone is -13 hours from Tokyo.  So `Datum::now('Asia/Tokyo')->isToday()` would only return false for any time past 1 PM my time.  This doesn't make sense since `now()` in tokyo is always today in Tokyo.  Thus the comparison to `now()` is done in the same timezone as the current instance.

<a name="api-instantiation"/>
### Instantiation

There are several different methods available to create a new instance of Datum.  First there is a constructor.  It overrides the [parent constructor](http://www.php.net/manual/en/datetime.construct.php) and you are best to read about the first parameter from the PHP manual and understand the date/time string formats it accepts.  You'll hopefully find yourself rarely using the constructor but rather relying on the explicit static methods for improved readability.

```php
$carbon = new Datum();                  // equivalent to Datum::now()
$carbon = new Datum('first day of January 2008', 'America/Vancouver');
echo get_class($carbon);                 // 'Datum\Datum'
```

You'll notice above that the timezone (2nd) parameter was passed as a string rather than a `\DateTimeZone` instance. All DateTimeZone parameters have been augmented so you can pass a DateTimeZone instance or a string and the timezone will be created for you.  This is again shown in the next example which also introduces the `now()` function.

```php
$now = Datum::now();

$nowInLondonTz = Datum::now(new DateTimeZone('Europe/London'));

// or just pass the timezone as a string
$nowInLondonTz = Datum::now('Europe/London');
```

To accompany `now()`, a few other static instantiation helpers exist to create widely known instances.  The only thing to really notice here is that `today()`, `tomorrow()` and `yesterday()`, besides behaving as expected, all accept a timezone parameter and each has their time value set to `00:00:00`.

```php
$now = Datum::now();
echo $now;                               // 2012-10-14 20:40:20
$today = Datum::today();
echo $today;                             // 2012-10-14 00:00:00
$tomorrow = Datum::tomorrow('Europe/London');
echo $tomorrow;                          // 2012-10-16 00:00:00
$yesterday = Datum::yesterday();
echo $yesterday;                         // 2012-10-13 00:00:00
```

The next group of static helpers are the `createXXX()` helpers. Most of the static `create` functions allow you to provide as many or as few arguments as you want and will provide default values for all others.  Generally default values are the current date, time or timezone.  Higher values will wrap appropriately but invalid values will throw an `InvalidArgumentException` with an informative message.  The message is obtained from an [DateTime::getLastErrors()](http://php.net/manual/en/datetime.getlasterrors.php) call.

```php
Datum::createFromDate($year, $month, $day, $tz);
Datum::createFromTime($hour, $minute, $second, $tz);
Datum::create($year, $month, $day, $hour, $minute, $second, $tz);
```

`createFromDate()` will default the time to now.  `createFromTime()` will default the date to today. `create()` will default any null parameter to the current respective value. As before, the `$tz` defaults to the current timezone and otherwise can be a DateTimeZone instance or simply a string timezone value.  The only special case for default values (mimicking the underlying PHP library) occurs when an hour value is specified but no minutes or seconds, they will get defaulted to 0.

```php
$xmasThisYear = Datum::createFromDate(null, 12, 25);  // Year defaults to current year
$Y2K = Datum::create(2000, 1, 1, 0, 0, 0);
$alsoY2K = Datum::create(1999, 12, 31, 24);
$noonLondonTz = Datum::createFromTime(12, 0, 0, 'Europe/London');

// A two digit minute could not be found
try { Datum::create(1975, 5, 21, 22, -2, 0); } catch(InvalidArgumentException $x) { echo $x->getMessage(); }
```

```php
Datum::createFromFormat($format, $time, $tz);
```

`createFromFormat()` is mostly a wrapper for the base php function [DateTime::createFromFormat](http://php.net/manual/en/datetime.createfromformat.php).  The difference being again the `$tz` argument can be a DateTimeZone instance or a string timezone value.  Also, if there are errors with the format this function will call the `DateTime::getLastErrors()` method and then throw a `InvalidArgumentException` with the errors as the message.  If you look at the source for the `createXX()` functions above, they all make a call to `createFromFormat()`.

```php
echo Datum::createFromFormat('Y-m-d H', '1975-05-21 22')->toDateTimeString(); // 1975-05-21 22:00:00
```

The final two create functions are for working with [unix timestamps](http://en.wikipedia.org/wiki/Unix_time).  The first will create a Datum instance equal to the given timestamp and will set the timezone as well or default it to the current timezone.  The second, `createFromTimestampUTC()`, is different in that the timezone will remain UTC (GMT).  The second acts the same as `Datum::createFromFormat('@'.$timestamp)` but I have just made it a little more explicit.  Negative timestamps are also allowed.

```php
echo Datum::createFromTimeStamp(-1)->toDateTimeString();                        // 1969-12-31 18:59:59
echo Datum::createFromTimeStamp(-1, 'Europe/London')->toDateTimeString();       // 1970-01-01 00:59:59
echo Datum::createFromTimeStampUTC(-1)->toDateTimeString();                     // 1969-12-31 23:59:59
```

You can also create a `copy()` of an existing Datum instance.  As expected the date, time and timezone values are all copied to the new instance.

```php
$dt = Datum::now();
echo $dt->diffInYears($dt->copy()->addYear());  // 1

// $dt was unchanged and still holds the value of Datum:now()
```

Finally, if you find yourself inheriting a `\DateTime` instance from another library, fear not!  You can create a `Datum` instance via a friendly `instance()` function.

```php
$dt = new \DateTime('first day of January 2008'); // <== instance from another API
$carbon = Datum::instance($dt);
echo get_class($carbon);                               // 'Datum\Datum'
echo $carbon->toDateTimeString();                      // '2008-01-01 00:00:00'
```

<a name="api-getters"/>
### Getters

The getters are implemented via PHP's `__get()` method.  This enables you to access the value as if it was a property rather than a function call.

```php
$dt = Datum::create(2012, 9, 5, 23, 26, 11);

// These getters specifically return integers, ie intval()
var_dump($dt->year);                                   // int(2012)
var_dump($dt->month);                                  // int(9)
var_dump($dt->day);                                    // int(5)
var_dump($dt->hour);                                   // int(23)
var_dump($dt->minute);                                 // int(26)
var_dump($dt->second);                                 // int(11)
var_dump($dt->dayOfWeek);                              // int(3)
var_dump($dt->dayOfYear);                              // int(248)
var_dump($dt->weekOfYear);                             // int(36)
var_dump($dt->daysInMonth);                            // int(30)
var_dump($dt->timestamp);                              // int(1346901971)
var_dump(Datum::createFromDate(1975, 5, 21)->age);    // int(37) calculated vs now in the same tz
var_dump($dt->quarter);                                // int(3)

// Returns an int of seconds difference from UTC (+/- sign included)
var_dump(Datum::createFromTimestampUTC(0)->offset);   // int(0)
var_dump(Datum::createFromTimestamp(0)->offset);      // int(-18000)

// Returns an int of hours difference from UTC (+/- sign included)
var_dump(Datum::createFromTimestamp(0)->offsetHours); // int(-5)

// Indicates if day light savings time is on
var_dump(Datum::createFromDate(2012, 1, 1)->dst);     // bool(false)

// Gets the DateTimeZone instance
echo get_class(Datum::now()->timezone);               // DateTimeZone
echo get_class(Datum::now()->tz);                     // DateTimeZone

// Gets the DateTimeZone instance name, shortcut for ->timezone->getName()
echo Datum::now()->timezoneName;                      // America/Toronto
echo Datum::now()->tzName;                            // America/Toronto
```

<a name="api-setters"/>
### Setters

The following setters are implemented via PHP's `__set()` method.  Its good to take note here that none of the setters, with the obvious exception of explicitly setting the timezone, will change the timezone of the instance.  Specifically, setting the timestamp will not set the corresponding timezone to UTC.

```php
$dt = Datum::now();

$dt->year = 1975;
$dt->month = 13;             // would force year++ and month = 1
$dt->month = 5;
$dt->day = 21;
$dt->hour = 22;
$dt->minute = 32;
$dt->second = 5;

$dt->timestamp = 169957925;  // This will not change the timezone

// Set the timezone via DateTimeZone instance or string
$dt->timezone = new DateTimeZone('Europe/London');
$dt->timezone = 'Europe/London';
$dt->tz = 'Europe/London';
```

<a name="api-settersfluent"/>
### Fluent Setters

No arguments are optional for the setters, but there are enough variety in the function definitions that you shouldn't need them anyway.  Its good to take note here that none of the setters, with the obvious exception of explicitly setting the timezone, will change the timezone of the instance.  Specifically, setting the timestamp will not set the corresponding timezone to UTC.

```php
$dt = Datum::now();

$dt->year(1975)->month(5)->day(21)->hour(22)->minute(32)->second(5)->toDateTimeString();
$dt->setDate(1975, 5, 21)->setTime(22, 32, 5)->toDateTimeString();
$dt->setDateTime(1975, 5, 21, 22, 32, 5)->toDateTimeString();

$dt->timestamp(169957925)->timezone('Europe/London');

$dt->tz('America/Toronto')->setTimezone('America/Vancouver');
```

<a name="api-isset"/>
### IsSet

The PHP function `__isset()` is implemented.  This was done as some external systems (ex. [Twig](http://twig.sensiolabs.org/doc/recipes.html#using-dynamic-object-properties)) validate the existence of a property before using it.  This is done using the `isset()` or `empty()` method.  You can read more about these on the PHP site: [__isset()](http://www.php.net/manual/en/language.oop5.overloading.php#object.isset), [isset()](http://www.php.net/manual/en/function.isset.php), [empty()](http://www.php.net/manual/en/function.empty.php).

```php
var_dump(isset(Datum::now()->iDoNotExist));       // bool(false)
var_dump(isset(Datum::now()->hour));              // bool(true)
var_dump(empty(Datum::now()->iDoNotExist));       // bool(true)
var_dump(empty(Datum::now()->year));              // bool(false)
```

<a name="api-formatting"/>
### Formatting and Strings

All of the available `toXXXString()` methods rely on the base class method [DateTime::format()](http://php.net/manual/en/datetime.format.php).  You'll notice the `__toString()` method is defined which allows a Datum instance to be printed as a pretty date time string when used in a string context.

```php
$dt = Datum::create(1975, 12, 25, 14, 15, 16);

var_dump($dt->toDateTimeString() == $dt);          // bool(true) => uses __toString()
echo $dt->toDateString();                          // 1975-12-25
echo $dt->toFormattedDateString();                 // Dec 25, 1975
echo $dt->toTimeString();                          // 14:15:16
echo $dt->toDateTimeString();                      // 1975-12-25 14:15:16
echo $dt->toDayDateTimeString();                   // Thu, Dec 25, 1975 2:15 PM

// ... of course format() is still available
echo $dt->format('l jS \\of F Y h:i:s A');         // Thursday 25th of December 1975 02:15:16 PM
```

<a name="api-commonformats"/>
## Common Formats

The following are wrappers for the common formats provided in the [DateTime class](http://www.php.net/manual/en/class.datetime.php).

```php
$dt = Datum::now();

echo $dt->toATOMString();        // same as $dt->format(DateTime::ATOM);
echo $dt->toCOOKIEString();
echo $dt->toISO8601String();
echo $dt->toRFC822String();
echo $dt->toRFC850String();
echo $dt->toRFC1036String();
echo $dt->toRFC1123String();
echo $dt->toRFC2822String();
echo $dt->toRFC3339String();
echo $dt->toRSSString();
echo $dt->toW3CString();
```

<a name="api-comparison"/>
### Comparison

Simple comparison is offered up via the following functions.  Remember that the comparison is done in the UTC timezone so things aren't always as they seem.

```php
$first = Datum::create(2012, 9, 5, 23, 26, 11);
$second = Datum::create(2012, 9, 5, 20, 26, 11, 'America/Vancouver');

echo $first->toDateTimeString();                   // 2012-09-05 23:26:11
echo $second->toDateTimeString();                  // 2012-09-05 20:26:11

var_dump($first->eq($second));                     // bool(true)
var_dump($first->ne($second));                     // bool(false)
var_dump($first->gt($second));                     // bool(false)
var_dump($first->gte($second));                    // bool(true)
var_dump($first->lt($second));                     // bool(false)
var_dump($first->lte($second));                    // bool(true)

$first->setDateTime(2012, 1, 1, 0, 0, 0);
$second->setDateTime(2012, 1, 1, 0, 0, 0);         // Remember tz is 'America/Vancouver'

var_dump($first->eq($second));                     // bool(false)
var_dump($first->ne($second));                     // bool(true)
var_dump($first->gt($second));                     // bool(false)
var_dump($first->gte($second));                    // bool(false)
var_dump($first->lt($second));                     // bool(true)
var_dump($first->lte($second));                    // bool(true)
```

To handle the most used cases there are some simple helper functions that hopefully are obvious from their names.  For the methods that compare to `now()` (ex. isToday()) in some manner the `now()` is created in the same timezone as the instance.

```php
$dt = Datum::now();

$dt->isWeekday();
$dt->isWeekend();
$dt->isYesterday();
$dt->isToday();
$dt->isTomorrow();
$dt->isFuture();
$dt->isPast();
$dt->isLeapYear();
```

<a name="api-addsub"/>
### Addition and Subtraction

The default DateTime provides a couple of different methods for easily adding and subtracting time.  There is `modify()`, `add()` and `sub()`.  `modify()` takes a *magical* date/time format string, 'last day of next month', that it parses and applies the modification while `add()` and `sub()` use a `DateInterval` class thats not so obvious, `new \DateInterval('P6YT5M')`.  Hopefully using these fluent functions will be more clear and easier to read after not seeing your code for a few weeks.  But of course I don't make you choose since the base class functions are still available.

```php
$dt = Datum::create(2012, 1, 31, 0);

echo $dt->toDateTimeString();            // 2012-01-31 00:00:00

echo $dt->addYears(5);                   // 2017-01-31 00:00:00
echo $dt->addYear();                     // 2018-01-31 00:00:00
echo $dt->subYear();                     // 2017-01-31 00:00:00
echo $dt->subYears(5);                   // 2012-01-31 00:00:00

echo $dt->addMonths(60);                 // 2017-01-31 00:00:00
echo $dt->addMonth();                    // 2017-03-03 00:00:00 equivalent of $dt->month($dt->month + 1); so it wraps
echo $dt->subMonth();                    // 2017-02-03 00:00:00
echo $dt->subMonths(60);                 // 2012-02-03 00:00:00

echo $dt->addDays(29);                   // 2012-03-03 00:00:00
echo $dt->addDay();                      // 2012-03-04 00:00:00
echo $dt->subDay();                      // 2012-03-03 00:00:00
echo $dt->subDays(29);                   // 2012-02-03 00:00:00

echo $dt->addWeekdays(4);                // 2012-02-09 00:00:00
echo $dt->addWeekday();                  // 2012-02-10 00:00:00
echo $dt->subWeekday();                  // 2012-02-09 00:00:00
echo $dt->subWeekdays(4);                // 2012-02-03 00:00:00

echo $dt->addWeeks(3);                   // 2012-02-24 00:00:00
echo $dt->addWeek();                     // 2012-03-02 00:00:00
echo $dt->subWeek();                     // 2012-02-24 00:00:00
echo $dt->subWeeks(3);                   // 2012-02-03 00:00:00

echo $dt->addHours(24);                  // 2012-02-04 00:00:00
echo $dt->addHour();                     // 2012-02-04 01:00:00
echo $dt->subHour();                     // 2012-02-04 00:00:00
echo $dt->subHours(24);                  // 2012-02-03 00:00:00

echo $dt->addMinutes(61);                // 2012-02-03 01:01:00
echo $dt->addMinute();                   // 2012-02-03 01:02:00
echo $dt->subMinute();                   // 2012-02-03 01:01:00
echo $dt->subMinutes(61);                // 2012-02-03 00:00:00

echo $dt->addSeconds(61);                // 2012-02-03 00:01:01
echo $dt->addSecond();                   // 2012-02-03 00:01:02
echo $dt->subSecond();                   // 2012-02-03 00:01:01
echo $dt->subSeconds(61);                // 2012-02-03 00:00:00

$dt = Datum::create(2012, 1, 31, 12, 0, 0);
echo $dt->startOfDay();                  // 2012-01-31 00:00:00

$dt = Datum::create(2012, 1, 31, 12, 0, 0);
echo $dt->endOfDay();                    // 2012-01-31 23:59:59

$dt = Datum::create(2012, 1, 31, 12, 0, 0);
echo $dt->startOfMonth();                // 2012-01-01 00:00:00

$dt = Datum::create(2012, 1, 31, 12, 0, 0);
echo $dt->endOfMonth();                  // 2012-01-31 23:59:59
```

For fun you can also pass negative values to `addXXX()`, in fact that's how `subXXX()` is implemented.

<a name="api-difference"/>
### Difference

These functions always return the **total difference** expressed in the specified time requested.  This differs from the base class `diff()` function where an interval of 61 seconds would be returned as 1 minute and 1 second via a `DateInterval` instance.  The `diffInMinutes()` function would simply return 1.  All values are truncated and not rounded.  Each function below has a default first parameter which is the Datum instance to compare to, or null if you want to use `now()`.  The 2nd parameter again is optional and indicates if you want the return value to be the absolute value or a relative value that might have a `-` (negative) sign if the passed in date is less than the current instance.  This will default to true, return the absolute value.  The comparisons are done in UTC.

```php
// Datum::diffInYears(Datum $dt = null, $abs = true)

echo Datum::now('America/Vancouver')->diffInSeconds(Datum::now('Europe/London')); // 0

$dtOttawa = Datum::createFromDate(2000, 1, 1, 'America/Toronto');
$dtVancouver = Datum::createFromDate(2000, 1, 1, 'America/Vancouver');
echo $dtOttawa->diffInHours($dtVancouver);                             // 3

echo $dtOttawa->diffInHours($dtVancouver, false);                      // 3
echo $dtVancouver->diffInHours($dtOttawa, false);                      // -3

$dt = Datum::create(2012, 1, 31, 0);
echo $dt->diffInDays($dt->copy()->addMonth());                         // 31
echo $dt->diffInDays($dt->copy()->subMonth(), false);                  // -31

$dt = Datum::create(2012, 4, 30, 0);
echo $dt->diffInDays($dt->copy()->addMonth());                         // 30
echo $dt->diffInDays($dt->copy()->addWeek());                          // 7

$dt = Datum::create(2012, 1, 1, 0);
echo $dt->diffInMinutes($dt->copy()->addSeconds(59));                  // 0
echo $dt->diffInMinutes($dt->copy()->addSeconds(60));                  // 1
echo $dt->diffInMinutes($dt->copy()->addSeconds(119));                 // 1
echo $dt->diffInMinutes($dt->copy()->addSeconds(120));                 // 2

// others that are defined
// diffInYears(), diffInMonths(), diffInDays()
// diffInHours(), diffInMinutes(), diffInSeconds()
```

<a name="api-humandiff"/>
### Difference for Humans

It is easier for humans to read `1 month ago` compared to 30 days ago.  This is a common function seen in most date libraries so I thought I would add it here as well.  It uses approximations for month being 30 days which then equates a year to 360 days.  The lone argument for the function is the other Datum instance to diff against, and of course it defaults to `now()` if not specified.

This method will add a phrase after the difference value relative to the instance and the passed in instance.  There are 4 possibilities:

* When comparing a value in the past to default now:
    * 1 hour ago
    * 5 months ago

* When comparing a value in the future to default now:
    * 1 hour from now
    * 5 months from now

* When comparing a value in the past to another value:
    * 1 hour before
    * 5 months before

* When comparing a value in the future to another value:
    * 1 hour after
    * 5 months after

```php
// The most typical usage is for comments
// The instance is the date the comment was created and its being compared to default now()
echo Datum::now()->subDays(5)->diffForHumans();               // 5 days ago

echo Datum::now()->diffForHumans(Datum::now()->subYear());   // 1 year after

$dt = Datum::createFromDate(2011, 2, 1);

echo $dt->diffForHumans($dt->copy()->addMonth());              // 28 days before
echo $dt->diffForHumans($dt->copy()->subMonth());              // 1 month after

echo Datum::now()->addSeconds(5)->diffForHumans();            // 5 seconds from now
```

<a name="api-constants"/>
### Constants

The following constants are defined in the Datum class.

* SUNDAY = 0
* MONDAY = 1
* TUESDAY = 2
* WEDNESDAY = 3
* THURSDAY = 4
* FRIDAY = 5
* SATURDAY = 6
* MONTHS_PER_YEAR = 12
* HOURS_PER_DAY = 24
* MINUTES_PER_HOUR = 60
* SECONDS_PER_MINUTE = 60

```php
$dt = Datum::createFromDate(2012, 10, 6);
if ($dt->dayOfWeek === Datum::SATURDAY) {
    echo 'Place bets on Ottawa Senators Winning!';
}
```

<a name="about"/>
## About

<a name="about-author"/>
### Author

Dan Horrigan - <me@dandoescode.com> - <http://twitter.com/dandoescode

Original Author: Brian Nesbitt - <brian@nesbot.com> - <http://twitter.com/NesbittBrian>

<a name="about-license"/>
### License

Datum is licensed under the MIT License - see the `LICENSE` file for details

<a name="about-history"/>
### History

You can view the history of the Datum project in the [history file](https://github.com/dandoescode/datum/blob/master/history.md).

<a name="about-whyname"/>
### Why the name Datum?

Datum translated from several languages to English is *Date*.
