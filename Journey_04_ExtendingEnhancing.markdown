## Chapter 4
# Extending and Enhancing the Orders and Registrations Bounded Contexts 

*Exploring further*

# A Description of the Orders and Registrations Bounded Context 

The Orders and Registrations Bounded Context was described in some detail 
in the previous chapter. This chapter describes some changes that the 
team made in this bounded context during the second stage of their CQRS 
journey. 

The specific topics described in this chapter include:

* Improvements to the way that message correlation works with the 
  **RegistrationProcess*. This illustrates how aggregate instances 
  within the bounded context can interact in a complex manner.
* Implementing a record locator to enable a registrant to retrieve an 
  order that was saved during a previous session. This illustrates
  adding some additional logic to the write-side that enables you to
  locate an aggregate instance without knowing its unique Id.
* Adding a countdown timer to the UI to enable a registrant to track how 
  much longer they have to complete an order. This illustrates
  enhancements to the write-side to support displaying rich information
  in the UI.
* Supporting orders for multiple seat types simultaneously. For example
  a registrant requests five seats for pre-conference event and eight
  seats for the full conference. This adds more complex business logic
  into the write-side.
* Supporting partially fulfilled orders where the only some of the
  seats that the registrant requests are available. This adds more
  complex business logic into the write-side.
* CQRS command validation using MVC. This illustrates how to make use
  of the model validation feature in MVC to validate your CQRS commands
  before you send them to the domain.

> **Note:** The Contoso Conference Management System described in this
> chapter is not the final version of the system. This guidance
> describes a journey, so some of the design decisions and
> implementation details change in later steps in the journey. These
> changes are described in subsequent chapters.

## Working Definitions for this Chapter 

The following definitions are used for the remainder of this chapter. 
For more detail, and possible alternative definitions see [A CQRS/ES 
Deep Dive][r_chapter4] in the Reference Guide. 

### Command

A command is a request for the system to perform an action that changes 
the state of the system. Commands are imperatives, for example 
**MakeSeatReservation**. In this bounded context, commands originate 
either from the UI as a result of a user initiating a request, or from 
a workflow when the workflow is directing an aggregate to perform an 
action. 

Commands are processed once by a single recipient. A command bus 
transports commands that command handlers then dispatch to aggregates. 
Sending a command is an asynchronous operation with no return value. 

### Event

An event describes something that has happened in the system, typically 
as a result of a command. Aggregates in the domain model raise events. 

Multiple subscribers can handle a specific event. Aggregates publish 
events to an event bus; handlers register for specific types of event on 
the event bus and then deliver the events to the subscriber. In this 
bounded context, the only subscriber is a workflow. 

### Coordinating Workflow

In this bounded context, a Coordinating Workflow (or workflow) is a 
class that coordinates the behavior of the aggregates in the domain. A 
workflow subscribes to the events that the aggregates raise, and then 
follow a simple set of rules to determine which command or commands to 
send. The workflow does not contain any business logic, simply logic to 
determine the next command to send. The workflow is implemented as a 
state machine, so when the workflow responds to an event, it can change 
its internal state in addition to sending a new command. 

The workflow in this bounded context can receive commands as well as 
subscribe to events. 

## User Stories 

This chapter discusses the implementation of two user stories in 
addition to describing some changes and enhancements to the **Orders and 
Registrations** bounded context. 

### Implement a Login using a Record Locator

When a registrant creates an order for seats at a conference, the system 
generates a five character **order access code** and sends it to the 
registrant by email. The registrant can use her email address and the 
**order access code** on the conference web site to retrieve the order 
from the system at a later date. The registrant may wish to retrieve the 
order to review it, or to complete the registration process by assigning 
attendees to seats. 

### Inform the Registrant How Much Time Remains to Complete an Order

When a registrant creates an order, the system reserves the seats 
requested by the registrant until the order is complete or the 
reservations expire. To complete an order, the registrant must submit 
her details, such as name and email address, and make a successful 
payment. 

To help the registrant, the system displays a count-down timer to inform 
the registrant how much time remains to complete the order before the 
seat reservations expire. 

### Enabling a Registrant to Create an Order that Includes Multiple Seat Types

When a registrant creates an order, the registrant may request different 
numbers of different seat types. For example, a registrant may request 
five seats for the full conference and three seats for the 
pre-conference workshop. 

### Handling Partially Fulfilled Orders

When a registrant creates an order, it may not be possible to completely 
fulfill the order. For example, a registrant may request five seats for the full conference, five seats for the 
drinks reception, and three seats for the pre-conference workshop. There 
may only be three seats available and one seat for the drinks reception, 
but more than three seats available for the pre-conference workshop. The 
system displays this information to the registrant and gives the 
registrant the opportunity to adjust the number of each type of seat in 
the order before continuing to the payment process. 

## Architecture 

The application is designed to deploy to Windows Azure. At this stage in 
the journey, the application consists of a web role that contains the 
MVC web application and a worker role that contains the message handlers 
and domain objects. The application uses SQL Azure databases for data 
storage, both on the write-side and the read-side. The application uses 
the Windows Azure Service Bus to provide its messaging infrastructure. 

While you are exploring and testing the solution, you can run it 
locally, either using the Windows Azure compute emulator or by running 
the MVC web application directly and running a console application that 
hosts the handlers and domain objects. When you run the application 
locally, you can use a local SQL Express database instead of SQL Azure, 
and use a simple messaging infrastructure implemented in a SQL Express 
database. 

For more information about the options for running the application, see 
[Appendix 1][appendix1]. 

# Patterns and Concepts 

* What are the primary patterns or approaches that have been adopted for this bounded context? (CQRS, CQRS/ES, CRUD, ...) 

* What were the motivations for adopting these patterns or approaches for this bounded context? 

* What trade-offs did we evaluate? 

* What alternatives did we consider? 

## Record Locators

The system uses **Access Codes** instead of passwords to avoid the 
overhead for the registrant of setting up an account with the system. 
Many registrants may use the system only once, so there is no need to 
create a permanent account with a user ID and a password. 

The system needs to be able to retrieve order information quickly based 
on the registrant's email address and access code. To provide a minimum 
level of security, the access codes that the system generates should not 
be predictable, and the order information that registrants can retrieve 
should not contain any sensitive information. 

## Querying the Read-side

The previous chapter focused on the write-side model and implementation, 
in this chapter we'll explore the read-side implementation in more 
detail. In particular you'll see how the team implemented the read model 
and the querying mechanism from the MVC controllers. 

In this initial exploration of the CQRS pattern, the team decided to use 
SQL views in the database as the underlying source of the data queried 
by the MVC controllers on the read-side. These views currently exist in 
the same database as the normalized tables that the write model uses. 

> **JanaPersona:** The team will split the database into two and explore
> options for pushing changes from the normalized write-side to the
> de-normalized read-side in a later stage of the journey.

### Storing De-normalized Views in a Database

One common option for storing the read-side data is to use a set of 
relational database tables to hold the de-normalized views. The 
read-side should be optimized for fast reads, so there is typically no 
benefit in storing normalized data because this will require complex 
queries to construct the data for the client. This implies that goals 
for the read-side should be to keep the queries as simple as possible, 
and to structure the tables in the database in such a way that they can 
be read quickly and efficiently. 

An important area for consideration is the interface whereby a client 
such as an MVC controller action submits a query to the read-side model. 

![Figure 1][fig1]

**The Read-side storing data in a relational database**

In figure 1, a client such as an MVC controller action invokes a method 
on a **ViewRepository** class to request the data that it needs. The 
**ViewRepository** class in turn runs a query against the de-normalized 
data in the database. 

The team at Contoso evaluated two approaches to implementing the **ViewRepository** class: using the **IQueryable** interface and using non-generic data access objects (DAOs).

#### Using the **IQueryable** Interface

One approach to consider for the **ViewRepository** class is to have it 
return an **IQueryable** instance that enables the client to use LINQ to 
specify its query. It is very easy to return an **IQueryable** instance 
from many ORMs such as Entity Framework or NHibernate. The following 
code snippet illustrates how the client can submit such queries. 

```Cs
var ordersummary = repository.Query<OrderSummary>().Where(LINQ query to retrieve order summary);
var orderdetails = repository.Query<OrderDetails>().Where(LINQ query to retrieve order details);
```

This approach has a number of advantages:

* **Simplicity #1.** This approach uses a thin abstraction layer over 
  the underlying database. It is supported by multiple ORMs and
  minimizes the amount of code that you must write. 
* **Simplicity #2.** You only need to define a single repository and a 
  single **Query** method. 
* **Simplicity #3.** You don't need a separate query object. On the 
  read-side the queries should be simple because you have already 
  de-normalized the data from the write-side to support the read-side
  clients. 
* **Simplicity #4.** You can make use of LINQ to provide support for 
  features such as filtering, paging, and sorting in the client. 
* **Testability.** You can use LINQ to Objects for mocking. 

> **MarkusPersona:** In the RI, using Entity Framework, we didn't need
> to write any code at all to expose the **IQueryable** instance. We
> also had just a single **ViewRepository** class. 

Possible objections to this approach include:

* It is not easy to replace the data store with a non-relational 
  database (that does not expose an **IQueryable** object. However, you
  can choose to implement the write-model differently in each bounded
  context using an approach that is appropriate to that bounded context. 
* The client might abuse the **IQueryable** interface be performing 
  operations that can be done more efficiently as a part of the 
  de-normalization process. You should ensure that the de-normalized
  data fully meets the requirements of the clients. 
* Using the **IQueryable** interface hides the queries away. However, 
  since you de-normalize the data from the write-side, the queries 
  against the relational database tables are unlikely to be complex. 
* It's hard to know if your integration tests cover all the different
  uses of the **Query** method.

#### Using Non-generic DAOs

An alternative approach is to have the **ViewRepository** expose custom 
**Find** and **Get** methods as shown in the following code snippets. 

```Cs
var ordersummary = dao.FindAllSummarizedOrders(userId);
var orderdetails = dao.GetOrderDetails(orderId);
```

You could also choose to use different DAO classes. This would make it 
easier to access different data sources. 

```Cs
var ordersummary = OrderSummaryDAO.FindAll(userId);
var orderdetails = OrderDetailsDAO.Get(orderId);
```

This approach has a number of advantages:

* **Simplicity #1.** Dependencies are clearer for the client. For 
  example, the client references an explicit **IOrderSummaryDAO**
  instance rather than a generic **IViewRepository** instance. 
* **Simplicity #2.** For the majority of queries, there are only one or 
  two predefined ways to access the object. Different queries typically 
  return different projections. 
* **Flexibility #1.** The **Get** and **Find** methods hide details such 
  as the partitioning of the data store and the data access methods such 
  as an ORM or executing SQL code explicitly. This makes it easier to 
  change these choices in the future. 
* **Flexibility #2.** The **Get** and **Find** methods could use an ORM, 
  LINQ, and the **IQueryable** interface behind the scenes to get the
  data from the data store. This is a choice that you could make on a
  method by method basis. 
* **Performance #1.** You can easily optimize the queries that the 
  **Find** and **Get** methods run. 
* **Performance #2.** The data access layer executes all queries. There 
  is no risk that the client MVC controller action tries to run complex 
  and inefficient LINQ queries against the data source. 
* **Testability.** It is easier to specify unit tests for the **Find** 
  and **Get** methods than to create suitable unit tests for the range
  of possible LINQ queries that a client could specify. 

Possible objections to this approach include:

* Using the **IQueryable** interface makes it much easier to use grids 
  that support features such as paging, filtering, and sorting in the
  UI.
  
The team decided to adopt the second approach. For examples, see the 
**ConferenceDao** and **OrderDao** classes in the **Registration** 
project. 

## Making Information about Partially Fulfilled Orders Available to the Read-side

The UI displays data about orders that it obtains by querying the model 
on the read-side. Part of the data that the UI displays to the 
registrant is information about partially fulfilled orders: for each 
seat type in the order, the number of seats requested and the number of 
seats that are available. This information is temporary data that is 
only used while the registrant is creating the order using the UI; the 
business only needs to store information about seats that were actually
purchased, not the difference between what the registrant requested and 
what the registrant purchased. 

The consequence of this is that the informationa about how many seats 
the registrant requested only needs to exist in the model on the 
read-side. 

> **JanaPersona:** You can't store this information in an HTTP session
> because the registrant may leave the site in between requesting the
> seats and completing the order.

A further consequence, is that the underlying storage on the read-side 
cannot be simple SQL views because it includes data that is not stored 
in the underlying table storage on the write-side. This means that this 
information must be passed to the read-side by using events. 

Figure 2 shows all the commands and events that the **Order** and 
**SeatsAvailability** aggregates use and how the **Order** aggregate 
pushes changes to the read-side by raising events. 

![Figure 2][fig2]

**The new architecture of the reservation process**

The **OrderPlaced**, **OrderUpdated**, **OrderPartiallyReserved**, 
**OrderRegistrantAssigned**, and **OrderReservationCompleted** events 
are handled by the **OrderViewModelGenerator** class that uses 
**DraftOrder** and **DraftOrderItem** instances to persist changes to the 
view tables. 

> **BharathPersona:** If you look ahead to the next chapter, [Preparing
> for the V1 Release][j_chapter5], you'll see that the team extended the
> use of events and migrated the **Orders and Registrations** bounded
> context to use event sourcing. 

## CQRS Command Validation

When you implement the write-model, you should try to ensure that 
commands very rarely fail. This gives the best user experience, and 
makes it much easier to implement the asynchronous behavior in your 
application. 

One approach, adopted by the team, is to use the model validation 
features in ASP.NET MVC 3. 

You should be careful to distinguish between errors and business
failures. Examples of errors include: 

* A message not being delivered due to a failure in the messaging
  infrastructure.
* Data not being persisted due to a connectivity problem with the
  database.

In many cases, especially in the cloud, you can handle these errors by
 retrying the operation.

A business failure should have a predetermined business response. For
 example:

* If a seat cannot be reserved because there are no seats left, then the
  system should add the request to a wait-list.
* If a credit card payment fails, the user should be given the chance to
  try a different card, or to set up payment by invoice.

> **BharathPersona:** Your domain experts should help you to identify
> possible business failures and determine the way that you handle
> them. 

## The Count-down Timer and the Read-model

The count-down timer that displays how much time remains to complete the 
order to the registrant is part of the business data in the system, and 
not just a part of the infrastructure. When a registrant creates an 
order and reserves seats, the count-down begins. The count-down 
continues, even if the registrant leaves the conference web site. The UI 
must be able to display the correct count-down value if the registrant 
returns to the site, therefore the reservation expiry time is a part of 
the data that is available from the read-model. 

# Implementation Details 

This section describes some of the significant features of the 
implementation of the Orders and Registrations bounded context that are 
described in this chapter. You may find it useful to have a copy of the 
code so you can follow along. You can download a copy of the code from 
the repository on github: [mspnp/cqrs-journey-code][repourl]. 

> **Note:** Do not expect the code samples to exactly match the code in
> the reference implementation. This chapter describes a step in the
> CQRS journey, the implementation may well change as we learn more and
> refactor the code.

## The Order Access Code Record Locator 

A registrant may need to retrieve an Order, either to view it, or to 
complete registering attendees to seats. This may happen in a different 
web session, so the registrant must supply some information to locate 
the previously saved order. 

The following code sample shows how the **Order** class generates an new 
five character order access code that is persisted as part of the 
**Order** instance. 

```Cs
public string AccessCode { get; set; }

protected Order()
{
	...
	this.AccessCode = HandleGenerator.Generate(5);
}
```

To retrieve an **Order** instance, a registrant must provide her email 
address and the order access code. The system will use these two items 
to locate the correct order. This logic is part of the read-side. 

The following code sample from the **OrderController** class in the web 
application shows how the MVC controller submits the query to the 
repository to discover the unique **OrderId** value. This **Find** 
action passes the **OrderId** value to a **Display** action that 
displays the order information to the registrant. 

```Cs
[HttpPost]
public ActionResult Find(string conferenceCode, string email, string accessCode)
{
	var repo = this.repositoryFactory();
	using (repo as IDisposable)
	{
		var order = repo.Query<DraftOrder>()
			.Where(o => o.RegistrantEmail == email && o.AccessCode == accessCode)
			.FirstOrDefault();

		if (order == null)
			return RedirectToAction("Find", new { conferenceCode = conferenceCode });

		return RedirectToAction("Display", new { conferenceCode = conferenceCode, orderId = order.OrderId });
	}
}
```

This illustrates two ways of querying the read-side. The first, as shown 
in the previous code sample, enables you to use a LINQ query against an 
**IQueryable** instance. The second, as used in the MVC controller's 
**Display** action, retrieves a single instance based on its unique Id. 
The following code sample shows the **Find** and **Query** methods in 
the **OrmViewRepository** class that the MVC controller uses. 

```Cs
public T Find<T>(Guid id) where T : class
{
	return this.Set<T>().Find(id);
}

public IQueryable<T> Query<T>() where T : class
{
	return this.Set<T>();
}
```

## The Count-down Timer

When a registrant creates and order and makes a seat reservation, those 
seats are reserved for a fixed period of time. The 
**RegistrationProcess** instance, which forwards the reservation from 
the **SeatsAvailability** aggregate, passes the time that the 
reservation expires to the **Order** aggregate. The following code 
sample shows how the **Order** aggregate receives and stores the 
reservation expiry time. 

```Cs
public DateTime? ReservationExpirationDate { get; private set; }

public void MarkAsReserved(DateTime expirationDate, IEnumerable<SeatQuantity> seats)
{
    ...

    this.ReservationExpirationDate = expirationDate;
    this.Items.Clear();
    this.Items.AddRange(seats.Select(seat => new OrderItem(seat.SeatType, seat.Quantity)));
}

```

> **MarkusPersona:** The **ReservationExpirationDate** is intially set
> in the **Order** constructor to a time 15 minutes after the **Order**
> is instantiated. This time may be revised by the
> **RegistrationProcess** workflow based on when the reservations
> are actually made. It is this time the workflow sends to the **Order**
> aggregate in the **MarkSeatsAsReserved** command.

The MVC **RegistrationController** class retrieves the order information 
on the read-side. The **DraftOrder** class includes the reservation expiry 
time that is then passed to the view using the **ViewBag** class, as 
shown in the following code sample. 

```Cs
[HttpGet]
public ActionResult SpecifyRegistrantDetails(string conferenceCode, Guid orderId)
{
	var repo = this.repositoryFactory();
	using (repo as IDisposable)
	{
		var draftOrder = repo.Find<DraftOrder>(orderId);
		var conference = repo.Query<Conference>()
			.Where(c => c.Code == conferenceCode)
			.FirstOrDefault();

		this.ViewBag.ConferenceName = conference.Name;
		this.ViewBag.ConferenceCode = conference.Code;
		this.ViewBag.ExpirationDateUTCMilliseconds = draftOrder.BookingExpirationDate.HasValue ? ((draftOrder.BookingExpirationDate.Value.Ticks - EpochTicks) / 10000L) : 0L;
		this.ViewBag.OrderId = orderId;

		return View(new AssignRegistrantDetails { OrderId = orderId });
	}
}
```

The MVC view then uses Javascript to display an animated count-down 
timer. 

## Using ASP.NET MVC 3 Validation for Commands

You should try to ensure that any commands that the MVC controllers in 
your application send to the write-model will succeed. You can use the 
features in MVC to validate the commands both client-side and 
server-side before sending them to the write-model. 

> **MarkusPersona:** Client-side validation is primarily a convenience
> to the user that avoids the need to for round trips to the server to
> help the user complete a form correctly. You still need server-side
> validation to ensure that the data is validated before it is forwarded
> to the write-model.

The following code sample shows the **AssignRegistrantDetails** command 
class that uses **DataAnnotations** to specify the validation 
requirements; in this example, that the **FirstName**, **LastName**, and 
**Email** fields are not empty. 

```Cs
using System;
using System.ComponentModel.DataAnnotations;
using Common;

public class AssignRegistrantDetails : ICommand
{
	public AssignRegistrantDetails()
	{
		this.Id = Guid.NewGuid();
	}

	public Guid Id { get; private set; }

	public Guid OrderId { get; set; }

	[Required(AllowEmptyStrings = false)]
	public string FirstName { get; set; }

	[Required(AllowEmptyStrings = false)]
	public string LastName { get; set; }

	[Required(AllowEmptyStrings = false)]
	public string Email { get; set; }
}
```

The MVC view uses this command class as its model class. The following 
code sample from the **SpecifyRegistrantDetails.cshtml** file shows how 
the model is populated. 

```HTML
@model Registration.Commands.AssignRegistrantDetails

...

<div class="editor-label">@Html.LabelFor(model => model.FirstName)</div><div class="editor-field">@Html.EditorFor(model => model.FirstName)</div>
<div class="editor-label">@Html.LabelFor(model => model.LastName)</div><div class="editor-field">@Html.EditorFor(model => model.LastName)</div>
<div class="editor-label">@Html.LabelFor(model => model.Email)</div><div class="editor-field">@Html.EditorFor(model => model.Email)</div>

```

Client-side validation based on the **DataAnnotations** attributes is 
configured in the **Web.config** file as shown in the following snippet. 

```XML
<appSettings>
    ...
    <add key="ClientValidationEnabled" value="true" />
    <add key="UnobtrusiveJavaScriptEnabled" value="true" />
</appSettings>
```

The server-side validation occurs in the controller before the command 
is sent. The following code sample from the **RegistrationController** 
class shows how the controller uses the **IsValid** property to validate 
the command. Remember that this example uses an instance of the command 
as the model. 

```Cs
[HttpPost]
public ActionResult SpecifyRegistrantDetails(string conferenceCode, Guid orderId, AssignRegistrantDetails command)
{
    if (!ModelState.IsValid)
    {
        return SpecifyRegistrantDetails(conferenceCode, orderId);
    }

    this.commandBus.Send(command);

    return RedirectToAction("SpecifyPaymentDetails", new { conferenceCode = conferenceCode, orderId = orderId });
}
```

For an additional example, see the **RegisterToConference** command and 
the **StartRegistration** action in the **RegistrationController** 
class. 

For more information, see [Models and Validation in ASP.NET 
MVC][modelvalidation] on MSDN. 

## Pushing Changes to the Read-side

Some information about orders only needs to exist on the read-side. In 
particular, the information about partially fulfilled orders is only 
used in the UI and is not part of the business information persisted by 
the domain model on the write-side. 

This means that the system can't use SQL views as the underlying storage 
mechanism on the read-side because views cannot contain data that does 
not exist in the tables that they are based on. 

The system stores the de-normalized order data in two tables in a SQL 
database: the **OrdersView** and **OrderItemsView** tables. The 
**OrderItemsView** table includes the **RequestedSeats** column that 
contains data that only exists on the read-side. 

<table border="1">
	<tr><th>Column</th><th>Description</th><tr>
	<tr><td>OrderId</td><td>A unique identifier for the Order</td><tr>
	<tr><td>ReservationExpirationDate</td><td>The time when the seat reservations expire</td><tr>
	<tr><td>StateValue</td><td>The state of the Order: Created, PartiallyReserved, ReservationCompleted, Rejected, Confirmed</td><tr>
	<tr><td>RegistrantEmail</td><td>The email address of the Registrant</td><tr>
	<tr><td>AccessCode</td><td>The Access Code that the Registrant can use to access the Order</td><tr>
</table>

**OrdersView Table**

<table border="1">
	<tr><th>Column</th><th>Description</th><tr>
	<tr><td>OrderItemId</td><td>A unique identifier for the Order Item</td><tr>
	<tr><td>SeatType</td><td>The type of Seat requested</td><tr>
	<tr><td>RequestedSeats</td><td>The number of seats requested</td><tr>
	<tr><td>ReservedSeats</td><td>The number of seats reserved</td><tr>
	<tr><td>OrderID</td><td>The OrderId in the parent OrdersView table</td><tr>
</table>

**OrderItemsView Table**

To populate these tables in the read-model, the read-side handles events 
raised by the write-side and uses them to write to these tables. See
Figure 2 above for more details.

The **OrderViewModelGenerator** class handles these events and updates
the read-side repository.

```Cs
public class OrderViewModelGenerator :
    IEventHandler<OrderPlaced>, IEventHandler<OrderUpdated>,
    IEventHandler<OrderPartiallyReserved>, IEventHandler<OrderReservationCompleted>,
    IEventHandler<OrderRegistrantAssigned>
{
    private readonly Func<ConferenceRegistrationDbContext> contextFactory;

    public OrderViewModelGenerator(Func<ConferenceRegistrationDbContext> contextFactory)
    {
        this.contextFactory = contextFactory;
    }

    public void Handle(OrderPlaced @event)
    {
        using (var repository = this.contextFactory.Invoke())
        {
            var dto = new DraftOrder(@event.SourceId, DraftOrder.States.Created)
            {
                AccessCode = @event.AccessCode,
            };
            dto.Lines.AddRange(@event.Seats.Select(seat => new DraftOrderItem(seat.SeatType, seat.Quantity)));

            repository.Save(dto);
        }
    }

    public void Handle(OrderRegistrantAssigned @event)
    {
        ...
    }

    public void Handle(OrderUpdated @event)
    {
        ...
    }

    public void Handle(OrderPartiallyReserved @event)
    {
        ...
    }

    public void Handle(OrderReservationCompleted @event)
    {
        ...
    }

    ...
}
```

The following code sample shows the **ConferenceRegistrationDbContext** 
class. 

```Cs
public class ConferenceRegistrationDbContext : DbContext
{
    ...

    public T Find<T>(Guid id) where T : class
    {
        return this.Set<T>().Find(id);
    }

    public IQueryable<T> Query<T>() where T : class
    {
        return this.Set<T>();
    }

    public void Save<T>(T entity) where T : class
    {
        var entry = this.Entry(entity);

        if (entry.State == System.Data.EntityState.Detached)
            this.Set<T>().Add(entity);

        this.SaveChanges();
    }
}
```

> **JanaPersona:** Notice that this **ConferenceRegistrationDbContext**
> in the read-side includes a **Save** method to persist the changes
> sent from the write-side and handled by the
> **OrderViewModelGenerator** handler class. 

## Querying the Read-side

The following code sample shows a non-generic DAO class that the MVC controllers use to query for conference information on the read-side. It wraps the **ConferenceRegistrationDbContext** class shown previously.

```Cs
public class ConferenceDao : IConferenceDao
{
    private readonly Func<ConferenceRegistrationDbContext> contextFactory;

    public ConferenceDao(Func<ConferenceRegistrationDbContext> contextFactory)
    {
        this.contextFactory = contextFactory;
    }

    public ConferenceDetails GetConferenceDetails(string conferenceCode)
    {
        using (var repository = this.contextFactory.Invoke())
        {
            return repository
                .Query<Conference>()
                .Where(dto => dto.Code == conferenceCode)
                .Select(x => new ConferenceDetails { Id = x.Id, Code = x.Code, Name = x.Name, Description = x.Description, StartDate = x.StartDate })
                .FirstOrDefault();
        }
    }

    public ConferenceAlias GetConferenceAlias(string conferenceCode)
    {
        ...
    }

    public IList<SeatType> GetPublishedSeatTypes(Guid conferenceId)
    {
        ...
    }
}
```

> **JanaPersona:** Notice how this **ConferenceDao** class only contains
> methods that return data. It is is used by the MVC controllers to
> retrieve data to display in the UI.

## Refactoring the SeatsAvailability Aggregates

In the first stage of our CQRS, the domain included a 
**ConferenceSeatsAvailabilty** aggregate root class that modelled the 
number of seats remaining for a conference. In this stage of the 
journey, the team replaced the **ConferenceSeatsAvailabilty** aggregate 
with a **SeatsAvailability** aggregate to reflect the fact that there 
may be multiple seat types available at a particular conference; for 
example, full conference seats, pre-conference workshop seats, and 
cocktail party seats. Figure 3 shows the new **SeatsAvailability** 
aggregate and its constituent classes. 

![Figure 3][fig3]

**The **SeatsAvailability** and its associated commands and events.

This aggregate now models the following facts:

* There may be multiple seat types at a conference.
* There may be different numbers of seats available for each seat type.

The domain now includes a **SeatQuantity** value type that you can use 
to represent a quantity of a particular seat type. 

Previously, the aggregate raised either a **ReservationAccepted** or 
**ReservationRejected** event depending on whether there were sufficient 
seats. Now the aggregate raises a **SeatsReserved** event that reports 
how many seats of a particular type could be reserved. This means that 
the number of seats reserved may not match the number of seats 
requested; this information is passed back to the UI for the registrant 
to make a decision on how to proceed with the registration. 

### The AddSeats Method

You may have noticed in Figure 2 that the **SeatsAvailability** 
aggregate includes an **AddSeats** method with no corresponding command. 
The **AddSeats** method adjusts the total number of available seats of a 
given type. The Business Customer is responsible for making any such 
adjustments, and does this in the Conference Management bounded context. 
The Conference Management bounded context raises an event whenever the 
total number of available seats changes, the **SeatsAvailability** class 
then handles the event when its handler invokes the **AddSeats** method. 

# Testing 

A common comment about implementations that use the CQRS pattern or that 
use messaging extensively is the difficulty in understanding how all of 
the different pieces of the application fit together through sending and 
receiving commands and events. You can help someone to understand your 
code base through appropriately designed unit tests. 

Consider this first example of a unit test for the **Order** aggregate:

```Cs
public class given_placed_order
{
    ...

    private Order sut;

    public given_placed_order()
    {
        this.sut = new Order(
            OrderId, new[] 
            {
                new OrderPlaced 
                { 
                    ConferenceId = ConferenceId,
                    Seats = new[] { new SeatQuantity(SeatTypeId, 5) },
                    ReservationAutoExpiration = DateTime.UtcNow
                }
            });
    }

    [Fact]
    public void when_updating_seats_then_updates_order_with_new_seats()
    {
        this.sut.UpdateSeats(new[] { new OrderItem(SeatTypeId, 20) });

        var @event = (OrderUpdated)sut.Events.Single();
        Assert.Equal(OrderId, @event.SourceId);
        Assert.Equal(1, @event.Seats.Count());
        Assert.Equal(20, @event.Seats.ElementAt(0).Quantity);
    }

    ...
}
``` 

This unit test creates an **Order** instance and directly invokes the 
**UpdateSeats** method. It does not provide any information to the 
person reading the test code about the command or event that causes this 
method to be invoked. 

Consider this second example that performs the same test, but in this 
case it does so by sending a command. 

```Cs
public class given_placed_order
{
    ...
    
    private EventSourcingTestHelper<Order> sut;

    public given_placed_order()
    {
        this.sut = new EventSourcingTestHelper<Order>();
        this.sut.Setup(new OrderCommandHandler(sut.Repository, pricingService.Object));

        this.sut.Given(
                new OrderPlaced 
                { 
                    SourceId = OrderId,
                    ConferenceId = ConferenceId,
                    Seats = new[] { new SeatQuantity(SeatTypeId, 5) },
                    ReservationAutoExpiration = DateTime.UtcNow
                });
    }

    [Fact]
    public void when_updating_seats_then_updates_order_with_new_seats()
    {
        this.sut.When(new RegisterToConference { ConferenceId = ConferenceId, OrderId = OrderId, Seats = new[] { new SeatQuantity(SeatTypeId, 20) }});

        var @event = sut.ThenHasSingle<OrderUpdated>();
        Assert.Equal(OrderId, @event.SourceId);
        Assert.Equal(1, @event.Seats.Count());
        Assert.Equal(20, @event.Seats.ElementAt(0).Quantity);
    }
    
    ...
}
```

This example uses a helper class that enables you to send a command to 
the **Order** instance. Now someone reading the test can see that when 
you send a **RegisterToConference** command you expect to see an 
**OrderUpdated** event. 

[j_chapter5]:       Journey_05_PaymentsBC.markdown
[r_chapter4]:       Reference_04_DeepDive.markdown
[appendix1]:        Appendix1_Running.markdown

[fig1]:             images/Journey_04_ViewRepository.png?raw=true
[fig2]:             images/Journey_04_Architecture.png?raw=true
[fig3]:             images/Journey_04_SeatsAvailability.png?raw=true
[modelvalidation]:  http://msdn.microsoft.com/en-us/library/dd410405(VS.98).aspx