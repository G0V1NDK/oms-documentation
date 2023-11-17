# Order Approval
Order approval in ADOC involves a two-step process. Initially, the shipping address is updated through an API call, and subsequently, a new order attribute named “SHIPTO_ADDRESS_UPDATED” is generated with a value of “true” after the address update. Orders qualify for approval only if they possess order attributes for both a government mandated Customer ID and SHIPTO_ADDRESS_UPDATED. This two-criteria condition ensures that only orders with both updated customer information and shipping addresses are processed for fulfillment.

## Sync Shopify order metafields
After an order is imported, ADOC has a job enabled for importing order metafields, under the namespace “HotwaxOrderDetails”, from Shopify by an independent job as Order Attributes in HotWax Commerce.

A job in HotWax syncs order metafields from Shopify and saves them in HotWax as order attributes against that order.
```
Import Order Metafield
```

This job specifically checks order metafields under the following namespace.
```
HotwaxOrderDetails
```

This job also has an optional buffer parameter that can be used to skip orders created less than a certain duration ago. For example, setting the buffer time of 60 minutes will exclude orders that were created in the last 60 minutes.


{% hint style="info" %}
    Buffer time is set in minutes
{% endhint %}

To verify the sync is running as expected, use the Shopify GraphQL MDM. This page is located under MDM > EXIM > Shopify Jobs > Shopify GraphQL Job.

Shopify GrqphQL config
>Import Shopify Order Metafields.

The municipio name in Shopify is stored as a Metafield upon order creation, which later becomes an order attribute in HotWax Commerce. To ensure accurate shipping, this order attribute must be transferred to both the City and Zipcode fields in the shipping address before sending it to the carrier.


## Enrich shipping address
HotWax updates the address by utilizing the `updatePostalAddressContactMech` API. After a successful order address update is completed, a new order attribute “SHIP_TO_ADDRESS_UPDATED” is added to orders with the corrected address. This systematic approach ensures that the city information is correctly reflected in the shipping details provided to the carrier.

Here is a step by step process of how HotWax validates if the address values stored in order attributes are valid before adding them to the shipping address of the order.

A schedule process identifies all orders that do not have the `SHIP_TO_ADDRESS_UPDATED` attribute. The job then checks the order for either of the following attributes:
  - Municipio
  - Canton

It then checks the value of this attribute against the `CarrierGeoMapping` records. If the job is able to successfully identify a match in the `GeoName` column with the expected carrier ID, then it takes the matching value and adds it to the order’s postal address.

Once the job receives a successful response that the address has been updated in the OMS, the order ID is added to a file containing orders with successfully updated order attributes and the value of the new attribute that is to be added to them. After a batch of orders is complete, this file is imported by HotWax, and the orders have their attributes added to them. Now when a check for order approval is run, it will 

### Handling values that do not have a valid carrier code mapping
We created a list of Municipios and Cantons where Shopify’s mapping was not aligned with what the shipping carriers were expecting. This meant that even though customers thought they were choosing the right value, behind the scenes on Shopify the ID of their selection was not what the actual carriers were expecting. 

To resolve this we manually mapped all the wrong values against what the right values should be for all the countries, and then added those mappings to the `CarrierGeoMapping` table and set their carrier to the system default carrier. These records have the expected erroneous mapping in the `GeoName` column with the corrected value in the `CarrierGeoValue` column. Now with that the value is corrected, when shipping labels are requested by HotWax to the carriers, it's able to send the correct carrier code.

If the job finds a matching value but the carrier is “NA”, otherwise known as the system default carrier, then it updates the postal address of the order with the data in the `CarrierGeoValue` column.

The process of address correction does not, however, offer a fix for CSRs misspelling municipio and canton names while creating orders. Because the error in spellings is not predictable, there is no way to auto correct them using this mapping methodology.

{% hint style="info" %}
If an order cannot be automatically handled, it requires manual correction of its order attributes by a user for HotWax approval against this value.
{% endhint %}

## Approve Order
A scheduled process checks orders for two attributes:
1. “CustomerId”: (any value)
2. “SHIP_TO_ADDRESS_UPDATED”: (“true”)

All orders that have these attributes are queued to be approved by the “Approve Orders” job.

Job details:
```
Approve orders
ConfigId: IMP_APR_SALES_ORD
```

### Order Brokering and Fulfillment
ADOC doesn’t have any warehouses so it is entirely dependent on its store network for order fulfillment.
The Central American countries that ADOC operates in do not have zip codes, so instead of using numerical zip codes for distance computation and routing, the ADOC team has created their own zone mapping using their primary geo zoning system, municipios and cantons in Costa Rica.

<!-- Need to add link to zone mapping files -->

Brokering is scheduled to run once every 15 minutes, which is very important since all orders are fulfilled from stores. If orders are not allocated fast enough, it can lead to that inventory being sold in stores to walk in customers and leading to a large queue of unfillable orders.

Orders are fulfilled using the HotWax’s store Fulfillment app. HotWax integrates with separate carriers for shipping labels in each country of operation. HotWax Commerce sends all orders to Retail Pro for invoicing. Fulfillment updates for all orders are pushed into Shopify once a day or at the decided frequency.

Here is a mapping of each carrier per country

| Country      | Carrier        |
|--------------|----------------|
| El Salvador  | C807           |
| Guatemala    | Guatex         |
| Nicaragua    | Cargo Trans     |
| Costa Rica   | Terminal Express|

Other than El Salvador,  the carrier partner only provides the carrier tracking details and it is then compiled in the shipping label designed by HotWax.

The BOPIS app is also deployed in stores for fulfillment of store pickup orders. Both of these apps have also been translated into Spanish by ADOC to make them easier for their stores to use.

Reports

<!-- Need a list of all reports -->

### Syncing fulfillment updates
Fulfillment updates are synced to Shopify and Retail Pro after full order completion. The sync waits for order completion because Retail Pro can only invoice orders from one location. In order to keep both systems in sync, HotWax only notifies both systems of fulfillment on order completion. If Shopify is notified of fulfillment before Retail Pro, it will capture payment against the customers payment method before the order has been legally invoiced in Retail Pro.