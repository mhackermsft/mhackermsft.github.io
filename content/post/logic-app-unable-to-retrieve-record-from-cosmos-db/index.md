---
title: "Logic App: Unable to retrieve record from Cosmos DB."
date: 2023-05-11T12:55:07.012Z
lastmod: 2023-05-11T12:58:27.879Z
author: Mike Hacker
url: /logic-app-unable-to-retrieve-record-from-cosmos-db/
tags:
  - 'Articles'
summary: 'Recently while I was working on a customer demo I ran into a bit of an issue when trying to delete a record from a Cosmos DB using Logic Apps. I kept receivi...'
draft: false
image: cover.jpg
---

Recently while I was working on a customer demo I ran into a bit of an issue when trying to delete a record from a Cosmos DB using Logic Apps.  I kept receiving an error indicating that the document id I was specifying didn't exist even though I could see it was in the database. I was also running into similar issues when trying to retrieve a specific document from the database.

The Logic Apps actions I was using included the *Delete a document (V2) and Get a document (V2).*  

I searched through documentation and found nothing that seemed to give any clear indication as to what the error was. I did some searches using Bing and found other folks who were having similar challenges but I didn't run across any one specific way to fix it. Everyone seemed to just be guessing as to what the issue was. 

In those actions, there are required fields which include Cosmos DB account name, database ID, Collection ID and Document ID.  In most cases you will need to also add the parameter Partition key value in order for the delete or get actions to find the proper document.  In my database, the Document ID and Partition key values are the same. 

My Logic app is acting as a REST API to do basic CRUD (create, read, update and delete) operations.  When a client calls the REST API it passes in a value which is stored as the variable id, which represents the id of the document I wish to get or delete.  Originally I was just taking that id variable and placing it in both the Document ID and partition key value fields in the Logic Apps action. As logical as that sounds, this did not work and I kept getting errors related to the document not being found.

So what ultimately fixed the issue?  The Partition key value needs to be surrounded by double quotes.  That's it.  Once I placed double quotes around this field, my Logic Apps started functioning correctly.

Hopefully this little nugget of information comes in handy for someone else who is running into similar challenges with the Cosmos DB actions for Logic Apps.
