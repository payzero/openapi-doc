# Payzero OpenAPI Cookbook

Payzero has multiple APIs that form the essential building blocks of a cross-border payment. Based on the cooperation mode between the account provider and their clients, developers may only need to integrate partial of the APIs.

To describe the business flow easily, let’s define the following entities: 

1.	{PAY} is a payment company who consumes Payzero OpenAPI. i.e. The company/org you come from.
2.	{AP} is an account provider who has already integrated with Payzero OpenAPI and uses Payzero’s platform to manage their virtual accounts e.g. FlexWallet.
3.	{MER} is a merchant who is the client of {PAY}, it needs virtual account for their daily business.
4.	{X} is the entity trading with {MER}, it may be the customer of {MER}, or anyone else who needs to send funds to the bank account of {MER}.

The purpose of this document is from the developers of {PAY}’s point of view, illustrate how to integrate with Payzero OpenAPI. Here is the end-to-end flow of how a {MER} can get a virtual bank account from a {AP} and how the money flows.

## Step 1: Authentication
You should already get the “clientId” and “clientSecret” from Payzero. Use these parameters in the “acquire access\_token” API to get an access token first. The access token expires in 2 hours unless token forced expiration API is called explicitly. You need to save this token somewhere in your application, all the following APIs need to use this token as the http authorization header. Please refer to the API documentation.

## Step 2: Configure notification receiver url
Lots of notifications will be sent by Payzero during the process, such as merchant status notification, income transaction notification, etc. In order to receive them, you need to configure your notification receiver url in Payzero. Payzero will send notification message to this url. 

The way to configure the notification receiver url, is to call the notification-service/setup API. Please refer to the API documentation.

## Step 3: Submit merchant basic information
Say you already collected your {MER}’s basic information from somewhere (e.g. online application, or offline paperwork, etc.), then you need to submit that information to Payzero so that the {AP} can review the information for their KYC (Know Your Client) compliance process. To do this, you need to call the /merchant-service/merchant API to create the merchant in Payzero.

Many fields of this API require you to provide image id (or we can call it object id). The object id is returned by calling the object uploading API. You need to call that API multiple times, upload all the documents related one by one, and then call the merchant creation API with these image/object ids afterwards.

## Step 4: Submit merchant’s business basic information
You will get a merchant\_id from the response of step-3 API call. {AP} not only need to know the basic information of the MERCHANT (KYC), but they also need to know its business mode (Know Your Business, KYB). You need to submit the basic information of {MER}’s business through API /merchant-service/merchant/store. The main information will be website url, contact email, etc. The reason why we call it “store” instead of “business” is just because the API was developed for E-commerce scenario at very first beginning, and change this naming convention will take unnecessay efforts for not only us but also all the API consumers.

Please note that for each of the business/store submitted, {AP} will assign only one virtual bank account for it per currency. Say you submit a business/store for a {MER}, indicating it needs AUD (Australian Dollar) bank account. Then {AP} will only assign ONE AUD virtual bank account for this business/store. If the merchant needs another AUD virtual bank account, they need to submit a new business/store. Again, the way it designed like this was because the API was first used in E-commerce scenario, and each store usually have one account per currency.

## Step 5a: Handle Merchant status notification and Store status notification
Till now you already submitted all the information needed by {AP}’s compliance team. What you need to do now is just waiting for the status notification.

Once {AP} provide their opinion and comments, Payzero will collect their comments and send them to you through the notification url configured in Step-2.

Please refer to the API documentation on how to parse the notification message. 

If the status of the merchant or the business/store is REJECTED, then you can check the comments in the response. Based on the comments, you probably will ask the {MER} to submit the information again, and then you can submit the new information again to Payzero. The API of updating merchant/store information is the same as creating one. the only difference is that you call updating API with the merchant/store ID. If the status of the merchant and the business/store is APPROVED, you don’t need to do anything else but waiting

## Step 5b: Handle Virtual Account assignment notification
Please note that Step-5b and Step-5a can happened parallelly. In order to improve the efficiency of account assignment. Virtual account assignment doesn’t require the merchant and the business/store be approved. 

However, just keep in mind that without the merchant and business/store being approved, even the {MER} already started to use the bank account to collect money abroad, the transactions will be still frozen in Payzero system. 

## Step 6: Handle Incoming transaction notification
Now you get the virtual account information from Payzero and already given it to your {MER}. It uses this account to collect money abroad.  

Once there is money goes into the virtual account, Payzero will notify {PAY} by sending an income transaction message. There are two different types of income transaction message. One is “pre\_income\_notify”, the other one is “actual\_income\_notify”. For normal scenarios, you should receive both “pre\_income\_notify” and “actual\_income\_notify” for each income transaction. But let’s say the merchant’s income transaction is frozen due to invalid merchant status or suspicious payers {X}, you will only receive “pre\_income\_notify” in this case. Only when the transaction is proved to be a valid incoming transaction, then you will receive the “actual\_income\_notify”.

It’s recommended for you to only count the income transaction amount into your customer’s available balance after you receive the “actual\_income\_notify” from Payzero.

## Step 7: Send a withdraw request email to your {AP}
This step is an operation step, but it will be helpful to add it here to understand the end to end flow.

Now many of your {MER}s use the virtual accounts, they have already collected all together 1m dollars. But currently all the money still lay in the virtual bank accounts, which is provided by your {AP}. In order to get this money, your operation team need to send an email to the {AP}, asking the {AP} to send the current master balance to your company’s bank account. {AP} is supposed to send you an email stating your master account balance and make a bank transfer for you. 

You are supposed to receive the total amount of money after 1-2 days depends on the banks. 

{MER} doesn’t make withdraw request directly to {AP}, so it’s always your responsibility to maintain your {MER}s money. Since {AP} already send out your balance to you, so from data point of view, your balance at {AP} will be zero now.

Based on the charge mode between you and your AP, the money received from your {AP} may be the amount after {AP} fee or before {AP} fee.


