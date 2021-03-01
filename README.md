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

### Script (Trigger: Confirm == true)

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
