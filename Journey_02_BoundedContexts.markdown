## Chapter 2
# Decomposing the Domain 

*Planning the journey: What will be the main stopping places along the way?*

# Definitions Used in This Chapter 

The following definitions are used for the remainder of this chapter. 
For more detail, and possible alternative definitions see [CQRS in 
Context][r_chapter1] in the Reference Guide. 

## Domain 

The **Domain** refers to the business domain for the Contoso Conference 
Management System (the reference implmentation). The previous chapter 
[The Contoso Conference Management System][j_chapter1] provides an 
overview of this domain. 

## Bounded Context 

The term **Bounded Context** comes from the book "Domain-Driven Design: 
Tackling Complexity in the Heart of Software" by Eric Evans. In brief, 
this concept is introduced as a way to decompose a large, complex system 
into more manageable pieces: a large system is composed of multiple 
bounded contexts. Each bounded context is the context for its own 
self-contained domain model, and has its own **Ubiquitous Language**.

A given Bounded Context could be divided into Business Components. 
A Business Component can exist in only one Bounded Context.

One bounded context typically communicates with another bounded context by raising events.

> **BharathPersona:** When you use the CQRS pattern, you often use
> events to communicate between bounded contexts. There are alternative
> approaches to integration such as sharing data at the database level.

## Context Map

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	To do - expand this section and add a definition in the Reference Guide.
  </span>
</div>  

# Bounded Contexts in the Conference Management System 

## The Orders and Registrations Bounded Context

When a registrant interacts with the system, the system creates an 
**Order** to manage the **Reservations**, payment, and 
**Registrations**. An Order contains one of more Order Items. 

A **Reservation** is a temporary reservation of one or more seats at 
conference. When a registrant begins the ordering process to purchase a 
number of seats at conference, the system creates **Reservations** for 
the number of seats requested by the registrant. These seats are then 
not available for other registrants to reserve. The **Reservations** are 
held for _n_ minutes (_n_ is one of the properties of the conference 
defined by the **Business Customer**) during which the registrant can 
complete the ordering process by making a payment for those seats. If 
the registrant does not pay for the tickets within _n_ minutes, the 
system deletes the **Reservation** and the seats become available to 
other registrants to reserve. 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
    At the moment the value of _n_ is hardcoded - this may change in the future and become a parameter than can be set by the Business Customer.
  </span>
</div>

## The Conference Management Bounded Context

## The On-site Client Application Bounded Context 

## The Payments Bounded Context 

## The Conference Feedback Bounded Context

## The Context Map for the Contoso Conference Management System

Figure 1 and the table that follows it provides a **Context Map** that 
shows the relationships between the different bounded contexts that make 
the complete system and as such it provides a high-level overview of how 
the system is put together. 

> **BharathPersona:** A frequent comment about CQRS projects is the
> difficulty in understanding how all of the pieces fit together,
> especially if there is a large number of commands and events in the
> system. Often, you can perform some static analysis on the code to
> determine where events and commands are handled, but it is more
> difficult to automatically determine where they originate. At a
> high-level, a context map can help to understand the the integration
> between the different bounded contexts and the events involved.
> Maintaining up to date documentation about the commands and events can
> offer a more detailed view. Additionally, if you have tests that use
> commands as inputs and then check for events, you can examine the
> tests to understand the expected consequences of particular commands
> (see the section on testing in [Extending and Enhancing the Orders and
> Registrations Bounded Contexts][j_chapter4] for an example of this
> style of test.

Figure 1 shows the six bounded contexts that make up the Contoso 
Conference Management System. The arrows on the diagram indicate the 
flow of data as events between them. 

![Figure 1][fig1]

**Bounded Contexts in the Contoso Conference Management System**

The following table lists the events that are associated with each of the numbered arrows.

<table border="1">
  <tr>
    <th>Integration #</th><th>Messages</th>
  </tr>
  <tr>
    <td>1</td>
    <td><b>ConferenceCreated</b> (Event)<br/>
      <b>ConferenceUpdated</b> (Event)<br/>
      <b>ConferencePublished</b> (Event)<br/>
      <b>ConferenceUnpublished</b> (Event)<br/>
      <b>SeatCreated</b> (Event)<br/>
      <b>SeatUpdated</b> (Event)<br/>
    </td>
  </tr>
  <tr>
    <td>2</td>
    <td>Attendee information - TBD</td>
  </tr>
  <tr>
    <td>3</td>
    <td>
      <b>InitiateInvoicePayment</b> (Command sent from Registration MVC controller)<br/>
      <b>InitiateThirdPartyProcessorPayment</b> (Command sent from Registration MVC controller)<br/>
    </td>
  </tr>
  <tr>
    <td>4</td>
    <td>
      <b>PaymentCompleted</b> (Event)<br/>
      <b>PaymentRejected</b> (Event)<br/>
    </td>
  </tr>
  <tr>
    <td>5</td>
    <td>
      <b>GetConferenceDescription</b> (request from client)<br/>
      ??
    </td>
  </tr>
  <tr>
    <td>6</td>
    <td>
      <b>AttendeeCheckedIn</b> (Event)<br/>
      <b>RegistrationUpdated</b> (Event)<br/>
      <b>GetConferenceStatus</b> (request from client)<br/>
    </td>
  </tr>
  <tr>
    <td>7</td>
    <td>
      <b>RegistrationAddedOrUpdated</b> (Event)<br/>
      <b>RegistrationRemoved</b> (Event)<br/>
      <b>AttendeeCheckedIn</b> (Event)<br/>
    </td>
  </tr>
  <tr>
    <td>8</td>
    <td>??</td>
  </tr>
  <tr>
    <td>9</td>
    <td>??</td>
  </tr>
  <tr>
    <td>10</td>
    <td>??</td>
  </tr>
</table>

> **BharathPersona:** Some of the events raised from the Conference
> Management bounded context are coarse-grained and contain multiple
> fields. Remember that Conference Management is a CRUD-style bounded
> context and does not raise fine-grained domain-style events.

# Why Did We Choose These Bounded Contexts? 


[j_chapter1]:     Journey_01_Introduction.markdown
[j_chapter4]:     Journey_04_ExtendingEnhancing.markdown
[r_chapter1]:     Reference_01_CQRSInContext.markdown

[fig1]:           images/Journey_02_BCs.png?raw=true