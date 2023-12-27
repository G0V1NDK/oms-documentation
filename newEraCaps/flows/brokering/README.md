# Brokering

New Era Caps will use its stores for fulfillment along with their warehouse, but the conditions for when to use stores fulfillment are very particular and require that brokering be configured correctly to accommodate for these preferences.

Online shipping orders are always routed to the warehouse unless the customer has selected a store that they want the product to be shipped from when adding the item to their cart. In the event that an order item contains a soft allocated ship-from facility, the OMS will read the pre-selected facility and allocate those order items to the respective locations for fulfillment.

When this kind of soft allocated order is imported, the soft allocated items of the order will be directly allocated to that store upon import and skip the normal brokering algorithm. Because customers must explicitly choose which items they want to be shipped from store, the OMS will split the items not soft allocated and leave them in the brokering queue to be brokered to the warehouse at scheduled times.

In the event that a store cannot fulfill an order and must reject it for reallocation, those orders will not be allocated to the warehouse due to the WMS's inability to differentiate between two separate shipments of the same order when creating its CSV feed of fulfilled orders.


{% hint style="danger" %}
To support this workflow, the warehouse facility type must be passed as an excluded `type` in the Rejected Order Brokering Job.
{% endhint %}

The rigidity of the WMS software used by New Era Caps also means that if the warehouse cannot fulfill an order item, it is canceled on Shopify manually by a CSR. When CSRs preform this cancellation on Shopify, the also add a “Reshipped” tag on the order to indicate that HotWax needs to resend it to the WMS.

## How “Reshipped” works in HotWax Commerce
A flow in NiFi checks order items that have been recently (time based cursor) canceled from the warehouse and does the following: 
Add an order level attribute that helps users track Reshipped progress and identify that the order name needs to be appended with “_R”

Key: "ReShipped"
Value: "Pending"

Delete the External Fulfillment Order Item for all items in the ship group where the item was canceled from.

Because canceled items are no longer located at the facility they were brokered too, NiFi will use the Order Facility Change history to identify canceled items that were at the warehouse facility before being canceled. This entity will also contain details of which shipgroup the item was removed from, helping identify which order items to delete the fulfillment history for.

Now that the External Fulfillment Order Item record is deleted for the items that need to be reshipped, the brokered items feed will automatically include these items the next time it runs. When this feed is then consumed by NiFi to be transformed into the custom New Era Caps WMS fixed byte file format, the “ReShipped: Pending” order attributes will be used as an identifier that the order name needs to have “_R” appended to it.

The field in the WMS feed where this change is made:
CUST ORDER NO. (S-3)

Parallel to the WMS feed transformation of the brokered feed, another processor will consume the same file to update all orders with the value of the ReShipped order attribute from Pending to the new constructed order name with the appended “_R” value.

New attribute value ```{orderName}_R```