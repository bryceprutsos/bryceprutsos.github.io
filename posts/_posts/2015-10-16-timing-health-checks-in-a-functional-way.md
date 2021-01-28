---
layout: post
title: Timing Health Checks in a Functional Way
description: >
  In this post I will demonstrate timing health checks for web services using functional programming features in C#
redirect_from:
    - /post/timing-health-checks-in-a-functional-way
---

While creating a health check page for a service I realized that I was repeating code that timed each health check method. On the health check page there are several methods and each is responsible for checking one thing. The methods all return the same object which is defined by this class:

```csharp
public class HealthStatus
{
    public bool Status { get; set; }
    public long RequestTimeMs { get; set; }
    public string ErrorMessage { get; set; }
    //other properties
}
```

I decided to refactor the code so that I was not repeating the same code over and over. The style of programming I have been learning from functional programming gave me an idea of how to refactor the code in a much simpler way than I would have without using a functional approach. I created this method which takes a method, starts a timer, runs the method, then puts the time it took to run in the return value.

```csharp
public HealthStatus TimedHealthCheck(Func<HealthStatus> f)
{
    var sw = new Stopwatch();
    sw.Start();
    
    var result = f();
    
    result.RequestTimeMs = sw.ElapsedMilliseconds;
    
    return result;
}
```

Now I can just wrap the work inside my health check methods with this and it will do the timing for me. Here is an example:

```csharp
public HealthStatus CanParseString()
{
    return TimedHealthCheck(() =>
    {
        var health = new HealthStatus
        {
            Status = false
        };

        try
        {
            var parseInt = 0;
            health.Status = int.TryParse("123", out parseInt);
        }
        catch (Exception ex)
        {
            health.ErrorMessage = ex.Message;
        }

        return health;
    });
}
```

To run the health checks I create a list and add each health check:

```csharp
var healthChecks = new List<HealthStatus>();
    healthChecks.Add(CanParseString());
```
