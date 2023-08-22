---
title: "NSubstitute from Moq"
seoTitle: "Moq NSubstitute Alternative"
seoDescription: "Transition from Moq to NSubstitute: concise guide with examples for mocking, returning values, and verifying method calls in both libraries"
datePublished: Tue Aug 22 2023 07:52:07 GMT+0000 (Coordinated Universal Time)
cuid: cllm0csjm000c09l5e5qmekl4
slug: nsubstitute-from-moq
tags: unit-testing, unit-tests, mocking, moq, nsubstitute

---

A concise and straightforward guide to transitioning from Moq to NSubstitute.

## Foreword

There has been a recent issue with version 4.2.0 of the popular mocking library [Moq](https://github.com/moq/moq). This is covered in many places by the community already, there is a [great video by Nick Chapsas](https://www.youtube.com/watch?v=A06nNjBKV7I&t=312s) here if you prefer to read, [this blog post](https://snyk.io/blog/moq-package-exfiltrates-user-emails/) is good as well.

It is worth noting that the issues explained above have since been reverted and are currently not an issue, however, there is a large shift for new and existing projects to look at using alternatives for mocking, [NSubstitute](https://nsubstitute.github.io/) being one of them.

The post aims to give a simple guide showing examples of performing the same type of mocking in both libraries to hopefully aid a conversion if required.

## The Example Solution

The example solution is used to demonstrate the differences between Moq and NSubstitute. It is supposed to simulate a very simple order processing pipeline. It focuses solely on the business logic and does not implement any external dependencies such as messaging services or databases. These are expressed as interfaces and these will be mocked using the two mocking libraries.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">The full code solution can be found on <a target="_blank" rel="noopener noreferrer nofollow" href="https://github.com/mumby0168/moq-vs-nsubstitute" style="pointer-events: none">GitHub</a>.</div>
</div>

The majority of this post will be showing the differences between the two libraries while unit testing the below piece of code:

```csharp
// OrderService.Place.cs
namespace OrderProcessor.Application.Services;

public partial class OrderService
{
    public async Task<Guid> PlaceOrderAsync(
        PlaceOrderDto dto)
    {
        var (customerId, orderLines) = dto;

        var orderId = _idGenerator.NewOrderId();

        var lines = orderLines.Select(
                x => new OrderLine(
                    _idGenerator.NewOrderLineId(),
                    x.ProductId,
                    x.Price))
            .ToList();

        var expectedDeliveryDate = _orderDateService
            .CalculateExpectedOrderDate(
                customerId,
                lines);

        var order = new Order(
            orderId,
            customerId,
            expectedDeliveryDate,
            lines);

        await _orderRepository.SaveAsync(order);
        await _eventPublisher.PublishOrderPlacedEventAsync(order);

        return orderId;
    }
}
```

This code has been written to use many injected services which are great candidates for being mocked, this will be demonstrated in the following sections.

## Examples

This section will work through a concise set of examples showing how you can achieve different mocking capabilities in both libraries.

### Creating Mocks

The first port of call for interacting with any mocking library is being able to create an instance of a set of mock objects. This is very simple in both libraries, see below:

```csharp
// Moq --> OrderServiceTests.cs
namespace OrderProcessor.MoqTests.Application;

public partial class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _orderRepository;
    private readonly Mock<IEventPublisher> _eventPublisher;
    private readonly Mock<IIdGenerator> _idGenerator;
    private readonly Mock<IOrderDateService> _orderDateService;

    public OrderServiceTests()
    {
        _orderRepository = new Mock<IOrderRepository>();
        _eventPublisher = new Mock<IEventPublisher>();
        _idGenerator = new Mock<IIdGenerator>();
        _orderDateService = new Mock<IOrderDateService>();
    }
    
    private IOrderService CreateSut() =>
        new OrderService(
            _orderRepository.Object,
            _eventPublisher.Object,
            _idGenerator.Object,
            _orderDateService.Object);
}
```

```csharp
// NSubstitute --> OrderServiceTests.cs
namespace OrderProcessor.NSubstituteTests.Application;

public partial class OrderServiceTests
{
    private readonly IOrderRepository _orderRepository;
    private readonly IEventPublisher _eventPublisher;
    private readonly IIdGenerator _idGenerator;
    private readonly IOrderDateService _orderDateService;

    public OrderServiceTests()
    {
        _orderRepository = Substitute.For<IOrderRepository>();
        _eventPublisher = Substitute.For<IEventPublisher>();
        _idGenerator = Substitute.For<IIdGenerator>();
        _orderDateService = Substitute.For<IOrderDateService>();
    }
    
    private IOrderService CreateSut() =>
        new OrderService(
            _orderRepository,
            _eventPublisher,
            _idGenerator,
            _orderDateService);
}
```

There are only a few key differences between creating objects using both libraries:

* Moq has an object that is created using the `new` keyword, whereas NSubsitute uses a static class to create its objects.
    
* Moq returns a sort of wrapper class around the real object, whereas NSubsitute returns the real interface, meaning there is no need for the `.Object` call as there is with Moq.
    

### Returning Values

It is possible to return values from both methods and properties in both mocking libraries, see below for some examples.

One of the most common tasks with mocking is returning values from a method, see how this is done below using both libraries when the method has no arguments.

**Methods with no arguments**

Setting up return values for method calls in both libraries differs quite a bit, see below:

```csharp
// Moq
namespace OrderProcessor.MoqTests.Application;

public partial class OrderServiceTests
{
    [Fact]
    public async Task PlaceOrderAsync_Order_ReturnsOrderId()
    {
        // Arrange
        var sut = CreateSut();

        var orderId = Guid.NewGuid();
        _idGenerator.Setup(x => x.NewOrderId()).Returns(orderId);

        var placeOrderDto = new PlaceOrderDto(
            Guid.NewGuid(),
            new List<PlaceOrderDto.OrderLineDto>
            {
                new(
                    Guid.NewGuid(),
                    10.0)
            });

        // Act
        var result = await sut.PlaceOrderAsync(placeOrderDto);

        // Assert
        result.Should().Be(orderId);
    }
}
```

```csharp
// NSubstitute
namespace OrderProcessor.NSubstituteTests.Application;

public partial class OrderServiceTests
{
    [Fact]
    public async Task PlaceOrderAsync_Order_ReturnsOrderId()
    {
        // Arrange
        var sut = CreateSut();

        var orderId = Guid.NewGuid();
        _idGenerator.NewOrderId().Returns(orderId);

        var placeOrderDto = new PlaceOrderDto(
            Guid.NewGuid(),
            new List<PlaceOrderDto.OrderLineDto>
            {
                new(
                    Guid.NewGuid(),
                    10.0)
            });

        // Act
        var result = await sut.PlaceOrderAsync(placeOrderDto);

        // Assert
        result.Should().Be(orderId);
    }
}
```

The key differences here come back to the fact that Moq uses a wrapper object whereas NSubsitutes API starts with the real interface which in a lot of ways makes it easier. Moq require a `.Setup` method to be called before getting to the real interface that you are working with. Both, however, use a `Return` method off the back of the setup to provide your return value.

**Methods with arguments**

The setting up of methods with arguments is also quite different, see below:

```csharp
namespace OrderProcessor.MoqTests.Application;

public partial class OrderServiceTests
{
    [Fact]
    public async Task PlaceOrderAsync_Order_SavesOrderWithCorrectDeliveryDate()
    {
        // Arrange
        var sut = CreateSut();
        var customerId = Guid.NewGuid();

        _idGenerator.Setup(x => x.NewOrderId()).Returns(Guid.NewGuid);

        var orderDeliveryDate = DateTime.UtcNow.AddDays(1);

        _orderDateService.Setup(
                o => o.CalculateExpectedOrderDate(
                    customerId,
                    It.IsAny<IEnumerable<OrderLine>>()))
            .Returns(orderDeliveryDate);

        var placeOrderDto = new PlaceOrderDto(
            customerId,
            new List<PlaceOrderDto.OrderLineDto>
            {
                new(
                    Guid.NewGuid(),
                    10.0)
            });

        // Act
        await sut.PlaceOrderAsync(placeOrderDto);

        // Assert
        _orderRepository.Verify(
            o => o.SaveAsync(
                It.Is<Order>(
                    order =>
                        order.ExpectedDeliveryDate == orderDeliveryDate)),
            Times.Once);
    }
}
```

```csharp
namespace OrderProcessor.NSubstituteTests.Application;

public partial class OrderServiceTests
{
    [Fact]
    public async Task PlaceOrderAsync_Order_SavesOrderWithCorrectDeliveryDate()
    {
        // Arrange
        var sut = CreateSut();
        var customerId = Guid.NewGuid();

        _idGenerator.NewOrderId().Returns(Guid.NewGuid());

        var orderDeliveryDate = DateTime.UtcNow.AddDays(1);

        _orderDateService.CalculateExpectedOrderDate(
                customerId,
                Arg.Any<IEnumerable<OrderLine>>())
            .Returns(orderDeliveryDate);

        var placeOrderDto = new PlaceOrderDto(
            customerId,
            new List<PlaceOrderDto.OrderLineDto>
            {
                new(
                    Guid.NewGuid(),
                    10.0)
            });

        // Act
        await sut.PlaceOrderAsync(placeOrderDto);

        //Assert
        await _orderRepository.Received(1).SaveAsync(
            Arg.Is<Order>(
                o =>
                    o.ExpectedDeliveryDate == orderDeliveryDate));
    }
}
```

The interaction with setting up return values is very similar with or without arguments in both libraries, but there is extra syntax required to provide the expected arguments. In this example the `IOrderDateService` is set up to return a specific date based on solely a customer ID in these examples.

This is done in both libraries by just calling the method via the mocking library and passing the expected customer ID, and then using the respective `It` or `Arg` classes simply accept any order lines passed in as the second argument. These two static classes are the key difference really between the two in terms of accepting arguments when setting up mocked objects.

**Properties**

It is possible to set up return values from properties in both libraries as well, in a very similar fashion. The tests below do not test anything useful, they simply demonstrate how to set up return values of properties.

```csharp
namespace OrderProcessor.MoqTests.Domain;

public class OrderTests
{
    [Fact]
    public void Id_Mock_ReturnsMockedId()
    {
        // Arrange
        var id = Guid.NewGuid();
        var sut = new Mock<IOrder>();
        sut.SetupGet(o => o.Id).Returns(id);

        // Act
        var result = sut.Object.Id;

        // Assert
        result.Should().Be(id);
    }
}
```

```csharp
namespace OrderProcessor.NSubstituteTests.Domain;

public class OrderTests
{
    [Fact]
    public void Id_Mock_ReturnsMockedId()
    {
        // Arrange
        var id = Guid.NewGuid();
        var sut = Substitute.For<IOrder>();
        sut.Id.Returns(id);

        // Act
        var result = sut.Id;

        // Assert
        result.Should().Be(id);
    }
}
```

I think this example, shows clearly the more concise API that is provided by NSubstitute without the need for in some ways the noise provided by Moq.

### Verifying method calls

Verifying that a method has been called as part of a unit test is also a common task and is possible in both libraries again with quite a difference in syntax, this repeats a test from an earlier section, so for brevity, the calls to check that an order has been saved to the database has been extracted solely in the snippets below:

```csharp
[Fact]
public async Task PlaceOrderAsync_Order_SavesOrderWithCorrectDeliveryDate()
{
    // Excluded for brevity ...
    // Assert
    _orderRepository.Verify(
        o => o.SaveAsync(
            It.Is<Order>(
                order =>
                    order.ExpectedDeliveryDate == orderDeliveryDate)),
            Times.Once);
}
```

```csharp
[Fact]
public async Task PlaceOrderAsync_Order_SavesOrderWithCorrectDeliveryDate()
{
    // Excluded for brevity ...
    //Assert
    await _orderRepository.Received(1).SaveAsync(
            Arg.Is<Order>(
                o =>
                    o.ExpectedDeliveryDate == orderDeliveryDate));
}
```

As shown above, there are some key differences to the approach taken by each library, one thing that stands out, is how the amount of calls is specified to a given method. It's always a good idea to be explicit about the number of calls expected to a given method in a unit test. NSubstitute puts this up front as the first thing required via the `.Received()` call before getting at the method you want to check. Moq on the other hand has it as an optional argument that isn't always apparent to someone who is just starting with the library.

The ceremony involved in mock with the expressions and extra method calls can make these verify steps a little hard to read, whereas the concise syntax of NSubsitutue can be a lot easier to read.

Both libraries again make use of there respective `It` and `Arg` classes for checking arguments as shown in the examples above.

# Summary

In summary, NSubstitue's API surface is much closer to that of the interface that is being tested, which makes a big difference in not having to go through the ceremony of the additional method calls in something like Moq. The verifying of method calls is also a little cleaner in terms of the syntax when being precise about the number of calls made to a method, this is something that feels a little clunky in Moq.

In terms of a potential transition from Moq to NSubstitute, the differences are very minor and should be a trivial task in terms of complexity. Depending on the size of the code base, however, this may be a bit of a laborious task.