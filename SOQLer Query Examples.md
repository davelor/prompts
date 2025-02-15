| Type of Search | Examples |
|---------------|----------|
| Simple query | `SELECT Id, Name, BillingCity FROM Account` |
| WHERE | `SELECT Id FROM Contact WHERE Name LIKE 'A%' AND MailingCity = 'California'` |
| ORDER BY | `SELECT Name FROM Account ORDER BY Name DESC NULLS LAST` |
| LIMIT | `SELECT Name FROM Account WHERE Industry = 'media' LIMIT 125` |
| ORDER BY with LIMIT | `SELECT Name FROM Account WHERE Industry = 'media' ORDER BY BillingPostalCode ASC NULLS LAST LIMIT 125` |
| count() | `SELECT COUNT() FROM Contact` |
| GROUP BY | `SELECT LeadSource, COUNT(Name) FROM Lead GROUP BY LeadSource` |
| HAVING | `SELECT Name, COUNT(Id) FROM Account GROUP BY Name HAVING COUNT(Id) > 1` |
| OFFSET with ORDER BY | `SELECT Name, Id FROM Merchandise__c ORDER BY Name OFFSET 100` |
| OFFSET with ORDER BY and LIMIT | `SELECT Name, Id FROM Merchandise__c ORDER BY Name LIMIT 20 OFFSET 100` |
| Relationship queries: child-to-parent | `SELECT Contact.FirstName, Contact.Account.Name FROM Contact`<br>`SELECT Id, Name, Account.Name FROM Contact WHERE Account.Industry = 'media'` |
| Relationship queries: parent-to-child | `SELECT Name, (SELECT LastName FROM Contacts) FROM Account`<br>`SELECT Account.Name, (SELECT Contact.LastName FROM Account.Contacts) FROM Account` |
| Relationship query with WHERE | `SELECT Name, (SELECT LastName FROM Contacts WHERE CreatedBy.Alias = 'x') FROM Account WHERE Industry = 'media'` |
| Relationship query: child-to-parent with custom objects | `SELECT Id, FirstName__c, Mother_of_Child__r.FirstName__c FROM Daughter__c WHERE Mother_of_Child__r.LastName__c LIKE 'C%'` |
| Relationship query: parent-to-child with custom objects | `SELECT Name, (SELECT Name FROM Line_Items__r) FROM Merchandise__c WHERE Name LIKE ‘Acme%’` |
| Relationship queries with polymorphic key | `SELECT Id, Owner.Name FROM Task WHERE Owner.FirstName like 'B%'`<br>`SELECT Id, Who.FirstName, Who.LastName FROM Task WHERE Owner.FirstName LIKE 'B%'`<br>`SELECT Id, What.Name FROM Event` |
| Polymorphic relationship queries using TYPEOF | `SELECT TYPEOF What WHEN Account THEN Phone, NumberOfEmployees WHEN Opportunity THEN Amount, CloseDate ELSE Name, Email END FROM Event` |
| Relationship queries with aggregate | `SELECT Name, (SELECT CreatedBy.Name FROM Notes) FROM Account`<br>`SELECT Amount, Id, Name, (SELECT Quantity, ListPrice, PricebookEntry.UnitPrice, PricebookEntry.Name FROM OpportunityLineItems) FROM Opportunity` |
| Simple query: the UserId and LoginTime for each user | `SELECT UserId, LoginTime from LoginHistory` |
| Relationship queries with number of logins per user in a specific time range | `SELECT UserId, COUNT(Id) from LoginHistory WHERE LoginTime > 2010-09-20T22:16:30.000Z AND LoginTime < 2010-09-21T22:16:30.000Z GROUP BY UserId` |
| Query to retrieve the RecordType per user | `SELECT Id, Name, IsActive, SobjectType, DeveloperName, Description FROM RecordType` |