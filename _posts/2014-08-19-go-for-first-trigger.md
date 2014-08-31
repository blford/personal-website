---
layout: post
title:  "go for first trigger"
date:   2014-08-19 18:24:00
permalink: /go-for-first-trigger/
tags: salesforce, trigger
---
My first trigger's in production! I've deployed a handful of triggers and test classes over the last few weeks, but they all started with a little copy & paste. Admittedly, this is a basic trigger, but you've got to start somewhere.

### Problem
There's an approval process on the child object of a master-detail relationship. When FIELD A is marked TRUE by the approval process, FIELD B on the parent object needs to be updated. A workflow rule doesn't work for this scenario for the following reason:

>An approval process can specify a field update action that re-evaluates workflow rules for the updated object. If, however, the re-evaluated workflow rules include a cross-object field update, those cross-object field updates will be ignored.
  
  
### Solution


	{% highlight java %}
	trigger ppsCheckbox on Partner_Payment_Submission__c (before update) {
	
	    Set<Id> oppIds = new Set<Id>();
	    
	    for(Partner_Payment_Submission__c pps : trigger.new) {
	        if (pps.Direct_to_Partner_Spiff__c && pps.Status__c == '4-Approved & Waiting on Bank Information') 
	        {
	        	oppIds.add(pps.Influenced_Opportunity__c);
	        }
	    }
	    
	    List<Opportunity> oppList = new List<Opportunity>([SELECT Id, Partner_Spiff_Qualified__c FROM Opportunity WHERE Id IN :oppIds]);
	    
	    for (Opportunity opp : oppList) {
	        opp.Partner_Spiff_Qualified__c = true;
	    }
	    
	    try
	    {
	    	if (!oppList.isEmpty())
	        {
	            update oppList;    
	        }
	            
	    }
	    catch (Exception e) {}
   
	}
	{% endhighlight %}

#### Test Class
I added the test to an existing opportunity test class, but here's the block specific to this trigger

	{% highlight java %}
	pps = [SELECT Id, Status__c, Direct_to_Partner_Spiff__c, Influenced_Opportunity__r.Partner_Spiff_Qualified__c
    	FROM Partner_Payment_Submission__c
    	WHERE Id = :pps.id];
    pps.Status__c = '4-Approved & Waiting on Bank Informaiton';
    update pps;
    
    pps = [SELECT Id, Status__c, Influenced_Opportunity__r.Partner_Spiff_Qualified__c
    	FROM Partner_Payment_Submission__c
    	WHERE Id = :pps.Id];
    System.assert(pps.Influenced_Opportunity__r.Partner_Spiff_Qualified__c);
    {% endhighlight %}