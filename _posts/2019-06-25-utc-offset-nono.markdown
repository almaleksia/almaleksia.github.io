---
layout: post
title: "UTC offset is not enough"
date: 2019-06-25 22:11:49 +0200
---

Recently I've run into one thread on Reddit, from nine years ago. Topic starter asked why there's no Time-Zone header in HTTP. First response liked by dozens of people was "How about if everyone just exchanges UTC and the local machine calculates its own local time? That's how **well-designed** systems work today." As I've got recently some experience working with well-designed systems, I decided to put together my chaotic thoughts on this topic.


### Time zone formats and their differences

There are two main time zone formats: IANA and Windows. If you see something like "Europe/Berlin" time zone name, it is IANA time zone standard. If you see "Pacific Standard Time" or "W. Europe Standard Time" - this time zone belongs to the standard historically used by Windows. Be ready to support both formats on the server side if you design your API to work with Windows desktop apps as well as with Android and iOS apps. 

By "supporting" I mean correctly interpreting requests with these time zone formats and returning time zone that client would understand. Many platforms do not have convenient way to work with Windows time zones, and C#, on the other hand, supports natively only Windows time zones. Be aware of these limitations. 

The most complicated part comes when you need to perform time zone mapping. IANA can be mapped to Windows equivalent without ambiguity because IANA standard is more detailed and contains more time zones. The opposite is trickier - Windows time zone can be mapped to several IANA time zones. Therefore, if your API usage patterns require you to actually store time zone (I'll touch this topic later in more details), better do it in IANA format.

### What does "time zone" actually mean

Time zone aware date and time have two components: local time and time zone.
The following examples describe the same time. 
{% highlight bash %}
2019-06-20T13:00:00 Europe/Berlin
2019-06-20T04:00:00 Pacific Standard Time
{% endhighlight  %}

 What is hidden behind "Europe/Berlin" or "Pacific Standard Time"? In short, set of rules that must be applied when converting date time between time zones. The rules include:
1. Time zone UTC offset (time difference between given time zone and Universal Coordinated Time)
2. Exact date and time when daylight shift takes place.
3. History of time zone changes like eliminating and restoring back daylight saving time (some politicians like to do such things because it's the only impact they are able to make on people's lives).

Below is an excerpt from IANA time zone database for Europe/Moscow time zone showing how offset and rules applied for daylight shift were changing over time.

{% highlight bash %}
Zone Europe/Moscow   2:30:17   -	    LMT	    1880
			               2:30:17   -	    MMT	    1916 Jul  3 # Moscow Mean Time
			               2:31:19   Russia	%s	    1919 Jul  1  0:00u
			               3:00	     Russia	%s	    1921 Oct
			               3:00	     Russia	MSK/MSD	1922 Oct
			               2:00	     -	    EET	    1930 Jun 21
			               3:00	     Russia	MSK/MSD	1991 Mar 31  2:00s
			               2:00	     Russia	EE%sT	  1992 Jan 19  2:00s 
			               3:00	     Russia	MSK/MSD	2011 Mar 27  2:00s
			               4:00	     -	    MSK	    2014 Oct 26  2:00s
			               3:00	     -    	MSK
{% endhighlight  %}

Now it's clear that time zone does not only represent UTC offset but many other things. The result of a conversion depends on the period given date falls into. Is it daylight saving time? Is it a period when daylight saving time was abolished? Thankfully, datetime libraries were invented to handle this, we just need to learn how to use what they provide in a right way.

### Everything is UTC offset

Imagine you have a calendar that your distributed team uses to arrange meetings. Employees may be located in different time zones but all of them need to attend meeting at the same time. Esther in Berlin books a slot to have a call with her colleauges from Seattle at 5 o'clock in the evening, 20th of June 2019 Berlin time. We know that there's 9 hour difference in the summer between Berlin time and Seattle time. That means employees in Seattle need to get invite for 8 o'clock in the morning.

In such cases it is often preferred to store date time in a "neutral" time zone - Universal Coordinated Time (UTC). Start time of the meeting will be stored as 2019-06-20T15:00:00 UTC on server side. Calendar client application located in Berlin will convert UTC to Europe/Berlin time, which is 2019-06-20T17:00:00 and client in Seattle - to pacific time: 2019-06-20T08:00:00. Seems that it works! Everyone will attend meeting at the same time.

API request may look like this:

{% highlight json %}
...
  "StartTime":{ 
    "DateTime":"2019-06-20T17:00:00",
    "Timezone":"Europe/Berlin"
   }
...   
{% endhighlight %}

Or like this (for clients that prefer windows system time zones):

{% highlight json %}
...
  "StartTime":{ 
    "DateTime":"2019-06-20T17:00:00",
    "Timezone":"W. Europe Standard Time"
   }
...   
{% endhighlight %}

The only thing server should do is to convert given local date time to UTC date and time and store it in database.
When client app requests calendar event it will be returned in UTC, and should be converted to local time zone again - locally. That's easy because client application is always aware of user's time zone.

### You just got fooled

Now, when UTC offset seems like legitimate way to handle time zones, let's see why it's not and you should, no, must store local time and local time zone instead.

Here they are - "highly hypothetical" requirements, that you won't be able to fulfill if you rely only on UTC offset.

1. Client app developers want to know local time for their time zone independent scenarios. Sometimes users don't need time zone adjustments. If I set a reminder to do my workout at 8pm, it will be always 8pm - no matter I'm in New York or in Moscow. Would be weird if I opened my reminder app after Moscow -> New York flight and found my reminder suddenly shifted to the afternoon. Reminder fixed to local time fits this use case much better.

2. Background analytics jobs. I want to know for what time of day users create more events. UTC offsets are completely useless for this task unless local time is present.

3. Integration with external systems. Some API implement their own (not always optimal) approach to dates. For example, there are APIs that are not timezone aware and do not accept UTC offsets as input. For such cases you need local date time. Some APIs are time zone aware but still do not accept UTC offsets, for example [Outlook Task REST API][outlook-api]. You need local time zone to update due date of a task.

Main take away from this: UTC offset does not replace local time with time zone and ignoring this fact will make your PM cry. And you don't want that. Remember that time is a very subjective matter and people have different perception of how it should work. So just make sure you have all data you need in place and be prepared for new datetime challenges.

[outlook-api]: https://docs.microsoft.com/en-us/previous-versions/office/office-365-api/api/version-2.0/task-rest-operations#specifying-the-startdatetime-and-duedatetime-properties
