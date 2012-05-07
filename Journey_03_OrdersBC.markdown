## Chapter 3
# Orders and Registrations Bounded Context 

_The first stopping point on our CQRS journey._

# A Description of the Orders and Registrations Bounded Context

The **Orders and Registrations** bounded context is partially responsible 
for the booking process for attendees planning to attend a conference. 
In the **Orders and Registrations** bounded context, a person (the 
registrant) purchases seats at a particular conference. In the 
**Registrations** bounded context, described in [To Do Identify 
Chapter][todo1], the registrant assigns names of attendees to the 
purchased seats. 

The ordering process must support wait listing, [To Do Identify 
Chapter][todo2], whereby requests for seats are placed on a wait-list if 
there are not sufficient seats available. The ordering process must 
enable the Business Customer to set various types of discount, [To Do 
Identify Chapter][todo3], for attendees. 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Revise the following paragraph.
  </span>
</div> 

This was the first stop on our CQRS journey, so the team decided to 
implement a core, but self-contained part of the system. The 
registrations process must be as painless as possible for attendees. The 
process must enable the Business Customer to ensure that the maximum 
possible number of seat can be booked, and give the Business Customer 
the flexibility to define a set of custom pricing and discount scheme. 

Because this was the first bounded context addressed by the team, they 
also implemented some infrastructure elements of the system to support 
the domain's functionality. These included command and event message 
buses and a persistence mechanism for aggregates. 

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
the event bus and then deliver the event to the subscriber. In this 
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

> **BharathPersona:** The team initially referred to the workflow class
> in the **Orders** bounded context as a saga. To find out why they
> decided to change the terminology. See the section [Patterns and
> Concepts](#patternsandconcepts) later in this chapter.

The Reference Guide contains additional definitions and explanations of 
CQRS related terms.

## User Stories

The bounded context descibed in this chapter addresses the following 
user story. 

### Seat Ordering

A registrant is the person who reserves and pays for seats at a 
conference. Ordering is a two-stage process: first the registrant 
reserves a number of seats, and then the registrant pays for the seats 
to confirm the reservation. If registrant does not complete the payment, 
the seat reservations expire after a fixed period of time and the system 
makes the seats available to other registrants to reserve. 

Figure 1 shows some of the early UI mockups that the team used to 
explore the seat ordering story. 

![Figure 1][fig1]

**Ordering UI mockups**

### Domain Definitions (Ubiquitous Language)

The following list defines the key domain related terms that the team 
used during the development of these **Orders and Registrations** 
bounded contexts. 

- **Attendee.** An Attendee is someone who is entitled to attend a 
  conference. An Attendee can interact with the system to perform tasks 
  such as manage his agenda, print his badge, and provide feedback after 
  the conference. An Attendee could also be a person who doesn't pay to 
  attend a conference such as a volunteer, speaker, or someone with a 
  100% discount. An Attendee may have multiple associated Attendee Types 
  (speaker, student, volunteer, track chair, etc.) 

- **Registrant.** A Registrant is a person who interacts with the 
  system to make Orders and to make payments for those Orders. A 
  Registrant also creates the Registrations associated with an Order. A 
  Registrant may also be an Attendee.
  
- **User.** A User is a person such as an Attendee, Registrant, Speaker, 
  Volunteer who is associated with a conference. Each User has a unique 
  Record Locator code that the User can use to access User specific 
  information in the system. For example, a Registrant can use a Record 
  Locator code to access her Orders, an Attendee can use a Record 
  Locator code to access his personalized conference agenda. 

- **Registration.** A Registration associates an Attendee with a Seat 
  in a confirmed Order. An Order may have one or more Registrations 
  associated with it. 

- **Order.** When a Registrant interacts with the system, the system
  creates an Order to manage the Reservations, payment, and 
  Registrations. An Order typically exists in one of four states: 
  Created, ReservationsMade, Rejected, and Confirmed. An Order is 
  initially in the Created state. An Order is in the ReservationsMade 
  state when the system has reserved some of the Seats requested by the 
  Registrant (one or more Order Items are in the Reserved state). An 
  Order is in the Confirmed state when the Registrant has successfully 
  paid for the Order Items in the Reserved state. An Order is in the 
  Rejected state if the system cannot reserve any of the Seats requested 
  by the Registrant (all the Order Items are in the Rejected state) or 
  if the payment process fails. An Order contains one of more Order 
  Items.

* **Order Item.** An Order Item represents a Seat Type and a Quantity
  and is associated with an Order. An Order Item exists in one of three 
  states: Created, Reserved, Rejected. An Order Item is initially in the 
  Created state. An Order Item is in the Reserved state if the system 
  has reserved the quantity of seats of the seat type requested by the 
  Registrant. An Order Item is in the Rejected state if the system 
  cannot reserve the quantity of seats of the seat type requested by the 
  Registrant. 

- **Seat.** A represents a space at a conference or access to a 
  specific session at the conference such as a cocktail party, a 
  tutorial, or a workshop. Each conference has a quota of Seats that may 
  be changed by the Business Customer. Each session has a quota of Seats 
  that may be changed by the Business Customer. 

- **Reservation.** A Reservation is a temporary reservation of one or 
  more seats. The ordering process creates Reservations. When a 
  Registrant begins the ordering process, the system makes Reservations 
  for the number of Seats requested by the Registrant. These Seats are 
  then not available for other Registrants to reserve. The Reservations 
  are held for n minutes (n is one of the properties of the conference 
  defined by the Conference Owner) during which the Registrant can 
  complete the ordering process by making a payment for those Seats. If 
  the Registrant does not pay for the seats within n minutes, the 
  system cancels the Reservation and the Seats become available to other 
  Registrants to reserve. 

- **Seats Availability.** Every conference tracks Seat availability 
  for each type of Seat. Initially, all of the Seats are available to 
  reserve and purchase. When a Seat is reserved, the number of available 
  Seats of that type is decremented. If the system cancels the 
  Reservation, the number of available Seats of that type is 
  incremented. The initial quota of each seat type is an attribute of 
  the conference and is defined by the Business Customer. A Conference 
  Owner may adjust the quotas for the individual seat types. 

- **Conference site.** Every conference defined in the system can be 
  accessed using a unique URL. Registrants can begin the ordering 
  process from this site. 

The following conversation between developers and domain experts 
illustrates how the team arrived at a definition of the term 
**Attendee**. 

> *Developer #1:* Here's an initial stab at a definition for 
> **Attendee**. "An attendee is someone who has paid to attend a 
> conference. An attendee can interact with the system to perform tasks 
> such as manage his agenda, print his badge, and provide feedback after 
> the conference." 

> *Domain Expert #1:* Not all attendees will pay to attend the 
> conference. For example, some conferences will have volunteer helpers, 
> also speakers typically don't pay. Also, there may be some cases where 
> an attendee gets a 100% discount. 

> *Domain Expert #1:* Don't forget that it's not the attendee who 
> pays; that's done by the registrant. 

> *Developer #1:* So we need to say that attendees are people who are 
> authorized to attend a conference? 

> *Developer #2:* We need to be careful about the choice of words 
> here. The term *authorized* will make some people think of security 
> and authentication and authorization. 

> *Developer #1:* How about *entitled*?

> *Domain Expert #1:* When the system performs tasks such as printing 
> badges, it will need to know what type of attendee the badge is for. 
> For example, speaker, volunteer, paid attendee, and so on. 

> *Developer #1:* Now we have this is a definition that captures 
> everything we've discussed. An Attendee is someone who is entitled to 
> attend a conference. An Attendee can interact with the system to 
> perform tasks such as manage his agenda, print his badge, and provide 
> feedback after the conference. An Attendee could also be a person who 
> doesn't pay to attend a conference such as a volunteer, speaker, or 
> someone with a 100% discount. An Attendee may have multiple associated 
> Attendee Types (speaker, student, volunteer, track chair, etc.) 

## Technologies and initial Architecture

The application is designed to deploy to Windows Azure. At this stage in 
the journey, the application consists of a web role that contains the 
MVC web application and a worker role that contains the message handlers 
and domain objects. The application uses SQL Azure databases for data 
storage, both on the write-side and the read-side (These datasources could 
change if using Event-Sourcing in future versions, as No-Sql datasources 
fit better in ES than relational DBs). 
The application uses the Windows Azure Service Bus to provide its messaging infrastructure. 

While you are exploring and testing the solution, you can run it 
locally, either using the Windows Azure compute emulator or by running 
the MVC web application directly and running a console application that 
hosts the handlers and domain objects. When you run the application 
locally, you can use a local SQL Express database instead of SQL Azure, 
and use a simple messaging infrastructure implemented in a SQL Express 
database. 

For more information about the options for running the application, see 
[Appendix 1][appendix1]. 

# Patterns and Concepts <a name="patternsandconcepts"/>

The team decided that they would try to implement the first bounded 
context without using event sourcing in order to keep things simple. 
However, they did agree that if they later decided that event sourcing 
would bring specific benefits to this bounded context, then they would 
revisit this decision. 

> **Note** For a description of how event sourcing relates to the CQRS
> pattern, see [Introducing Event Sourcing][r_chapter3] in the Reference
> Guide.

One of the important discussions in the team was around the choice of 
aggregates and entities that they would implement. The following images 
from the team's white-board illustrate some of their initial thoughts, 
and questions about the alternative approaches they could take with a 
simple conference seat reservation scenario to try and understand the
pros and cons of alternative approaches.

This scenario considers what happens when a registrant tries to book
several seats at a conference. The system must:

- Check that sufficient seats are available.
- Record details of the registration.
- Update the total number of seats booked for the conference.

> **Note:** The scenario is kept deliberately simple to avoid
  distractions while the team examines the alternatives. These examples
  do not illustrate the final implementation of this bounded context. 

The first approach considered by the team, shown in figure 2, uses two 
separate aggregates.

![Figure 2][fig2]

**Approach #1, two separate aggregates**

The numbers in the diagram correspond to the following steps:

1. The UI sends a command to register attendees X and Y onto conference
   157. The command is routed to a new **Order** aggregate.
2. The **Order** aggregate invokes a method on a **Conference Seats
   Availability** aggregate.
3. The **Conference Seats Availability** aggregate with an ID of 157 is
   re-hydrated from the data store.
4. The **Conference Seats Availability** aggregate updates its total
   number of seats booked.
5. The updated version of the **Conference Seats Availability**
   aggregate is persisted to the data store.
6. The new **Order** aggregate, with an ID of 4239, is persisted to the
   data store.

The second approach considered by the team, shown in figure 3, uses a 
single aggregate in place of two. 

![Figure 3][fig3]

**Approach #2, a single aggregate**

The numbers in the diagram correspond to the following steps:

1. The UI sends a command to register attendees X and Y onto conference
   157. The command is routed to the **Conference** aggregate with an
   ID of 157.
2. The **Conference** aggregate with an ID of 157 is re-hydrated from
   the data store.
3. The **Order** entity validates the booking (it queries the 
   **SeatsAvailability** entity to see if there are enough
   seats left), and then invokes the method to update the number of
   seats booked on the conference entity.
4. The **SeatsAvailability** entity updates its total number
   of seats booked.
5. The updated version of the **Conference** aggregate is persisted to
   the data store.

The third approach considered by the team, shown in figure 4, uses a 
workflow to coordinate the interaction between two aggregates. 

![Figure 4][fig4]

**Approach #3, using a workflow**

The numbers in the diagram correspond to the following steps:

1. The UI sends a command to register attendees X and Y onto conference
   #157. The command is routed to a new **Order** aggregate.
2. The new **Order** aggregate, with an ID of 4239,  is persisted to
   the data store.
3. The **Order** aggregate raises an event that is handled by the
   **RegistrationProcess** workflow.
4. The **RegistrationProcess** workflow determines that a command should be
   sent to the **SeatsAvailability** aggregate with an ID of
   157.
5. The **SeatsAvailability** aggregate is re-hydrated from the
   data store.
6. The total number of seats booked is updated in the
   **SeatsAvailability** aggregate and it is persisted to the
   data store.

> **BharathPersona:** Workflow or saga? Initially the team referred to
> the **RegistrationProcess** workflow as a saga. However, after they
> reviewed the original definition of a saga from the paper
> [Sagas](sagapaper) by Hector Garcia-Molina and Kenneth Salem, they
> revised their decision. The key reasons for this are that reservation
> process does not include explicit compensation steps, and does not
> need to be represented as a long-lived transaction.

For more information about workflows and sagas see the chapter
[Coordinating Workflows and Sagas][r_chapter6] in the Reference Guide.
   
The team identified these questions about these approaches:

- Where does the validation that there are sufficient seats for the 
  registration take place, in the **Order** or 
  **SeatsAvailability** aggregate? 
- Where are the transaction boundaries? 
- How does this model deal with concurrency issues when multiple 
  registrants try to place orders simultaneously? 
- What are the aggregate roots?

The following sections discuss these questions in relation to the three
approaches considered by the team.

## Validation

Before a registrant can reserve a seat, the system must check that there 
are enough seats available. Although logic in the UI can attempt to 
verify that there are sufficient seats available before it sends a 
command, the business logic in the domain must also perform the check 
because the state may change between the time the UI performs the 
validation and the time the command is delivered to the aggregate in the 
domain. 

In the first model, the validation must take place in either the 
**Order** or **SeatsAvailability** aggregate. If it is the 
former, the **Order** aggregate must discover the current seat 
availability from the **SeatsAvailability** aggregate before 
the reservation is made. If it is the later, the 
**SeatsAvailability** aggregate must return a success or 
failure code to the **Order** aggregate. 

The second model behaves similarly, except that it is **Order** and 
**SeatsAvailability** entities cooperating within a 
**Conference** aggregate. 

In the third model, with the workflow, the aggregates exchange messages
through the workflow about whether the registrant can make the reservation
at the current time. 

All three models require entities to communicate about the validation 
process, but the third model with the workflow appears more complex than the 
other two. 

## Transaction Boundaries

An aggregate, in the DDD approach, represents a consistency boundary. 
Therefore, the first model with two aggregates, and the third model with 
two aggregates and a workflow will involve two transactions: one when the 
system persists the new **Order** aggregate, and one when the system 
persists the updated **SeatsAvailability** aggregate. 

To ensure the consistency of the system when a registrant creates an 
order, both transactions must succeed. To guarantee this, we must take 
steps to ensure that the system is eventually consistent by ensuring 
that infrastructure reliably delivers messages to aggregates. 

The second approach that uses a single aggregate, we will only have a 
single transaction when a registrant makes an order. This appears to be 
the simplest approach of the three. 

## Concurrency

The registration process takes place in a multi-user environment where 
many registrants could attempt to purchase seats simultaneously. The 
team decided to use the [reservation pattern][res-pat] to address the 
concurrency issues in the registration process. In this scenario, this 
means that a registrant initially reserves seats (which are then 
unavailable) for other registrants; if the registrant completes the 
payment within a timeout period, the system keeps the reservations; 
otherwise the system cancels the reservation. 

This reservation system introduces the need for additional message 
types, for example an event to report that a registrant has made a 
payment, or report that a timeout has occurred. 

This timeout also requires the system to incorporate a timer somewhere 
to track when reservations expire. 

Modeling this complex behavior with sequences of messages and the 
requirement for a timer is best done using a workflow. 

## Aggregates and Aggregate Roots

In the two models where there are the **Order** aggregate and the 
**SeatsAvailability** aggregate, the team easily identified 
the entities that make up the aggregate, and the aggregate root. The 
choice is not so clear in the model with a single aggregate: it does not 
seem natural to access orders through a **SeatsAvailability** 
entity, or to access the seat availability through an **Order** entity. 
Creating a new entity to act as an aggregate root seems unnecessary. 

The team decided on the model that incorporated a workflow because this 
offers the best way to handle the concurrency requirements in this 
bounded context. 
	
# Implementation Details

This section describes some of the significant features of the 
implementation of the Orders and Registrations bounded context. You may 
find it useful to have a copy of the code so you can follow along. You 
can download a copy of the code from the repository on github: 
[mspnp/cqrs-journey-code][repourl]. 

> **Note:** Do not expect the code samples to exactly match the code in
> the reference implementation. This chapter describes a step in the
> CQRS journey, the implementation may well change as we learn more and
> refactor the code.

## High-level Architecture

As we described in the previous section, the team initially decided to 
implement the reservations story in the Conference Management System 
using the CQRS pattern but without using event sourcing. Figure 5 shows 
the key elements of the implementation: an MVC web application, a data 
store implemented using a SQL database, the read and write models, and 
some infrastructure components. 

> **Note:** We'll describe what goes on inside the read and write models 
> later in this section. 

![Figure 5][fig5]

**High-level architecture of the registrations bounded context**

The following sections relate to the numbers in figure 5 and provide 
more detail about these elements of the architecture. 

### 1. Querying the Read Model

The **ConferenceController** class includes an action named **Display** 
that creates a view that contains information about a particular 
conference. This controller class queries the read model using the 
following code: 


```Cs
public ActionResult Display(string conferenceCode)
{
	var conference = this.GetConference(conferenceCode);

	return View(conference);
}

private Conference.Web.Public.Models.Conference GetConference(string conferenceCode)
{
	var repo = this.repositoryFactory();
	using (repo as IDisposable)
	{
		var conference = repo.Query<Conference>().First(c => c.Code == conferenceCode);

		var conference =
			new Conference.Web.Public.Models.Conference { Code = conference.Code, Name = conference.Name, Description = conference.Description };

		return conference;
	}
}
```

The read model retrieves the information from the data store and returns 
it to the controller using a Data Transfer Object (DTO) class. 

### 2. Issuing Commands

The web application sends commands to the write model through a command 
bus. This command bus is an infrastructure element that provides 
reliable messaging. In this scenario, the bus delivers messages 
asynchronously and once only to a single recipient.

The **RegistrationController** class can send a **RegisterToConference** 
command to the write model in response to user interaction. This command 
sends a request to register one or more seats at the conference. The 
**RegistrationController** class then polls the read model to discover 
whether the registration request succeeded. See the section "6. Polling 
the Read Model" below for more details 

The following code sample shows how the **RegistrationController** sends
a **RegisterToConference** command:

```Cs
var viewModel = this.UpdateViewModel(conferenceCode, contentModel);

var command =
	new RegisterToConference
	{
		OrderId = viewModel.Id,
		ConferenceId = viewModel.ConferenceId,
		Seats = viewModel.Items.Select(x => new RegisterToConference.Seat { SeatTypeId = x.SeatTypeId, Quantity = x.Quantity }).ToList()
	};

this.commandBus.Send(command);
```

> **Note:** All of the commands are sent asynchronously and do not 
expect return values. 

### 3. Handling Commands

Command handlers register with the command bus; the command bus can then 
forward commands to the correct handler. 

The **OrderCommandHandler** class handles the **RegisterToConference** 
command sent from the UI. Typically, the handler is responsible for 
initiating any business logic in the domain and persisting any state 
changes to the data store. 

The following code sample shows how the **OrderCommandHandler** 
class handles the **RegisterToConference** command: 

```Cs
public void Handle(RegisterToConference command)
{
    var repository = this.repositoryFactory();

    using (repository as IDisposable)
    {
        var seats = command.Seats.Select(t => new OrderItem(t.SeatTypeId, t.Quantity)).ToList();

        var order = new Order(command.OrderId, Guid.NewGuid(), command.ConferenceId, seats);

        repository.Save(order);
    }
}
```

### 4. Initiating Business Logic in the Domain

In the previous code sample, the **OrderCommandHandler** class 
creates a new **Order** instance. The **Order** entity is an aggregate 
root, and its constructor contains code to initiate the domain logic. 
See the section "Inside the Write Model" below for more details of what 
actions this aggregate root performs. 

### 5. Persisting the Changes

In the previous code sample, the handler persists the new **Order** 
aggregate by calling the **Save** method in the repository class. This 
**Save** method also publishes any events raised by the **Order** 
aggregate on the command bus. 

### 6. Polling the Read Model

To provide feedback to the user, the UI must have a way to check whether 
the **RegisterToConference** command succeeded. Like all commands in the 
system, this command is executed asynchronously and does not return a 
result. The UI queries the read model to check whether the 
command succeeded. 

The following code sample shows the initial implementation where the 
**RegistrationController** class polls the read model until either the 
order is created or a timeout occurs. The **WaitUntilUpdated** method
polls the read-model until either it finds that the order has been 
persisted or it times out.


```Cs
[HttpPost]
public ActionResult StartRegistration(string conferenceCode, OrderViewModel contentModel)
{
    ...
	
	this.commandBus.Send(command);

    var draftOrder = this.WaitUntilUpdated(viewModel.Id);

    if (draftOrder != null)
    {
        if (draftOrder.State == "Booked")
        {
            return RedirectToAction("SpecifyPaymentDetails", new { conferenceCode = conferenceCode, orderId = viewModel.Id });
        }
        else if (draftOrder.State == "Rejected")
        {
            return View("ReservationRejected", viewModel);
        }
    }

    return View("ReservationUnknown", viewModel);
}
```

The team later replaced this mechanism for checking whether the order 
was saved with an implementation of the Post-Redirect-Get pattern. The 
following code sample shows the new version of the **StartRegistration** 
action method. 

```Cs
[HttpPost]
public ActionResult StartRegistration(string conferenceCode, OrderViewModel contentModel)
{
    ...

    this.commandBus.Send(command);

    return RedirectToAction("SpecifyRegistrantDetails", new { conferenceCode = conferenceCode, orderId = command.Id });
}
```

The action method now redirects to the **SpecifyRegistrantDetails** view 
immediately after it sends the command. The following code sample shows 
how the **SpecifyRegistrantDetails** action polls for the order in the 
repository before returning a view. 

```Cs
[HttpGet]
public ActionResult SpecifyRegistrantDetails(string conferenceCode, Guid orderId)
{
    var draftOrder = this.WaitUntilUpdated(orderId);
    
	...
}
```
The advantages of this second approach, using the Post-Redirect-Get 
pattern instead of in the **StartRegistration** post action are that it 
works better with the browser's forward and back navigation buttons, and 
that it gives the infrastructure more time to process the command before 
the MVC controller starts polling. 

## Inside the Write Model

### Aggregates

The following code sample shows the **Order** aggregate.

```Cs
public class Order : IAggregateRoot, IEventPublisher
{
    public static class States
    {
        public const int Created = 0;
        public const int Booked = 1;
        public const int Rejected = 2;
        public const int Confirmed = 3;
    }

    private List<IEvent> events = new List<IEvent>();

    ...

    public Guid Id { get; private set; }

    public Guid UserId { get; private set; }

    public Guid ConferenceId { get; private set; }

    public virtual ObservableCollection<TicketOrderLine> Lines { get; private set; }

    public int State { get; private set; }

    public IEnumerable<IEvent> Events
    {
        get { return this.events; }
    }

    public void MarkAsBooked()
    {
        if (this.State != States.Created)
            throw new InvalidOperationException();

        this.State = States.Booked;
    }

    public void Reject()
    {
        if (this.State != States.Created)
            throw new InvalidOperationException();

        this.State = States.Rejected;
    }
}
```

Notice how the properties of the class are not virtual. In the original 
version of this class, the properties **Id**, **UserId**, 
**ConferenceId**, and **State** were all marked as virtual. The 
following conversation between two developers explores this decision. 

> *Developer #1:* I'm really convinced you should not make the 
> property virtual, except if required by the ORM. If this is just for 
> testing purpose, entities and aggregate roots should never be tested 
> using mocking. If you need mocking to test your entities, this is a 
> clear smell that something is wrong in the design. 

> *Developer #2:* I prefer to be open and extensible by default.You 
> never know what needs may arise in the future, and making things 
> virtual is hardly a cost. This is certainly controversial and a bit 
> non-standard in .NET, but I think it's OK. We may only need virtuals 
> on lazy-loaded collections. 

> *Developer #1:* Since CQRS usually make the nead for lazy load 
> vanish you should not need it either. This leads to even simpler code. 

> *Developer #2:* CQRS does not dictate usage of ES, so if you're 
> using an aggregate root that contains an object graph, you'd need that 
> anyway, right? 

> *Developer #1:* This is not about ES, it's about DDD. When your 
> aggregate boundaries are right, you don't need delay load. 

> *Developer #2:* To be clear, the aggregate boundary is here to group 
> things that should change together for reasons of consitency. A lazy 
> load would indicate that things that have been grouped together don't 
> really need this grouping? 

> *Developer #1:* I agree. I have found that lazy-loading in the 
> command side means I have it modeled wrong. If I don't need the value 
> in the command side, then it shouldn't be there. In addition, I 
> dislike virtuals unless they have an intended purpose (or some 
> artificial requirement from an ORM). In my opinion, it violates the 
> Open-Closed principle: you have opened yourself up for modification in 
> a variety of ways that may or may not be intended and where the 
> repercussions might not be immediately discoverable, if at all. 

> *Developer #2:* Our **Order** aggregate in the model has a list of 
> **Order Items**. Surely we don't need to load the lines to mark it as 
> Booked? Do we have it modeled wrong there? 

> *Developer #1:* Is the list of **Order Items** that long ? If it is, 
> the modeling is maybe wrong because you don't necesserily need 
> transactionality at that level. Often, doing a late roundtrip to get 
> and update **Order Items** can be more costly that loading them 
> upfront: you should evaluate the usual size of the collection and do 
> some performance measurement. Make it simple first, optimize if 
> needed. 

> *Thanks to J&eacute;r&eacute;mie Chassaing and Craig Wilson*

### Aggregates and Workflows

Figure 6 shows the entities that exist in the write-side model. There 
are two aggregates, **Order** and **SeatsAvailability**, each 
one containing multiple entity types. There is also a 
**RegistrationProcess** class that manages the interaction between the
aggregates. 

The table in the figure 6 shows how the workflow behaves given a current 
state and a particular type of incoming message. 

![Figure 6][fig6]

**Domain objects in the write model**

The process of registering for a conference begins when the UI sends a 
**RegisterToConference** command. The infrastructure delivers this 
command to the **Order** aggregate. The result of this command is that 
the system creates a new **Order** instance, and that the new **Order** 
instance raises an **OrderPlaced** event. The following code sample from 
the constructor in the **Order** class shows this happening. Notice how 
the system uses GUIDs to identify the different entities. 

```Cs
public Order(Guid id, Guid userId, Guid conferenceId, IEnumerable<OrderItem> lines)
{
    this.Id = id;
    this.UserId = userId;
    this.ConferenceId = conferenceId;
    this.Lines = new ObservableCollection<OrderItem>(items);

    this.events.Add(
        new OrderPlaced
        {
            OrderId = this.Id,
            ConferenceId = this.ConferenceId,
            UserId = this.UserId,
            Seats = this.Lines.Select(x => new OrderPlaced.Seat { SeatTypeId = x.SeatTypeId, Quantity = x.Quantity }).ToArray()
        });
}
```

> **Note:** To see how the infrastructure elements deliver commands and
  events, see figure 7.

The system creates a new **RegistrationProcess** instance to manage the 
new order. The following code sample from the **RegistrationProcess** 
class shows how the workflow handles the event. 

```Cs
public void Handle(OrderPlaced message)
{
    if (this.State == ProcessState.NotStarted)
    {
        this.OrderId = message.OrderId;
        this.ReservationId = Guid.NewGuid();
        this.State = ProcessState.AwaitingReservationConfirmation;

        this.AddCommand(
            new MakeSeatReservation
            {
                ConferenceId = message.ConferenceId,
                ReservationId = this.ReservationId,
                NumberOfSeats = message.Items.Sum(x => x.Quantity)
            });
    }
    else
    {
        throw new InvalidOperationException();
    }
}
```

The code sample shows how the workflow changes its state and sends a new 
**MakeSeatReservation** command that is handled by the 
**SeatsAvailability** aggregate. The code sample also 
illustrates how the workflow is implemented as a state machine that receives 
messages, changes its state, and sends new messages. 

> **DeveloperPersona:** Notice how we generate a new Guid to identify 
> the new reservation. We use these Guids to correlate messages to the 
> correct workflow and aggregate instances. 

When the **SeatsAvailability** aggregate receives a 
**MakeReservation** command, it makes a reservation if there are enough 
available seats. The following code sample shows how the 
**SeatsAvailability** class raises different events depending 
on whether or not there are sufficient seats. 

```Cs
public void MakeReservation(Guid reservationId, int numberOfSeats)
{
    if (numberOfSeats > this.RemainingSeats)
    {
        this.events.Add(new ReservationRejected { ReservationId = reservationId, ConferenceId = this.Id });
    }
    else
    {
        this.PendingReservations.Add(new Reservation(reservationId, numberOfSeats));
        this.RemainingSeats -= numberOfSeats;
        this.events.Add(new ReservationAccepted { ReservationId = reservationId, ConferenceId = this.Id });
    }
}
```

The **RegistrationProcess** class handles the the 
**ReservationAccepted** and **ReservationRejected** events. This 
reservation is a temporary reservation for seats to give the user the 
opportunity to make a payment. The workflow is responsible for releasing 
the reservation when either the purchase is complete, or the reservation 
timeout period expires. The following code sample shows how the workflow 
handles these two messages. 

```Cs
public void Handle(ReservationAccepted message)
{
    if (this.State == ProcessState.AwaitingReservationConfirmation)
    {
        this.State = ProcessState.AwaitingPayment;

        this.AddCommand(new MarkOrderAsBooked { OrderId = this.OrderId });
        this.commands.Add(
            new Envelope<ICommand>(new ExpireOrder { OrderId = this.OrderId, ConferenceId = message.ConferenceId })
            {
                Delay = TimeSpan.FromMinutes(15),
            });
    }
    else
    {
        throw new InvalidOperationException();
    }
}

public void Handle(ReservationRejected message)
{
    if (this.State == ProcessState.AwaitingReservationConfirmation)
    {
        this.State = ProcessState.Completed;
        this.AddCommand(new RejectOrder { OrderId = this.OrderId });
    }
    else
    {
        throw new InvalidOperationException();
    }
}
```

If the reservation is accepted, the workflow starts a timer running by 
sending an **ExpireOrder** command to itself, and sends a 
**MarkOrderAsBooked** command to the **Order** aggregate. Otherwise, it 
sends a **ReservationRejected** message back to the **Order** aggregate. 

The previous code sample shows how the workflow sends the 
**ExpireOrder** command. The infrastructure is responsible for 
holding the message in a queue for the delay of fifteen minutes. 

You can examine the code in the **Order**, 
**SeatsAvailability**, and **RegistrationProcess** classes 
to see how the other message handlers are implemented. They all follow 
the same pattern: receive a message, perform some logic, and send a 
message. 

> **JanaPersona:** The code samples shown in this chapter are from an
> early version of the Conference Management System. The next chapter
> shows how the design and implementation evolved as the team explored
> the domain and and learned more about the CQRS pattern.

### Infrastructure

The sequence diagram in figure 7shows how the infrastructure elements 
interact with the domain objects to deliver messages. 

![Figure 7][fig7]

**Infrastructure sequence diagram**

A typical interaction begins when an MVC controller in the UI sends a 
message using the command bus. The message sender invokes the **Send** 
method on the command bus asynchronously. The command bus then stores 
the message until the message recipient retrieves the message and 
forwards it to the appropriate handler. The system includes a number of 
command handlers that register with the command bus to handle specific 
types of command. For example, the **OrderCommandHandler** class defines 
handler methods for the **RegisterToConference**, **MarkOrderAsBooked**, 
and **RejectOrder** commands. The following code sample shows the 
handler method for the **MarkOrderAsBooked** command. Handler methods 
are responsible for locating the correct aggregate instance, calling 
methods on that instance, and then saving that instance. 

```Cs
public void Handle(MarkOrderAsBooked command)
{
    var repository = this.repositoryFactory();

    using (repository as IDisposable)
    {
        var order = repository.Find<Order>(command.OrderId);

        if (order != null)
        {
            order.MarkAsBooked();
            repository.Save(order);
        }
    }
}
```

The class that implements the **IRepository** interface is responsible 
for persisting the aggregate and publishing any events raised by the 
aggregate on the event bus, all as part of a transaction. 

**CarlosPersona:** The team later discovered an issue with this when 
they tried to use Windows Azure Service Bus as the messaging 
infrastructure. Windows Azure Service Bus does support distributed 
transactions with databases. For a discussion of this issue, see 
[Preparing for the V1 Release][j_chapter5] later in this guide. 

The only event subscriber in the reservations bounded context is the 
**RegistrationProcess** class. Its router subscribes to the event bus to 
handle specific events as shown in the following code sample from the 
**RegistrationProcess** class. 

> **Note:* We use the term handler to refer to the classes that handle 
> commands and forward them to aggregate instances, and the term router 
> to refer to the classes that handle events and commands and forward 
> them to workflow instances. 

```Cs
public void Handle(ReservationAccepted @event)
{
	var repo = this.repositoryFactory.Invoke();
	using (repo as IDisposable)
	{
        lock (lockObject)
        {
            var process = repo.Find<RegistrationProcess>(@event.ReservationId);
            process.Handle(@event);

            repo.Save(process);
        }
	}
}
```

Typically, an event handler method loads a workflow instance, passes the 
event to the workflow, and then persists the workflow instance. In this case, 
the **IRepository** instance is responsible for persisting the workflow 
instance and sending any commands from the workflow instance to the command 
bus. 

## Using the Windows Azure Service Bus

To transport Command and Event messages, the team decided to use the 
Windows Azure Service Bus to provide the basic, low-level messaging 
infrastructure. This section describes how the system uses the Windows 
Azure Service Bus and some of the alternatives and trade-offs the team 
considered considered during the design phase. 

> **JanaPersona:** The team at Contoso desided to use the Windows Azure
> Service Bus because it offers out-of-the-box support for the messaging
> scenarios in the Conference Management System. This minimizes the
> amount of code that the team needs to write, and provides for a
> robust, scalable messaging infrastructure.

Figure 8 shows how messages, both commands and events, flow through the 
system. Objects in the UI and domain objects use **CommandBus** and 
**EventBus** instances to send **BrokeredMessage** messages to a topic 
in the Windows Azure Service Bus. Command and event handler classes 
register use a **SubscriptionReceiver** instance to receive messages 
from subscriptions within a topic. The command and event handler classes 
then deliver the message to a domain object. 

> **Note:** A Windows Azure Service Bus topic can have multiple 
> subscribers. The Windows Azure Service Bus delivers messages sent to a 
> topic to all its subscribers. Therefore one message can have multiple 
> recipients. 

![Figure 8][fig8]

**Message flows through a Windows Azure Service Bus topic.**

In the intial implementation, the **CommandBus** and **EventBus** 
classes are very similar. The only difference between the **Send** 
method and the **Publish** method is that the **Send** method expects 
the message to be wrapped in an **Envelope** class. The **Envelope** 
class enables the sender to specify a time delay for the message 
delivery. 

Because an event may be processed by multiple subscribers, the topic 
that the **TopicSender** class sends events to can have multiple 
subscriptions. Each subscription is associated with a particular handler 
type so that events reach all of their subscribers. In the example shown 
in figure 8, the **ReservationRejected** event is sent to the 
**RegistrationProcess**, the **WaitListProcess**, and one other 
destination. 

A command has only one recipient. In figure 8, the 
**MakeSeatReservation** is sent to the **SeatsAvaialbility** aggregate. 
There is just a single handler registered for this subscription. 

There are a number of questions that arise from this implementation:

1. How do you limit delivering a command to a single recipient?
2. Why have separate **CommandBus** and **EventBus** classes if they are
   so similar?
3. How scalable is this approach?
4. How robust is this approch?
5. What is the granularity of a topic and a subscription?
6. How are commands and events serialized?

These questions are discussed in the following sections.

### How do you limit delivering a command to a single recipient?

This discussion assumes you that you have a basic understanding of the 
differences between Windows Azure Service Bus queues and topics. For an 
introduction to Windows Azure Service Bus, see [Technologies Used in the 
Reference Implementation][r_chapter9] in the Reference Guide. 

With the implementation shown in figure 8, the only way to ensure that a 
command is delivered to a single recipient is to ensure that a topic has 
only one subscription. After an instance of the **SubscriptionReceiver** 
class has received a message, then it is deleted from the subscription. 
There is no way in Windows Azure Service Bus to restrict a topic to a 
single subscription, therefore the developers must be careful to create 
just a single subscription on a topic that is delivering commands. 

> **Note:** It is possible to have multiple **SubscriptionReceiver** 
> instances running, perhaps in multiple worker role instances. If 
> multiple **SubscriptionReceiver** instances can receive messages from 
> the same topic subscription, then the first one call the **Receive** 
> method on the **SubscriptionClient** object will get and handle the 
> command. 

An alternative approach is to use a Windows Azure Service Bus queue in 
place of a topic for delivering command messages. Windows Azure Service 
Bus queues differ from topics in that they are designed to deliver 
messages to a single recipient instead of to multiple recipients through 
multiple subscriptions. The developers plan to evaluate this option in 
more detail with a view to implementing this approach later in the 
project. 

The following code sample from the **SubscriptionReceiver** class shows 
how it receives a message from the topic subscription. 

```Cs
private SubscriptionClient client;
private CancellationTokenSource cancellationSource;

...

private void ReceiveMessages(CancellationToken cancellationToken)
{
    while (!cancellationToken.IsCancellationRequested)
    {
        var message = this.client.Receive(TimeSpan.FromSeconds(10));

        if (message == null)
        {
            Thread.Sleep(100);
            continue;
        }

        if (!cancellationToken.IsCancellationRequested)
            this.MessageReceived(this, new BrokeredMessageEventArgs(message));
    }
}
```

The Windows Azure Service Bus **SubscriptionClient** class uses a 
peek/lock technique to retrieve a message from a subscription. In the 
code sample, the **Receive** method locks the message on the 
subscription. If the client calls the **Complete** method, the message 
is deleted from the subscription. Otherwise, if the client calls the 
**Abandon** method or a timeout occurs, the lock on the message is 
released and it can be received again by the same, or a different, 
client. 

> **Note:** The **MessageReceived** event passes a reference to the 
> **SubscriptionReceiver** instance so that the handler can call either 
> the **Complete** or **Abandon** methods when it processes the message.

The following code sample from the **MessageProcessor** class shows how 
to call the **Complete** method asynchronously using the 
**BrokeredMessage** instance passed as a parameter to the 
**MessageReceived** event. 

```Cs
private void OnMessageReceived(object sender, BrokeredMessageEventArgs args)
{
    var message = args.Message;

    using (var stream = message.GetBody<Stream>())
    {
        var payload = this.serializer.Deserialize(stream);
        
		...
		
        try
        {
            ProcessMessage(payload);
            message.Async(message.BeginComplete, message.EndComplete);
        }
        catch (Exception)
        {
            ...
			
            if (args.Message.DeliveryCount >= 5)
                args.Message.Async(args.Message.BeginDeadLetter, args.Message.EndDeadLetter);
        }
    }
}
``` 
> **Note:** This example uses an extension method to invoke the
  **BeginComplete** and **EndComplete** methods.

### Why have separate **CommandBus** and **EventBus** classes if they are so similar?

Although at this early stage in the development of the Conference 
Management system the implementations of the **CommandBus** and 
**EventBus** classes are very similar, the team anticipates that they 
will diverge in the future. 

> **DeveloperPersona:** There may be differences in how we invoke 
> handlers and what context we capture for them: commands may want to 
> capture additional runtime state, whereas events typically don't need 
> to. Because of these potential future differences, I didn't want to 
> unify the implementations. I've been there before and ended up 
> splitting them when further requirements came in. 

### How scalable is this approach?

With this approach you can run multiple instances of the 
**SubscriptionReceiver** class and the various handlers in different 
Windows Azure worker role instances, which enables you to scale-out your 
solution. You can also have multiple instances of the **CommandBus**, 
**EventBus**, and **TopicSender** classes in different Windows Azure 
worker role instances. 

For information about scaling the Windows Azure Service Bus 
infrastructure, see [Best Practices for Performance Improvements Using 
Service Bus Brokered Messaging][sbperf] on MSDN. 

### How robust is this approch?

This approach uses the brokered messaging option of the Windows Azure 
Service Bus to provided asynchronous messaging. The Service Bus reliably 
stores messages until consumers connect and retrieve their messages. 

Also, the peek/lock approach to retrieving messages from a queue or 
topic subscription adds reliability in the scenario that a message 
consumer fails while it is processing the message. If a consumer fails 
before it calls the **Complete** method, the message is still available 
for processing when the consumer restarts. 

### What is the granularity of a topic and a subscription?

As stated previously, a subscription should be associated with a single 
event handler type, although an event may be handled by multiple handler 
types. For example, the **ReservationRejected** event may be handled by 
both the **RegistrationProcessHandler** and **WaitListProcessHandler** 
handler classes because it must be delivered to the two workflows. 

Figure 8 suggests that Topic B is only reponsible for delivering 
**ReservationRejected** events. However, it could also deliver 
additional event types such as **ReservationAccepted** events. In this 
scenario, the handler classes might need to include some additional 
logic: if the **ReservationAccepted** event only needs to go to the 
**RegistrationProcess** workflow, then the **WaitListProcess** would 
need to discard any **ReservationAccepted** events that it retrieved 
from its subscription. 

In practice, it may be simpler to use one topic per event type. For 
example a topic for **ReservationAccepted** events and another topic for 
**ReservationRejected** events. 

> **ArchitectPersona:** There are no costs associated with having 
> multiple topics, subscriptions, or queues. Windows Azure Service Bus 
> usage is billed based on the number of messages sent and the amount of 
> data transfered out of Windows Azure sub-region. 

### How are commands and events serialized?

The Contoso Conference Management System uses the [Json.NET][jsonnet] 
serializer. For details of how this serializer is used within the 
application, see [Technologies Used in the Reference 
Implementation][r_chapter9] in the Reference Guide. 

> You should consider whether you really need to use the Windows Azure
> Service Bus for commands. Commands are typically used within a bounded
> context and you may not need to send them across a process boundary
> (on the write-side you may not need additional tiers), in which case
> you can use an in memory queue to deliver your commands.
  
> Greg Young - Conversation with the PnP team.

# Testing
Because this was the first bounded context the team tackled, one of the 
key concerns was how to approach testing given that the team wanted to 
adopt a Test-Driven Development approach. The following conversation 
summarizes their thoughts: 

**A conversation between two developers about how to do TDD when they 
are implementing the CQRS pattern without ES.** 


> *Developer #1*: If we were using event sourcing it would be easy to use 
> a TDD approach when we are creating our domain objects. The input to the 
> test would be a command (that perhaps originated in the UI), and we 
> could then test that the domain object fires the expected events. 
> However if we're not using event sourcing we don't have any events: the 
> behavior of the domain object is to persist its changes in data store 
> through an ORM layer. 

> *Developer #2*: So why don't we raise events anyway? Just because we're 
> not using event sourcing doesn't mean that our domain objects can't 
> raise events. We can then design our tests in the usual way to check for 
> the correct events firing in response to a command. 

> *Developer #1*: Isn't that just making things more complicated than they 
> need to be? One of the motivations for using CQRS is to simplify things! 
> We now have domain objects that need to persist their state using an ORM 
> layer, and also raise events reporting on what they have persisted just 
> so we can run our unit tests. 

> *Developer #2*: I see what you mean. 

> *Developer #1*: Perhaps we're getting hung up on how we're doing the 
> tests. Maybe instead of designing our tests based on the expected 
> *behavior* of the domain objects, we should think about testing the 
> *state* of the domain objects after they've processed a command? 

> *Developer #2*: That should be easy to do, after all the domain objects 
> will have all of the data we want to check stored in properties so that 
> the ORM can persist the right information to the store. 

> *Developer #1*: So we really just need to think about a different style 
> of testing in this scenario. 

> *Developer #2*: There is another aspect of this we'll need to consider: 
> we might have a set of tests that we can use to test our domain objects, 
> and all of those tests might be passing. We might also have a set of 
> tests to test that our ORM layer can save and retrieve objects 
> successfully. However, we will also have to test that our domain objects 
> function correctly when we run them against the ORM layer. It's possible 
> that a domain object performs the correct business logic, but can't 
> properly persist its state, perhaps because of a problem related to how 
> the ORM handles specific data types. 

For more information about the two approaches to testing discussed here, 
see Martin Fowler's article [Mocks Aren't Stubs][tddstyle]. 

> **Note:** The tests included in the solution are written using
> xUnit.net.

The following code sample shows two examples of tests written using the 
behavioural approach discussed above. 

> **MarkusPersona:** These are the tests we started with, but we then
  replaced them with state based tests.

```Cs
public SeatsAvailability given_available_seats()
{
	var sut = new SeatsAvailability(SeatTypeId);
	sut.AddSeats(10);
	return sut;
}
		
[TestMethod]
public void when_reserving_less_seats_than_total_then_succeeds()
{
	var sut = this.given_available_seats();
	sut.MakeReservation(Guid.NewGuid(), 4);
}

[TestMethod]
[ExpectedException(typeof(ArgumentOutOfRangeException))]
public void when_reserving_more_seats_than_total_then_fails()
{
	var sut = this.given_available_seats();
	sut.MakeReservation(Guid.NewGuid(), 11);
}
```

This two tests work together to verify the behavior of the 
**SeatsAvailability** attribute. In the first test, the 
expected behavior is that the **MakeReservation** method succeeds and 
does not throw an exception. In the second test, the expected behavior 
is for the **MakeReservation** method to throw an exception because 
there are not enough free seats available to complete the reservation. 

It is difficult to test the behavior in any other way without the 
aggregate raising events. For example, if you tried to test the behavior 
by checking that the correct call is made to persist the aggregate to 
the data store, the test becomes coupled to the data store 
implementation: if you want to change the data store implementation, you 
will need to change the tests on the aggregates in the domain model. 

The following code sample shows an example of a test written using the 
state of the objects being tested. This style of tests are the ones used
in the project.

```Cs
public class given_available_seats
{
	private static readonly Guid SeatTypeId = Guid.NewGuid();

	private SeatsAvailability sut;
	private IPersistenceProvider sutProvider;

	protected given_available_seats(IPersistenceProvider sutProvider)
	{
		this.sutProvider = sutProvider;
		this.sut = new SeatsAvailability(SeatTypeId);
		this.sut.AddSeats(10);

		this.sut = this.sutProvider.PersistReload(this.sut);
	}

	public given_available_seats()
		: this(new NoPersistenceProvider())
	{
	}

	[Fact]
	public void when_reserving_less_seats_than_total_then_seats_become_unavailable()
	{
		this.sut.MakeReservation(Guid.NewGuid(), 4);
		this.sut = this.sutProvider.PersistReload(this.sut);

		Assert.Equal(6, this.sut.RemainingSeats);
	}

	[Fact]
	public void when_reserving_more_seats_than_total_then_rejects()
	{
        var id = Guid.NewGuid();
        sut.MakeReservation(id, 11);

        Assert.Equal(1, sut.Events.Count());
        Assert.Equal(id, ((ReservationRejected)sut.Events.Single()).ReservationId);
	}
}
```

The two tests shown here test the state of the **SeatsAvailability** 
aggregate after invoking the **MakeReservation** method. The first test, 
tests the scenario where there are enough seats available. The second 
test, tests the scenario where there are not enough seats available. 
This second test can make use of the behavior the **SeatsAvailability** 
aggregate because the aggregate does raise an event if it rejects a 
reservation. 

[j_chapter5]:     Journey_05_PaymentsBC.markdown
[r_chapter3]:     Reference_03_ESIntroduction.markdown
[r_chapter4]:     Reference_04_DeepDive.markdown
[r_chapter6]:     Reference_06_Sagas.markdown
[r_chapter9]:     Reference_09_Technologies.markdown
[appendix1]:      Appendix1_Running.markdown

[tddstyle]:       http://martinfowler.com/articles/mocksArentStubs.html
[repourl]:        https://github.com/mspnp/cqrs-journey-code
[res-pat]:        http://www.rgoarchitects.com/nblog/2009/09/08/SOAPatternsReservations.aspx
[sbperf]:         http://msdn.microsoft.com/en-us/library/hh528527.aspx
[jsonnet]:        http://james.newtonking.com/pages/json-net.aspx
[sagapaper]:      http://www.amundsen.com/downloads/sagas.pdf

[fig1]:           images/OrderMockup.png?raw=true
[fig2]:           images/Journey_03_Aggregates_01.png?raw=true
[fig3]:           images/Journey_03_Aggregates_02.png?raw=true
[fig4]:           images/Journey_03_Aggregates_03.png?raw=true
[fig5]:           images/Journey_03_Architecture_01.png?raw=true
[fig6]:           images/Journey_03_Architecture_02.png?raw=true
[fig7]:           images/Journey_03_Sequence_01.png?raw=true
[fig8]:           images/Journey_03_ServiceBus_01.png?raw=true