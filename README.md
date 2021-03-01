# Dynamically-Populate-Creator-Subform-Rows-and-Add-Picklist-Values
This set up allows you to [dynamically add rows](https://www.zoho.com/deluge/help/miscellaneous/insert-subform-row.html) and [add to picklist dynamically](https://www.zoho.com/creator/help/fields/add-to-picklist-dynamically.html) with values unique to each row on a Zoho Creator subform.

## Example Scenario
Suppose you have a Creator Form to help a recruitment agency place talents to job openings, where upon submission of the form, it creates a job placement record in CRM and sends an email notification to the employer via CRM. In the Creator Form, you have a subform where on addition of a row, it dynamically adds to a picklist field existing job openings which are records stored in a custom module in CRM. Each job opening is linked to a partner (an Account in CRM), and each Account is linked to several Contacts. Here's the challenge - users need to be able to select the Contact for each job opening they would like to have the email notification sent to.

## Configuration
* Create a new subform called "Employer Contact Person" with two fields:
  * Employer (single line)
  * Contact Person (dropdown/picklist)
* Create a checkbox field called "Confirm", that requires users to click on it once they have selected the job opening(s) for the talent. 
* Then, we write a script that triggers on user input of "Confirm" == true that does the following:
  * Iterate through the Job Opening subform, get the related Account and dynamically add the account name(s) as row(s) to the "Employer" field in the "Employer Contact Person" subform.
  * Get the Contact list of every related Account, and
    * Create a map of Account (key) and Contact list (value) and put it in a hidden multi-line field.
    * Create a map of each Contact Name (key) and Contact ID (value) and put it in another hidden multi-line field.
  * Update a hidden checkbox field called "Trigger", that will trigger another script to get the contact list of each Account from the hidden multi-line field, then add them as picklist values to the "Contact Person" field respective to the Employer field for each row created in the "Employer Contact Person" subform.

## Tutorial

### Script (Confirm == true)

#### Show the Subform and Set Counters
* When Confirm == true, we *show* the Employer Contact Person subform (remember to *hide* the subform on load of the form for this to work).
* *mp* is used to store the Account Name - Contact List map.
* *string* is defined as an empty string to collect values to form the Contact Name - Contact ID map.
* *employerList* is used for validation later so that the Employer Contact Person does not have repeated values if there are multiple job openings selected that belong to the same account.

```javascript
if(input.Confirm == true)
{
	show Employer_Contact_Person;
	//Counter
	mp = Map();
	string = "";
	employerList = List();
 ```
 
 #### Iterate through the Existing Job Opening subform
 * Have a criteria at the top to only iterate through the subform if it's not empty
 * While iterating through each row, add another criteria to also check that the value of the row is not null
 * If it passes through both criteria above, then we get the account ID (employer ID)
 	* Note: This configuration is not mentioned in this post. Job Openings Map is another hidden multi-line field that stores a map of the job opening name (key) and the ID (value). It's written in the same script that populates the job opening picklist value on the Existing Job Openings subform
 
 ```javascript
 	if(input.Existing_Job_Opening.toList().size() > 0)
	{
		for each  e in input.Existing_Job_Opening
		{
			jo = ifNull(e.Active_Job_Openings,"");
			if(jo != "")
			{
				//Get the Job Opening ID
				joMap = input.Job_Openings_Map.toMap();
				//info joMap;
				joID = joMap.get(employer).toLong();
				info joID;
 ```
 
 #### Get the Contact and Account Name
 * With the Job Openings ID, get the Job Openings record and get the Account ID that's related to it.
 	* This may differ based on your configuration.
 * With the Account ID, get the related Contacts and Account Name
 ```javascript
 
 				//Get the Account
				jobOpening = zoho.crm.getRecordById("Job_Openings",joID);
				accountid = jobOpening.get("Account_Lookup").get("id").toLong();
				account = zoho.crm.getRecordById("Accounts",accountid);
				//Get the Contacts for later
				contacts = zoho.crm.getRelatedRecords("Contacts","Accounts",accountid);				
				//Get the Account Name
				employerName = account.get("Account_Name");
```


#### Add the Account Name to the Employer Contact Person subform
* We define *row_n* as the variable for the subform rows
* Then have a criteria check above to only perform the function if the employer name is not in the *employerList* variable (this is to prevent duplicate rows).
* Then, inside the *if* condition, add the employer name to *employerList* and add that into the Employer Contact Person subform.

```javascript
				//Add that into the Employer Contact Person subform
				row_n = Zoho_Job_Placements_and_Intros.Employer_Contact_Person();
				if(employerName not in employerList)
				{
					row_n.Employer=employerName;
					employerList.add(employerName);
					//Declare a variable to hold the collection of rows
					update = Collection();
					update.insert(row_n);
					//Insert the rows into the subform through the variable
					input.Employer_Contact_Person.insert(update);
```

#### Create the Contact Maps and Put in the Respective Multi-Line Fields
* There are 2 maps that we need to create here:
	* Map 1 - Account Name (key), Contact Names (value)
	* Map 2 - Contact Name (key), Contact ID (value)
* For Map 1:
	* Iterate through the contact list that we have gotten earlier, get the Full Name and put it in the *contactList* list.
	* Outside of the contact loop, we put variable *mp* with the Account Name (*employerName*) as the key and *contactList* as the value.
* For Map 2:
	* In the same contact loop, use the *string* variable that we have defined earlier to create a map of the contact Full Name and ID. 
	* This map will be used later in your on-submission script to find the contact ID based on the contact selected by the user.

```javascript
					if(contacts.size() > 0)
					{
						contactList = List();
						for each  c in contacts
						{
							contactList.add(c.get("Full_Name"));
							string = string + "\"" + c.get("Full_Name") + "\"" + ":" + c.get("id") + ",";
						}
						mp.put(employerName,contactList);
					}
				}
			}
		}
	}
```

#### Update the Maps and Trigger the Hidden Checkbox
* This part of the script has to be outside the Existing Job Opening subform loop.
* Insert the *mp* variable into the Employer Contact Map hidden multi-line field
* Add "{" to the front of the *string* variable and "}" to the back to make it a map, and remove the additional "," at the back. Then, insert it into the 
Contact ID Map hidden multi-line field.
* Check the hidden "Trigger" checkbox to run [the other script](#dynamically-add-contact-list-as-picklist-values-in-the-employer-contact-person-subform) that dynamically adds to the picklist fields (this should be placed last after everything).
* Add an else condition to hide the Employer Contact Person subform and also clear it. To increase intuitiveness, we want to hide and also clear the form should the Confirm checkbox gets unchecked (this allows users to "refresh" the Employer Contact Person" table should there be any changes or mistakes.

```javascript
	//Insert Map 1
	input.Employer_Contact_Map = mp.toString();
	//Insert Map 2
	mapstring = "{" + string + "}";
	mapstring = mapstring.removeLastOccurence(",");
	input.Contact_ID_Map = mapstring;
	//Trigger the hidden checkbox
	input.Trigger = true;
}
else
{
	hide Employer_Contact_Person;
	if(input.Employer_Contact_Person.toList().size() > 0)
	{
		input.Employer_Contact_Person.clear();
	}
}
```
### Script (Trigger == true)

#### Dynamically Add Contact List as Picklist Values in the Employer Contact Person Subform
* Iterate through each row of the Employer Contact Person subform.
* For each row, get the hidden multi-line field (Employer Contact Map), convert it into a readable map with a *toMap()* function.
* Use a *get()* function to get the Contact List with the Employer Name that is already populated on the subform.
* *ui.add* function is used to dynamically add the contact list into the respective Contact Person field in the subform.

```javascript
for each  emp in Employer_Contact_Person
{
	mp = input.Employer_Contact_Map.toMap();
	emp.Contact_Person:ui.add(mp.get(emp.Employer));
}
```
