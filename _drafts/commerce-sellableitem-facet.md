---
layout: post
title: Adding custom facetable attributes to products in Sitecore Commerce 9
---

In this post, I'm going to walk through the steps involved in exposing custom data on a SellableItem as a facet (as displayed on the Storefront's category page).

Let's start with a quick summary of how all these parts are connected, since this alone would have helped immensely when I started looking into making this customisation:

* Data is added to Sellable Items via Components
* Fields are exposed to Sitecore via the ConnectSellableItem view
* Templates are updated to include the custom field
* The custom field is added to the index
* A facet item is defined on the indexed field
* The facet is associated with a Commerce category

## Add Component to SellableItem

This is the only step that I'm going to omit the details for, both because it involves a fair amount of boilerplate code and because there's a fairly exhaustive "How to" available from Sitecore: [How to extend Catalog system entities schema in Sitecore Experience Commerce 9.0](https://kb.sitecore.net/articles/083614)

In case that link becomes unavailable, it involves:

* Adding a block to `IGetEntityViewPipeline` that exposes data as "Properties" to the view and also defines "Actions" that will be handled by another pipeline when triggered
* Adding a block to `IDoActionPipeline` that updates the component based on the action and data

There are plenty of examples of these blocks in the stock plugins, so with a little help from ILSpy (or similar) it will all become clear.

## Include fields in commerce templates

The "Update Commerce Templates" button in the ribbon updates the Catalog, Category, and SellableItem templates by querying a specific view for the first available entity and then populates each item by querying the same view again for each entity. This also makes use of the `IGetEntityViewPipeline`, with the only real important differences being:

1. Child views are required (as with Master views) since they map to field sections in Sitecore
2. Properties should be added regardless of whether the data is available, as "Update Commerce Templates" generates the template fields from the first available SellableItem and that might not be relevant to your custom data.

For SellableItem, the view is `KnownCatalogViewsPolicy.ConnectSellableItem`, so we'll just include a basic block here:

```csharp
public class GetCustomDataBlock : 
    PipelineBlock<EntityView, EntityView, CommercePipelineExecutionContext>
{
    public override Task<EntityView> Run(EntityView arg, 
        CommercePipelineExecutionContext context)
    {
        var request = context.CommerceContext.GetObject<EntityViewArgument>();
        var viewsPolicy = context.GetPolicy<KnownCatalogViewsPolicy>();
        var entity = request.Entity;

        if (!(entity is SellableItem))
        {
            return Task.FromResult(arg);
        }

        var isCommerceConnectTemplateUpdate = 
            request.ViewName == viewsPolicy.ConnectSellableItem;

        if (isCommerceConnectTemplateUpdate)
        {
            var fieldSectionView = new EntityView
            {
                Name = "Fabrikam",
                DisplayName = "Fabrikam"
            };

            entityView.ChildViews.Add(fieldSectionView);

            var component = entity.HasComponent<CustomComponent>()
                ? entity.GetComponent<CustomComponent>()
                : null;

            fieldSectionView.Properties.Add(new ViewProperty
            {
                Name = "CustomData",
                DisplayName = "Custom Data",
                RawValue = (component?.CustomData ?? string.Empty),
                IsReadOnly = true,
                IsRequired = false,
                IsHidden = false
            });
        }

        return Task.FromResult(arg);
    }
}
```

In reality it's likely that you'll want to combine this block with whatever exposes these properties for the "Details" view since in many use cases the output will be the same.

Once that is deployed, simply click the "Update Commerce Templates" button and the data will be populated in the catalog items.

## Include the data in the index

The faceting queries use `sitecore_(db)_index`, so we just need to make sure the custom field is included in those. 
All of the base factes are simply defined on   `defaultSolrIndexConfiguration`, but free to be more specific if your indexing setup is more complex.

Here's an example cloned from the built in 'brand' field:

```xml
<field
    fieldName="customdata"
    storageType="YES"
    indexType="UN_TOKENIZED"
    vectorType="NO"
    boost="1f"
    returnType="text"
    settingType="Sitecore.ContentSearch.SolrProvider.SolrSearchFieldConfiguration, Sitecore.ContentSearch.SolrProvider" />
```

Be aware that using text/UN_TOKENIZED will split multi-word values into multiple facet values and display them in lowercase, so you may want to customise the field configuration if that doesn't work for you. A `returnType` of `string`, for example, will prevent any processing occuring.

You may need to rebuild your index(es), at least for the Commerce tree.

## Define a custom facet

To define a new facet, simply create an item in  `/sitecore/system/Settings/Buckets/Facets` using the `Facet` template, and set the `Field Name` property to match your indexed `fieldName` ("customdata" in our example above).

The name of the facet item isn't too important, but I tend to prefix it with the same "namespace" that I use else where. For example "Fabrikam Custom Data"

## Apply the custom facet to the category

Finally, to display the facet on a category page edit the Category item in `/Commerce/Catalogs` and add the facet to the `Runtime Search Facets` field.

There is unfortunately no installation specific "default" value for this field, so if the facet applies globally you either need to manually change each category or update the standard values for `/sitecore/templates/Commerce/Catalog/CommerceSearchSettings` and remember to change it again when you perform a future Commerce upgrade.