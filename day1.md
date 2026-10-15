# Salesforce Apex + LWC: Account Related Data

## Goal

Pass a list of Account IDs to Apex, fetch each Account's related Contacts, Opportunities, and Cases, group the records in a wrapper class, and display the results in LWC datatables.

> Assumption: “code” meant Salesforce **Cases**. If you meant a custom object such as `Code__c`, change the query and relationship field accordingly.

## 1. Apex class — `AccountRelatedDataController.cls`

```java
public with sharing class AccountRelatedDataController {

    @AuraEnabled(cacheable=true)
    public static List<AccountWrapper> getAccountRelatedData(
        List<Id> accountIds
    ) {
        List<AccountWrapper> result = new List<AccountWrapper>();

        if (accountIds == null || accountIds.isEmpty()) {
            return result;
        }

        Map<Id, AccountWrapper> wrapperMap =
            new Map<Id, AccountWrapper>();

        // Fetch Accounts and create one wrapper for each Account.
        for (Account acc : [
            SELECT Id, Name, Industry, Phone
            FROM Account
            WHERE Id IN :accountIds
        ]) {
            AccountWrapper wrapper = new AccountWrapper();
            wrapper.accountRecord = acc;
            wrapper.contacts = new List<Contact>();
            wrapper.opportunities = new List<Opportunity>();
            wrapper.cases = new List<Case>();

            wrapperMap.put(acc.Id, wrapper);
        }

        // Fetch and group Contacts.
        for (Contact con : [
            SELECT Id, Name, Email, Phone, AccountId
            FROM Contact
            WHERE AccountId IN :wrapperMap.keySet()
        ]) {
            wrapperMap.get(con.AccountId).contacts.add(con);
        }

        // Fetch and group Opportunities.
        for (Opportunity opp : [
            SELECT Id, Name, StageName, Amount, CloseDate, AccountId
            FROM Opportunity
            WHERE AccountId IN :wrapperMap.keySet()
        ]) {
            wrapperMap.get(opp.AccountId).opportunities.add(opp);
        }

        // Fetch and group Cases.
        for (Case c : [
            SELECT Id, CaseNumber, Subject, Status, Priority, AccountId
            FROM Case
            WHERE AccountId IN :wrapperMap.keySet()
        ]) {
            wrapperMap.get(c.AccountId).cases.add(c);
        }

        // Preserve the order of the supplied IDs; skip missing/inaccessible Accounts.
        for (Id accountId : accountIds) {
            if (wrapperMap.containsKey(accountId)) {
                result.add(wrapperMap.get(accountId));
            }
        }

        return result;
    }

    public class AccountWrapper {
        @AuraEnabled public Account accountRecord { get; set; }
        @AuraEnabled public List<Contact> contacts { get; set; }
        @AuraEnabled public List<Opportunity> opportunities { get; set; }
        @AuraEnabled public List<Case> cases { get; set; }
    }
}
```

### Key logic

- **Input:** `List<Id> accountIds`
- **Map:** `Map<Id, AccountWrapper>` associates each Account ID with its wrapper.
- **Grouping:** A related record's `AccountId` identifies which wrapper receives it.
- **Bulkification:** SOQL queries are outside loops. The method uses up to four queries regardless of the number of Accounts.
- **Wrapper:** Packages one Account plus its related record lists into a single return object.
- **Sharing/security:** `with sharing` respects record-level sharing, but does not automatically enforce object permissions or field-level security. Review CRUD/FLS before production use.

## 2. Apex test class — `AccountRelatedDataControllerTest.cls`

```java
@IsTest
private class AccountRelatedDataControllerTest {

    @IsTest
    static void testGetAccountRelatedData() {
        // Create Accounts.
        Account acc1 = new Account(Name = 'ABC Company');
        Account acc2 = new Account(Name = 'XYZ Company');
        insert new List<Account>{ acc1, acc2 };

        // Create Contacts.
        Contact con1 = new Contact(
            LastName = 'Rahul',
            Email = 'rahul@example.com',
            AccountId = acc1.Id
        );
        Contact con2 = new Contact(
            LastName = 'Priya',
            Email = 'priya@example.com',
            AccountId = acc1.Id
        );
        Contact con3 = new Contact(
            LastName = 'Amit',
            Email = 'amit@example.com',
            AccountId = acc2.Id
        );
        insert new List<Contact>{ con1, con2, con3 };

        // Create Opportunities.
        Opportunity opp1 = new Opportunity(
            Name = 'ABC Deal',
            AccountId = acc1.Id,
            StageName = 'Prospecting',
            CloseDate = Date.today().addDays(30),
            Amount = 50000
        );
        Opportunity opp2 = new Opportunity(
            Name = 'XYZ Deal',
            AccountId = acc2.Id,
            StageName = 'Prospecting',
            CloseDate = Date.today().addDays(30),
            Amount = 75000
        );
        insert new List<Opportunity>{ opp1, opp2 };

        // Create Cases.
        Case case1 = new Case(
            AccountId = acc1.Id,
            Subject = 'Login issue',
            Status = 'New',
            Origin = 'Phone'
        );
        Case case2 = new Case(
            AccountId = acc2.Id,
            Subject = 'Payment issue',
            Status = 'New',
            Origin = 'Phone'
        );
        insert new List<Case>{ case1, case2 };

        Test.startTest();
        List<AccountRelatedDataController.AccountWrapper> result =
            AccountRelatedDataController.getAccountRelatedData(
                new List<Id>{ acc1.Id, acc2.Id }
            );
        Test.stopTest();

        System.assertEquals(2, result.size());
        System.assertEquals('ABC Company', result[0].accountRecord.Name);
        System.assertEquals(2, result[0].contacts.size());
        System.assertEquals(1, result[0].opportunities.size());
        System.assertEquals(1, result[0].cases.size());

        System.assertEquals('XYZ Company', result[1].accountRecord.Name);
        System.assertEquals(1, result[1].contacts.size());
        System.assertEquals(1, result[1].opportunities.size());
        System.assertEquals(1, result[1].cases.size());
    }

    @IsTest
    static void testEmptyInput() {
        List<AccountRelatedDataController.AccountWrapper> result =
            AccountRelatedDataController.getAccountRelatedData(
                new List<Id>()
            );
        System.assertEquals(0, result.size());
    }

    @IsTest
    static void testNullInput() {
        List<AccountRelatedDataController.AccountWrapper> result =
            AccountRelatedDataController.getAccountRelatedData(null);
        System.assertEquals(0, result.size());
    }
}
```

> If your org has custom validation rules or additional required fields, populate those fields in the test records.

## 3. LWC folder structure

```text
force-app/main/default/
├── classes/
│   ├── AccountRelatedDataController.cls
│   └── AccountRelatedDataControllerTest.cls
└── lwc/
    └── accountRelatedData/
        ├── accountRelatedData.html
        ├── accountRelatedData.js
        └── accountRelatedData.js-meta.xml
```

## 4. HTML — `accountRelatedData.html`

```html
<template>
    <lightning-card title="Account Related Records" icon-name="standard:account">
        <div class="slds-p-around_medium">
            <lightning-textarea
                label="Account IDs"
                placeholder="Enter Account IDs separated by commas"
                value={accountIdsInput}
                onchange={handleAccountIdsChange}
                required>
            </lightning-textarea>

            <div class="slds-m-top_medium">
                <lightning-button
                    label="Fetch Records"
                    variant="brand"
                    onclick={handleFetch}
                    disabled={isLoading}>
                </lightning-button>
            </div>

            <template lwc:if={isLoading}>
                <lightning-spinner alternative-text="Loading records" size="medium">
                </lightning-spinner>
            </template>

            <template lwc:if={hasResults}>
                <template for:each={accounts} for:item="item">
                    <div key={item.accountRecord.Id} class="slds-box slds-m-top_large">
                        <h2 class="slds-text-heading_medium">
                            {item.accountRecord.Name}
                        </h2>
                        <p class="slds-m-top_x-small">
                            Industry: {item.accountRecord.Industry}
                        </p>
                        <p>Phone: {item.accountRecord.Phone}</p>

                        <h3 class="slds-text-heading_small slds-m-top_large">
                            Contacts ({item.contacts.length})
                        </h3>
                        <lightning-datatable
                            key-field="Id"
                            data={item.contacts}
                            columns={contactColumns}
                            hide-checkbox-column>
                        </lightning-datatable>

                        <h3 class="slds-text-heading_small slds-m-top_large">
                            Opportunities ({item.opportunities.length})
                        </h3>
                        <lightning-datatable
                            key-field="Id"
                            data={item.opportunities}
                            columns={opportunityColumns}
                            hide-checkbox-column>
                        </lightning-datatable>

                        <h3 class="slds-text-heading_small slds-m-top_large">
                            Cases ({item.cases.length})
                        </h3>
                        <lightning-datatable
                            key-field="Id"
                            data={item.cases}
                            columns={caseColumns}
                            hide-checkbox-column>
                        </lightning-datatable>
                    </div>
                </template>
            </template>

            <template lwc:if={showEmptyMessage}>
                <p class="slds-m-top_medium">
                    No Accounts found for the supplied IDs.
                </p>
            </template>
        </div>
    </lightning-card>
</template>
```

## 5. JavaScript — `accountRelatedData.js`

```javascript
import { LightningElement } from 'lwc';
import getAccountRelatedData from
    '@salesforce/apex/AccountRelatedDataController.getAccountRelatedData';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

const CONTACT_COLUMNS = [
    { label: 'Name', fieldName: 'Name' },
    { label: 'Email', fieldName: 'Email', type: 'email' },
    { label: 'Phone', fieldName: 'Phone', type: 'phone' }
];

const OPPORTUNITY_COLUMNS = [
    { label: 'Opportunity Name', fieldName: 'Name' },
    { label: 'Stage', fieldName: 'StageName' },
    { label: 'Amount', fieldName: 'Amount', type: 'currency' },
    { label: 'Close Date', fieldName: 'CloseDate', type: 'date' }
];

const CASE_COLUMNS = [
    { label: 'Case Number', fieldName: 'CaseNumber' },
    { label: 'Subject', fieldName: 'Subject' },
    { label: 'Status', fieldName: 'Status' },
    { label: 'Priority', fieldName: 'Priority' }
];

export default class AccountRelatedData extends LightningElement {
    accountIdsInput = '';
    accounts = [];
    isLoading = false;
    hasSearched = false;

    contactColumns = CONTACT_COLUMNS;
    opportunityColumns = OPPORTUNITY_COLUMNS;
    caseColumns = CASE_COLUMNS;

    handleAccountIdsChange(event) {
        this.accountIdsInput = event.target.value;
    }

    async handleFetch() {
        const accountIds = [
            ...new Set(
                this.accountIdsInput
                    .split(',')
                    .map(id => id.trim())
                    .filter(id => id !== '')
            )
        ];

        if (accountIds.length === 0) {
            this.showToast(
                'Missing Account IDs',
                'Please enter at least one Account ID.',
                'warning'
            );
            return;
        }

        this.isLoading = true;
        this.hasSearched = false;
        this.accounts = [];

        try {
            const result = await getAccountRelatedData({
                accountIds: accountIds
            });

            this.accounts = result || [];
            this.hasSearched = true;

            if (this.accounts.length === 0) {
                this.showToast(
                    'No Results',
                    'No matching Accounts were found.',
                    'info'
                );
            }
        } catch (error) {
            this.showToast('Error', this.getErrorMessage(error), 'error');
        } finally {
            this.isLoading = false;
        }
    }

    get hasResults() {
        return this.accounts.length > 0;
    }

    get showEmptyMessage() {
        return this.hasSearched && this.accounts.length === 0;
    }

    getErrorMessage(error) {
        if (error.body && error.body.message) {
            return error.body.message;
        }
        return 'An unexpected error occurred while fetching records.';
    }

    showToast(title, message, variant) {
        this.dispatchEvent(
            new ShowToastEvent({ title, message, variant })
        );
    }
}
```

## 6. Metadata — `accountRelatedData.js-meta.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>65.0</apiVersion>
    <isExposed>true</isExposed>
    <masterLabel>Account Related Data</masterLabel>
    <description>
        Displays Accounts and their related Contacts, Opportunities, and Cases.
    </description>
    <targets>
        <target>lightning__AppPage</target>
        <target>lightning__HomePage</target>
        <target>lightning__RecordPage</target>
    </targets>
</LightningComponentBundle>
```

Use an API version supported by your Salesforce org.

## 7. Deploy and run

1. Save all Apex and LWC files in VS Code.
2. Deploy the files to your Salesforce org.
3. Run `AccountRelatedDataControllerTest` from the Developer Console or VS Code.
4. Open **Setup → Lightning App Builder**.
5. Add **Account Related Data** to an App, Home, or Record page.
6. Save and activate the page.
7. Enter Account IDs separated by commas and click **Fetch Records**.

## 8. Data flow summary

```text
LWC input: comma-separated Account IDs
        ↓
JavaScript splits IDs into an array
        ↓
Calls getAccountRelatedData({ accountIds })
        ↓
Apex queries Accounts, Contacts, Opportunities, Cases
        ↓
Map<AccountId, AccountWrapper> groups related records
        ↓
Returns List<AccountWrapper>
        ↓
LWC renders each Account and its three datatables
```

### Key concepts

- `@AuraEnabled`: Exposes Apex methods/properties to Lightning.
- `@AuraEnabled(cacheable=true)`: Allows caching for this read-only method when called through supported mechanisms.
- `with sharing`: Enforces record-level sharing, not CRUD/FLS by itself.
- `Map<Id, AccountWrapper>`: Helps group child records under the correct Account.
- `lightning-datatable`: Displays each related record list in a table.
- `for:each`: Renders one Account section per wrapper returned.
- `Test.startTest()` / `Test.stopTest()`: Delimit the code under test and allow asynchronous work to be processed where applicable.
