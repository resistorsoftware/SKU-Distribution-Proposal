### Discussion and Requirements

What happens when client downloads app and what functions we need out of the box:

* Pulls Account details about the merchant – name, address, phone, email – the usual
* Pulls product inventory and puts it into an SFTP – we’ll pull it from the SFTP and do stuff with it.
* The App should create a new location = SKU Distribution. This is in Setting “Locations” in Shopify. Merchant needs to be able to select SKU as a “Location” for each product
  * We need the ability to “Activate” an account because not every merchant who downloads the app will become a client.
  * 1st - SKU will need to create an account in Datex and then input the new client name in to the app’s activation page/section/whatever you want to call it.
  * For example: Paragon Fit Wear (company name) downloads the app, SKU negotiates pricing and client signs the contracts. SKU then creates “Paragon” as the new account name in Datex. In the Shopify App (admin side which SKU only has access to) we type in “Paragon” in the field (you decide what to call it: field, space, blank, whatever). We click Save. This activates the app/hooks/etc...

* Next step – Once activated and before we start importing orders, we need the ability to determine a starting date and time when we will start pulling orders. We don’t want to pull every “unfulfilled” and “Partially Fulfilled” order from the dawn of time for each of these clients.
  * Need option to pull orders based on financial Status as well = “Paid” and “Partially Refunded”

* We need the option to include or not include partially fulfilled order on initial import of order.
* Need shipping translator – as discussed before to determine what SKU Distribution shipping service we map the merchant’s shipping descriptions to.
* Need to omit gift cards
* Finally, we are ready for the initial download/import from Shopify to Datex.
* Once initial import is complete, run orders every 15 minutes for Paid and unfulfilled orders – sending those orders to Datex.
* When merchant changes an order attribute in Shopify, we push change to Datex. This would include “REFUND” or Partial Refunded. (This is based on Payment Status)
  * Partially Refunded would send a call to Datex to remove the item (product ID/SKU the Merchant is refunding. If the entire order is REFUNDED, then it sends a call/command/whatever to “CANCEL” the order altogether.  

* Datex will flag orders that have been “Hard Allocated” (API call looking to verify if flag is present or not). Hard Allocation means = no more changes can be made to those orders.
  * Not sure how we do this but would need to communicate that back the Merchant’s customer service rep (CSR) who’s making the change to the order. The message to the CSR = this can no longer be changed because the fulfillment company is in the process of fulfilling the order. Something along those lines.

* Need the ability to receive a write-back from Datex with tracking numbers and products SKUs shipped for each order. If everything in the order was not shipped, it would mark the order in Shopify as “Partially Fulfilled”, and list the items that were “Fulfilled” and leave the other units remaining = “unfulfilled”.
* Need the ability to record multiple tracking numbers for one order for orders like the example given above. (Partially Fulfilled).
* `Tags in Shopify – Datex has order status field for each Order ID. Need to send a specific status back to Shopify as a TAG = “Back Order”
* I would like the Merchant to be able send Datex (using Datex’s User Defined Fields) a note or status for orders that are back ordered and the merchant wants to ship out what’s in stock and back order the rest. We will create a User Defined Field in Datex that would be specifically for this processing. We might need to have a conversation about this.
* Inventory Sync – Datex will provide qty of each SKU that needs to write-back to Shopify. This is the final piece to complete the app. This needs to be right. With back orders, inventory and a lot of moving pieces happening at the same time, we should probably have this sync happening every 30 minutes. If we miss a unit by 1 or 2 due to a customer ordering an item at the time we’re writing back data, if we do it often enough it should even out.

I think I broke this down in segments to hopefully get off the stalled status and into development. Let me know what questions you have.

### Deliverables

There seem to be two distinct Apps in place from the requirements. A Shopify App for merchants, that provides two-way transfer of information between **SKU Distribution** and **Shopify** and a **SKU Distribution Administration App**. 

#### Shopify Merchant App

One principle deliverable is a Shopify App that can be installed in as many client shops as needed. Shopify will have to approve the App by testing it and ensuring it meets their standards and guidelines. Regardless of being **private** for only merchants you specify, or the App store, the same process of Shopify testing is done. In the meantime the App is installable on just a single test shop in the Partner account.

The App will be built with two versions, one for localhost development work, and one for production deployment to the cloud. The cloud choice remains Heroku.com. 

When a merchant installs the App, it has access to their Shopify store details, such as the name and email of the account owner. These particulars can be saved for use by SKU Distribution. The aspect of turning a merchant ON/OFF is a little tricky in that when Shopify tests the App you would not want them to experience an App that does nothing when installed. You would want them to test the stated functions if they desire. So at least in the early stages of testing, I would leave the default to ON until such time as the App is approved and ready for general use. After a successful install of the App, we can run a job that calls Datex with the merchant details and therefore there would be an analog account at Datex with the store details, and if you wanted to **match** the two, that could be a thing. Whenever the merchant authenticates the Shopify App, this handshake can be used to exchange a valid token as an example of **being connected**. So upon a **connection** being made, the App could ensure webhooks are up to date or any other housekeeping operations take place. The Datex account could setup the initial order pull to ignore those orders that remain open but are **partially fulfilled**, or to just pull all open orders regardless of their fulfillment status. Not sure how you can control the pulling of orders based on transactions but they are available, so it is likely you can chew threw the transactions to decide what to do with the items. 

It takes a special application to Shopify in the App setup details of the partner dashboard to pull orders over 60 days old from an App install. If you request this, for example a starting date more than 60 days ago, you have to justify why you need old orders like that. If you do not need to go back that far, you can just capture a calendar date presented to the merchant that signifies when to start working with orders. Orders come with financial status. This can be used when the administration App calls for orders manually.

Creating a location is accomplished when we add a **fulfillment service** to the shop, in this case **SKU Distribution**. When that is created in the merchant shop, the merchant can assign inventory to the location SKU Distribution and the product can be fulfilled by SKU Distribution. One typical mechanism for this is for the App to listen to orders paid in the merchant job. If the order contains a line item marked as **fulfillment_service: 'sku_distribution'** that item and the important order details would be added to some file loaded into a folder on the FTP folder for processing. There are other patterns of use too. The merchant can create a location manually labelled "SKU Distribution", which for all intents can then hold inventory. If an order is booked but the line item remains **fulfillment_service: 'manual'**, the App can still send the item to SKU Distribution as the location was set to SKU Distribution. 

When a merchant makes changes to orders, prior to them being marked as fulfilled, those changes can be exchanged with the fulfillment service. So if a merchant that has not marked an item as fulfilled chooses to issue a refund or cancellation, that can be send to Datex to ensure those important status changes are registered. Once an order line item is marked as fulfilled in Shopify, it cannot be edited or changed. That is a complication but there are workarounds like using Draft Orders where a change is desired to add quantity for example. On the other hand if you make some change in Datex like "HARD" or "DO NOT ALLOW EDITS" that only way that can be used to stop the Shopify merchant from editing the order, is by marking the items as fulfilled. This could mean adding a tracking number to a fulfilled item later in the lifecycle, which can be done. 

Shopify uses the concept of a fulfillment order to govern fulfillments. In this manner you can assign a tracking number to a fulfillment order that contains line items to fulfill. If the quantity of line items is not totally fulfilled, that fulfillment order is closed off with the tracking number, to inform the customer of the partial shipment, and Shopify creates a new fulfillment order with the remaining items, so they can be fulfilled at a later date with a different shipment.

#### SKU Distribution App

This App would have a list of Shopify stores. A simple filter could show stores that are signed up but not active, active stores, and perhaps closed or archived stores. Selecting a store would then present a view of the orders of the shop. Simple filtering could show open orders, partially fulfilled orders, orders with refunds, cancellations etc. A basic edit would be to alter tags in a Shopify order, to say **Back Ordered**, or other status. The merchant App could present a note area per order to be filled in, and that note field could be populated in Datex. 

#### Inventory Sync

This is an external case of computing being inside the merchant App but but also being external in the sense that that external aspect is the scheduled nature of operations. As long as a store has a reasonably small inventory, updating can be done every hour. I have had stores with 180,000 SKUs take 25 hours to update using the Shopify API system, so there are limits to inventory sync. The good news is that there is bulk inventory update at a location, meaning you can update up to 100 SKUs with one API call, making it reasonable to keep stores with accurate inventory over time. 


