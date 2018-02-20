---
layout: post
title: "Decomposing Monolothic ERP Exports using Sitecore Commerce 9 Pipelines"
date: 2018-02-21 07:00:16.000000000 +11:00
type: post
published: true
status: publish
tags:
- sitecore
- experience-commerce-9
author:
  login: richardszalay
  email: richard@richardszalay.com
  display_name: Richard Szalay
  first_name: ''
  last_name: ''
---

Exporting orders is a critical part of integrating an ecommerce website with an ERP system, and the integration often takes the form of an XML file or some other large data structure.

Sitecore's Experience Commerce 9 finally brings the entire model into Sitecore Engine with it's testable architecture, but generating a complete XML structure from the available data involves numerous dependencies:

* Billing information is obtained via FederatedPaymentComponent on the Order
* Shipping information is obtained via PhysicalFulfillmentComponent on the Order, but this can change for more complex shipping scenarios
* Customer contact address information is obtained via a AddressComponent on the Customer, which is in turn dereferenced with GetCustomerCommand using the customer id extracted from the ContactComponent on the Order

Attempting to test a monolothic implementation of such a mapping would involve significant setup of both data and mocks, resulting in tests that pass but that are difficult to understand and maintain. Refactoring the mapping process into a series of services that make use of Dependency Injection would help, but isn't particularly in line with Commerce Engine's architecture.

This article will explore making use of Sitecore Commerce Pipelines to decompose the mapping process. Pipelines are not only composable but also individually support dependency injection, and using pipeline blocks to mutate the pipeline arguments has significant precence within the core plugins.

Let's start with a monolothic implementation. It's still testable due to the use of interfaces and the separation of the mapping into its own public method, but there's a lot happening which means a lot to setup in our tests.

{% highlight csharp %}
public class MyErpOrderExport
    : PipelineBlock<Order, Order, CommercePipelineExecutionContext>
{
    private readonly GetCustomerCommand getCustomerCommand;
    private readonly ISubmitMyErpFilePipeline submitMyErpFilePipeline

    public MyErpOrderExport(GetCustomerCommand getCustomerCommand, 
        ISubmitMyErpFilePipeline submitMyErpFilePipeline)
    {
        this.getCustomerCommand = getCustomerCommand;
        this.submitMyErpFilePipeline = submitMyErpFilePipeline;
    }

    public async Task<Order> Run(Order order, CommercePipelineExecutionContext context)
    {
        var document = await CreateOrderDocument(order);
        await fileSubmissionPipeline.Run(document, context);
        return order;
    }

    public async Task<XDocument> CreateOrderElement(
        Order order, CommercePipelineExecutionContext context)
    {
        return new XDocument(
            new XElement("order",
                await CreateCustomerElement(order, context),
                await CreateBillingElement(order),
                await CreateShippingElement(order),
                await CreateLineItemsElement(order),
            )
        );
    }

    private async Task<XElement> CreateCustomerElement(Order order, 
        CommercePipelineExecutionContext context)
    {
        var contactComponent = arg.Order.GetComponent<ContactComponent>();
        var customerId = contactComponent.CustomerId;

        var customer = await getCustomerCommand.Process(
            context.CommerceContext, customerId);

        var address = customer.GetComponent<AddressComponent>();
        var party = address.Party;

        return new XElement("customer",
            new XElement("firstName", party.FirstName),
            // ...
        );
    }

    private Task<XElement> CreateBillingElement(Order order)
    {
        // ...
    }

    private Task<XElement> CreateShippingElement(Order order)
    {
        // ...
    }

    private Task<XElement> CreateLineItemsElement(Order order)
    {
        // ...
    }
}
{% endhighlight %}

Now, you could argue that a few more of these methods could be made public to increase testability, and you'd be right, but it wouldn't diminish the fact that this ends up being a big class does a lot, increasing the cognitive load required to reason about it.

Let's look at what it might look like if we built up the XML in pipeline blocks instead:

{% highlight csharp %}
public class MyErpOrderExport
    : PipelineBlock<Order, Order, CommercePipelineExecutionContext>
{
    private readonly IBuildErpOrderDocumentPipeline buildErpOrderDocumentPipeline;
    private readonly ISubmitMyErpFilePipeline submitMyErpFilePipeline;

    public MyErpOrderExport(IBuildErpOrderDocumentPipeline buildErpOrderDocumentPipeline,
        ISubmitMyErpFilePipeline submitMyErpFilePipeline)
    {
        this.buildErpOrderDocumentPipeline = buildErpOrderDocumentPipeline;
        this.submitMyErpFilePipeline = submitMyErpFilePipeline;
    }

    public async Task<Order> Run(Order order, CommercePipelineExecutionContext context)
    {
        var document = await CreateOrderDocument(order);

        await submitMyErpFilePipeline.Run(document, context);

        return order;
    }

    public async Task<XDocument> CreateOrderElement(Order order, CommercePipelineExecutionContext context)
    {
        var args = new BuildMyErpOrderDocumentArgument
        {
            Order = order,
            Element = new XElement("order")
        };

        await buildErpOrderDocumentPipeline.Run(args, context);

        return new XDocument(args.Element);
    }
}
{% endhighlight %}

Our top level orchestrator now simply moves data between two pipelines interfaces, making it very testable itself, but the real value is how much easier it is to manage each aspect of the document being built. Here's `AddBillingElementBlock`:

{% highlight csharp %}
public class AddBillingElementBlock : PipelineBlock<BuildMyErpOrderDocumentArgument, BuildMyErpOrderDocumentArgument, CommercePipelineExecutionContext>
{
    public Task<BuildMyErpOrderDocumentArgument> Run(BuildMyErpOrderDocumentArgument args, CommercePipelineExecutionContext context)
    {
        // Ain't no party like a...
        var billingParty = arg.Order.GetComponent<FederatedPaymentComponent>().BillingParty;

        arg.Element.Add(
            new XElement("billingInformation",
                new XElement("address1", billingParty.Address1),
                // ...
            )
        );

        return Task.FromResult(arg);
    }
}
{% endhighlight %}

Each block in the pipeline can now add its own elements to the document, and can be tested in isolation.

And finally, we compose all the blocks together in `ConfigureSitecore`:

{% highlight csharp %}
services.Sitecore().Pipelines(p =>
    p.AddPipeline<IBuildMyErpOrderDocument, BuildMyErpOrderDocument>(c => c
        .Add<AddCustomerElementBlock>()
        .Add<AddBillingElementBlock>()
        .Add<AddShippingElementBlock>()
    )
);
{% endhighlight %}