---
layout: post
title: Creating a Mock IDbSet using Moq
description: >
  In this post I will demonstrate creating a mock IDbSet for unit testing with Moq
redirect_from:
   - /post/creating-a-mock-idbset-using-moq
---

When using Entity Framework for a project I needed to be able to mock the database context and sets in order to do unit testing. I was able to find some ways to mock the Entity Framework context, but I did not find a complete solution. Most examples did not take into account setting the identity column or deleting items. In order to handle these scenarios I created my own MockDbSet.

I will demonstrate how this works with a sample project for creating and editing reminders.

The context is represented by a simple interface:

```csharp
public interface IRemindersContext
{
    IDbSet<Reminder> Reminders { get; set; }

    Task<int> SaveChangesAsync();
}
```

The business logic is in a manager class that takes the context interface as a parameter to enable dependency injection and unit testing:

```csharp
public class ReminderManager : IReminderManager
{
    IRemindersContext _context;
    public ReminderManager(IRemindersContext context)
    {
        _context = context;
    }

    public List<Reminder> Get()
    {
        return _context.Reminders.ToList();
    }

    public Reminder Get(int reminderId)
    {
        return _context.Reminders.Find(reminderId);
    }

    public async Task<Reminder> Add(string description)
    {
        var reminder = new Reminder { Description = description };

        _context.Reminders.Add(reminder);

        await _context.SaveChangesAsync();

        return reminder;
    }

    public async Task<bool> Delete(int reminderId)
    {
        var reminder = _context.Reminders.Find(reminderId);

        _context.Reminders.Remove(reminder);

        var result = await _context.SaveChangesAsync();

        return result > 0;
    }
}
```

In order to do unit testing of the business logic I need to be able to mock the IDbSet. This is where my mock class comes in. I am using Moq to do create the mocks. The mock IDbSet will be created by a static method on a static class. The method will accept a generic list containing the data that should be returned by the mock IDbSet. The class and method look like this:

```csharp
public static class MockDbSet
{
    public static Mock<IDbSet<T>> CreateMockDbSet<T>(List<T> data) where T : class
    {
        ...
    }
}
```

The first thing I need to do is to set up the mock to behave like an IQueryable<T>. To do this I will create the mock object mock some of the IQueryable methods.

```csharp
var mock = new Mock<IDbSet<T>>();
var queryData = data.AsQueryable();
mock.As<IQueryable<T>>().Setup(m => m.Provider).Returns(queryData.Provider);
mock.As<IQueryable<T>>().Setup(m => m.Expression).Returns(queryData.Expression);
mock.As<IQueryable<T>>().Setup(m => m.ElementType).Returns(queryData.ElementType);
mock.As<IQueryable<T>>().Setup(m => m.GetEnumerator()).Returns(queryData.GetEnumerator());
```

This is all that is needed in order to create a mock that will allow you to select items from it. I want to have add and remove functionality so I need to add those methods to my mock. First, I will find the primary key by using the class name with the suffix “Id”.

```csharp
Type type = typeof(T);
string colName = type.Name + "Id";
var pk = type.GetProperty(colName);
if (pk == null)
{
    colName = type.Name + "ID";
    pk = type.GetProperty(colName);
}
```

Note: Before using the pk you must validate that it is not null.

Now that I have the primary key and I create a method to add new items to the mock IDbSet and return them with the new ID. My implementation supports integer and Guid primary keys.

```csharp
mock.Setup(x => x.Add(It.IsAny<T>())).Returns((T x) =>
{
    if (pk.PropertyType == typeof(int)
        || pk.PropertyType == typeof(Int32))
    {
        var max = data.Select(d => (int)pk.GetValue(d)).Max();
        pk.SetValue(x, max + 1);
    }
    else if (pk.PropertyType == typeof(Guid))
    {
        pk.SetValue(x, Guid.NewGuid());
    }
    data.Add(x);
    return x;
});
```

The code for remove is much simpler:

```csharp
mock.Setup(x => x.Remove(It.IsAny<T>())).Returns((T x) =>
{
    data.Remove(x);
    return x;
});
```

I also need to support Find for my application. This turns out to be the trickiest method to implement. The difficult part is determining the primary key column and using it in a LINQ statement. I already know the primary key column so I will reuse that variable to write the query to find the item. To do this I create an expression tree to build the lambda expression that matches the given value to the primary key.

```csharp
mock.Setup(x => x.Find(It.IsAny<object[]>())).Returns((object[] id) =>
{
    var param = Expression.Parameter(type, "t");
    var col = Expression.Property(param, colName);
    var body = Expression.Equal(col, Expression.Constant(id[0]));
    var lambda = Expression.Lambda<Func<T, bool>>(body, param);
    return queryData.FirstOrDefault(lambda);
});
```

Here is the complete code for the MockDbSet:

```csharp
public static class MockDbSet
{
    public static Mock<IDbSet<T>> CreateMockDbSet<T>(List<T> data) where T : class
    {
        var mock = new Mock<IDbSet<T>>();
        var queryData = data.AsQueryable();
        mock.As<IQueryable<T>>().Setup(m => m.Provider).Returns(queryData.Provider);
        mock.As<IQueryable<T>>().Setup(m => m.Expression).Returns(queryData.Expression);
        mock.As<IQueryable<T>>().Setup(m => m.ElementType).Returns(queryData.ElementType);
        mock.As<IQueryable<T>>().Setup(m => m.GetEnumerator()).Returns(queryData.GetEnumerator());

        Type type = typeof(T);
        string colName = type.Name + "Id";
        var pk = type.GetProperty(colName);
        if (pk == null)
        {
            colName = type.Name + "ID";
            pk = type.GetProperty(colName);
        }
        if (pk != null)
        {
            mock.Setup(x => x.Add(It.IsAny<T>())).Returns((T x) =>
            {
                if (pk.PropertyType == typeof(int)
                    || pk.PropertyType == typeof(Int32))
                {
                    var max = data.Select(d => (int)pk.GetValue(d)).Max();
                    pk.SetValue(x, max + 1);
                }
                else if (pk.PropertyType == typeof(Guid))
                {
                    pk.SetValue(x, Guid.NewGuid());
                }
                data.Add(x);
                return x;
            });
            mock.Setup(x => x.Remove(It.IsAny<T>())).Returns((T x) =>
            {
                data.Remove(x);
                return x;
            });
            mock.Setup(x => x.Find(It.IsAny<object[]>())).Returns((object[] id) =>
            {
                var param = Expression.Parameter(type, "t");
                var col = Expression.Property(param, colName);
                var body = Expression.Equal(col, Expression.Constant(id[0]));
                var lambda = Expression.Lambda<Func<T, bool>>(body, param);
                return queryData.FirstOrDefault(lambda);
            });
        }

        return mock;
    }
}
```

In order to use this in unit tests I will create a mock context class that I can create an instance of for each unit test.

```csharp
public static class RemindersContextMock
{
    public static Mock<IRemindersContext> GetMockContext()
    {
        var context = new Mock<IRemindersContext>();

        context.Setup(c => c.Reminders).Returns(GetReminders().Object);

        return context;
    }

    public static Mock<IDbSet<Reminder>> GetReminders()
    {
        var reminders = new List<Reminder>
        {
            new Reminder { ReminderId = 1, Description = "Do work" },
            new Reminder { ReminderId = 2, Description = "Read a book" },
            new Reminder { ReminderId = 3, Description = "Make dinner" }
        };

        return MockDbSet.CreateMockDbSet<Reminder>(reminders);
    }
}
```

I will create a setup method for my unit tests where I create the instance of the manager class to test and pass in my mocked context.

```csharp
ReminderManager _manager;
Mock<IRemindersContext> _context;

[SetUp]
public void SetUp()
{
    _context = RemindersContextMock.GetMockContext();

    _manager = new ReminderManager(_context.Object);
}
```

Now I can write tests for the methods in the manager class. First I will verify that I can get the list of reminders from the context.

```csharp
[Test]
public void CanGetReminders()
{
    var reminders = _manager.Get();
    Assert.AreEqual(3, reminders.Count());
}
```

Next I will verify that when I add a reminder it gets a valid ID.

```csharp
[Test]
public async Task CanAddReminder()
{
    var reminder = await _manager.Add("Walk the dog");
    Assert.AreEqual(4, reminder.ReminderId);
}
```

Finally I will test the remove method to ensure that it is removing the reminder from the context.

```csharp
[Test]
public async Task CanDeleteReminder()
{
    var reminder = _manager.Get().FirstOrDefault(r => r.Description == "Read a book");
    await _manager.Delete(reminder.ReminderId);
    var reminder2 = _manager.Get(reminder.ReminderId);
    Assert.IsNull(reminder2);
}
```

I hope this helps you if you need to create a mock IDbSet. It should be clear now how to add to the mock DbSet’s methods if you need additional functionality.
