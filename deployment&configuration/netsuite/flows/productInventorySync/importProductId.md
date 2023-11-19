# Sync NetSuite Product ID

Syncing the NetSuite internal ID for products from NetSuite ensures that subsequent product dependent syncs like order, transfer shipment, and cycle counts are always successful. Trying to reliably use a well known name of products like SKU or UPC, often leads to conflicts with customizations of the CSV import forms in NetSuite. To avoid having to handle these custom identification formats for each deployment, HotWax syncs NetSuite’s internal product ID.

Job data tbd