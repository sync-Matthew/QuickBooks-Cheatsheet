# QuickBooks-Cheatsheet
How to: Add QuickBooks online integration to a Django project. Development and Production versions.  

<h3>Useful links:</h3>  

[Python-Quickbooks Github Reference](https://github.com/ej2/python-quickbooks)  

[QuickBooks OAuth2.0 Tutorial](https://developer.intuit.com/app/developer/qbo/docs/develop/authentication-and-authorization/oauth-2.0)  

[QuickBooks API Reference](https://developer.intuit.com/app/developer/qbo/docs/api/accounting/most-commonly-used/account)  

<h3>Getting Started</h3>  

Before you can begin development, you'll first need to create a QuickBooks developer account:  

[Intuit Developer Sign Up](https://accounts.intuit.com/signup.html?offering_id=Intuit.devx.devx&iux_target_aal=25&iux_sso_mfa=true&redirect_url=https%3A%2F%2Fdeveloper.intuit.com%2Fapp%2Fdeveloper%2Fqbo%2Fdocs%2Fdevelop%2Fauthentication-and-authorization%2Foauth-2.0%3FdevXlogin%3Dtrue)  

After creating your account, navigate to the "Dashboard" tab and create your first App:  

![image](https://github.com/sync-Matthew/QuickBooks-Cheatsheet/assets/109091963/e23a32f3-5047-447a-a961-ba4f2cf936cc)  

Click into your new App, and navigate to:  
`Development Settings` --> `Keys & credentials`  
Take note of:
1. `Client ID`  
2. `Client Secret`  
3. `Redirect URIs`  

Add the following `Redirect URI`:  
`http://localhost:8000/qbo/callback`

*After our user is authorized, they will be redirected to the given URI.  

Afterwards, add a new sandbox company from the "API Docs & Tools" tab:  

![image](https://github.com/sync-Matthew/QuickBooks-Cheatsheet/assets/109091963/e217229a-f58c-4800-867a-a5b56f0d0185)  

*Take note of your sandbox company's ID as we'll need that later:  

![image](https://github.com/sync-Matthew/QuickBooks-Cheatsheet/assets/109091963/7cca1b5a-6d08-41a8-8667-c68d5c62f987)  

Finally, initialize your virtual environment and create your new Django project:  
`python -m venv .venv`  
`. .venv/bin/activate`  
`pip install django`  
`django-admin startproject your_project .`  

<h1> Development </h1>

1. Install `intuit-oauth` and `python-quickbooks` to your virtual environment:  

`pip install intuit-oauth`  
`pip install python-quickbooks`  

2. From `your_app/settings.py`, add the following credentials:  

```
# your_app/settings.py
# ...
# This is QBO's equivalent of DEBUG=True
QUICKBOOKS_SANDBOX_MODE = True

# Quickbooks sandbox credentials
QUICKBOOKS_CLIENT_ID = 'your_client_id'
QUICKBOOKS_CLIENT_SECRET = 'your_client_secret'
QUICKBOOKS_REDIRECT_URI = 'http://localhost:8000/qbo/callback'

QUICKBOOKS_COMPANY_ID = 'your_company_id'

# ...
```  

3. Create a new app called `qbo`:  

`django-admin startapp qbo`  

*Remember to add the app to your INSTALLED_APPS in `your_app/settings.py`  

4. From `qbo/views.py`, add the view which will handle the authorization process:  

```
from django.shortcuts import render, redirect
from django.conf import settings
from intuitlib.client import AuthClient
from intuitlib.enums import Scopes

# This function is responsible for handling the initial authorization process to QBO.  
# Note* redirect(auth_url) will redirect the user to wherever you've set your 'Redirect URI`.  
# If you encounter any errors check both your QBO settings and your 'your_app/settings.py' REDIRECT_URI  
def authorize_quickbooks(request):

    auth_client = AuthClient(
      client_id=(settings.QUICKBOOKS_CLIENT_ID if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_CLIENT_ID),
      client_secret=(settings.QUICKBOOKS_CLIENT_SECRET if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_CLIENT_SECRET),
      environment=('sandbox' if settings.QUICKBOOKS_SANDBOX_MODE else 'production'),
      redirect_uri=(settings.QUICKBOOKS_REDIRECT_URI if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_REDIRECT_URI),
    )

    auth_url = auth_client.get_authorization_url([Scopes.ACCOUNTING])
    request.session['state'] = auth_client.state_token

    return redirect(auth_url)
```  

**In order to initiate this process, you will need to create a separate view that renders a template with some piece of UI that will direct the user to our newly created authorize view.

5. From `qbo/views.py`, add the view which will handle the callback process after authorization:  

Within this view, you can optionally store your token data within a **Model** to prevent the need for constant user authorization during API calls.

```
def get_quickbooks_client(request):
    auth_client = AuthClient(
        client_id=(settings.QUICKBOOKS_CLIENT_ID if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_CLIENT_ID),
        client_secret=(settings.QUICKBOOKS_CLIENT_SECRET if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_CLIENT_SECRET),
        environment=('sandbox' if settings.QUICKBOOKS_SANDBOX_MODE else 'production'),
        redirect_uri=(settings.QUICKBOOKS_REDIRECT_URI if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_REDIRECT_URI),
    )
    
    auth_code = request.GET.get('code', None)
    realm_id = request.GET.get('realmId', None)
    
    auth_client.get_bearer_token(auth_code, realm_id=realm_id)
    access_token = auth_client.access_token
    refresh_token = auth_client.refresh_token
    
    # Add logic for storing your token here
    
    return redirect('your-url-here')
```  

<h2>How to: Refresh an expired access token</h2>  

0. Create your model to store the token information:  

```
from django.db import models
from some_app.models import CustomUser as User

# Create a QuickBooks Token model with a foreign key to the User model
class QuickBooksToken(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, null=True, blank=True)
    identifier = models.CharField(max_length=255, unique=True)
    access_token = models.CharField(max_length=255)
    refresh_token = models.CharField(max_length=255)
    realm_id = models.CharField(max_length=255)
    auth_code = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```  
*Note: You will need to add your own User model if your app will utilize numerous different verified QuickBooks accounts  

1. Get the Auth Client  

```
    auth_client = AuthClient(
      client_id=(settings.QUICKBOOKS_CLIENT_ID if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_CLIENT_ID),
      client_secret=(settings.QUICKBOOKS_CLIENT_SECRET if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_CLIENT_SECRET),
      environment=('sandbox' if settings.QUICKBOOKS_SANDBOX_MODE else 'production'),
      redirect_uri=(settings.QUICKBOOKS_REDIRECT_URI if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_REDIRECT_URI),
    )
```  

2:  

```
    # Get the token object from the database
    ...

    auth_client.refresh(refresh_token=token.refresh_token)
    token.access_token = auth_client.access_token
    token.refresh_token = auth_client.refresh_token
    token.save()
```  

<h2> How to: Query for Customers, Items, ... </h2>  

<h3> Customers </h3>  

By default, QuickBooks only allows the API to return up to 1000 records at a time. Due to this, we'll have to use pagination to retrieve more than that.  

You can use basic SQL queries to pull objects from the API.  

In the example below, we iterate through all Customer objects of the authorized QuickBooks account.  
```
    # Default limit on number of results that can be returned is 1000, so we set this as our max.
    limit = 1000
    # The offset is used to keep track of our position within the table we are pulling from.
    offset = 0
    # This list will store all customers as they are pulled.
    all_customers = []

    while True:
        # Retrieve customers with pagination parameters
        customers = Customer.query("SELECT * FROM Customer WHERE Active = True STARTPOSITION {} MAXRESULTS {}".format(offset, limit), qb=client)
        # Add the retrieved customers to the overall list
        all_customers.extend(customers)

        if len(customers) < limit:
            # Break the loop if the number of retrieved customers is less than the limit
            break

        offset += limit
    
    # For a full list of properties that the object has, see the python-quickbooks documentation at the top of this README.
    for customer in all_customers:
      print(customer.Id)
      print(customer.DisplayName)
      print(customer.CompanyName)
```  

<h3> Items </h3>  

In the example above, we pulled all Customers from the authorized QuickBooks account using pagination. If you want to pull all records from a table, use pagination in the same way.

```
    from quickbooks.objects.item import Item

    # Pagination
    ...
    
    # Query:
    items = Item.query("SELECT * FROM Item WHERE Active = True STARTPOSITION {} MAXRESULTS {}".format(offset, limit), qb=client)
    
    # Add your logic here for handling the pulled data.
    for item in items:
        print(item.Name)
```

You can follow the same basic formula for any different object type in the Python-Quickbooks library.  

<h2> How to: Save items to QuickBooks </h2>  

*Before beginning, you should note that there will be required fields for saving objects to QuickBooks. To find out what fields are required, reference the API docs at the top of this README.  

<h3> Create Customer - Singular </h3>  

```
from quickbooks.objects.customer import Customer

customer = Customer()
customer.CompanyName = "Test Company"
customer.save(qb=client)
```  

<h3> Create Customer - Batch </h3>  

**Alternatively**, you can batch_create records in QuickBooks. This operation is particularly useful in the case of doing an initial sync of data between two sources.

```
from quickbooks.batch import batch_create
from quickbooks.objects.customer import Customer

# Get customers from some quickbase table and store those results in the customers_response

new_customers = []

for customer in customers_response:
    new_customer = Customer()
    new_customer.DisplayName = customer['some_field']
    new_customer.CompanyName = customer['some_field']
    new_customers.append(new_customer)

results = batch_create(new_customers, qb=client)
```  

Handle the errors from `batch_create`:  

```
    results = batch_create(new_customers, qb=client)

    for fault in results.faults:
        print("Operation failed on " + fault.original_object.DisplayName)
        for error in fault.Error:
            print("Error " + error.Message)
```  

<h1> Production </h1>  

<h3>Getting Started</h3>  

1. Sign into the Intuit developer portal, select the `Dashboard` tab, then open your company.  

2. Navigate to the `Production Settings` on the left-hand side and obtain your "Keys & credentials" after completing a short survey about how your application will use QBO.  

3. After cloning your development environment to your server, create your virutal environment and install:  
`pip install django`  
`pip install intuit-oauth`  
`pip install python-quickbooks`  

4. Once you have all the necessary packages, go to `your_app/settings.py` and add your **PRODUCTION** credentials:  

```
# your_app/settings.py
# ...
# This is QBO's equivalent of DEBUG=True
QUICKBOOKS_SANDBOX_MODE = False

# Quickbooks sandbox credentials
PRODUCTION_QUICKBOOKS_CLIENT_ID = 'your_client_id'
PRODUCTION_QUICKBOOKS_CLIENT_SECRET = 'your_client_secret'
PRODUCTION_QUICKBOOKS_REDIRECT_URI = 'https://your-domain.com/qbo/callback'

# ...
```  
*Make sure you turn off QUICKBOOKS_SANDBOX_MODE, i.e. set it to "False"*  

From our previous development setup, we added checks within our views based on the QUICKBOOKS_SANDBOX_MODE. After setting it to false, our view will now utilize Production credentials.  

```
def authorize_quickbooks(request):
    auth_client = AuthClient(
        client_id=(settings.QUICKBOOKS_CLIENT_ID if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_CLIENT_ID),
        client_secret=(settings.QUICKBOOKS_CLIENT_SECRET if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_CLIENT_SECRET),
        environment=('sandbox' if settings.QUICKBOOKS_SANDBOX_MODE else 'production'),
        redirect_uri=(settings.QUICKBOOKS_REDIRECT_URI if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_REDIRECT_URI),
    )
    auth_url = auth_client.get_authorization_url([Scopes.ACCOUNTING])
    request.session['state'] = auth_client.state_token
    return redirect(auth_url)
```  

5. Add your necessary logic as a Celery task. For more information, see the reference below.  


<h2> How to: Add Celery </h2>  

Some tasks may take longer than the timeout on your server allots. For the aforementioned tasks, use Celery - a task manager which allows you to return a response to the client and continue processing data in the background.  

[My Celery Tutorial](https://github.com/sync-Matthew/Celery-Cheatsheet)  

Here's an example of what a Celery task for your QBO integration might look like:  

`your_app/qbo/tasks.py`:  

```
# Imports from celery and quickbooks-python
from celery import shared_task
from intuitlib.client import AuthClient
from intuitlib.enums import Scopes
from quickbooks import QuickBooks
from quickbooks.objects.customer import Customer
from quickbooks.objects.item import Item
from quickbooks.batch import batch_create
from django.conf import settings
from .models import QuickBooksToken

# For information on setting up a client with QuickBase - see SyncQB
from dpf360.client import get_client

# Helper method which takes a QuickBooksToken object and returns the actionable QuickBooks client object.
# For information on the QuickBooksToken model, see the development portion of this tutorial.
def client_helper(token):

    auth_client = AuthClient(
        client_id=(settings.QUICKBOOKS_CLIENT_ID if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_CLIENT_ID),
        client_secret=(settings.QUICKBOOKS_CLIENT_SECRET if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_CLIENT_SECRET),
        environment=('sandbox' if settings.QUICKBOOKS_SANDBOX_MODE else 'production'),
        redirect_uri=(settings.QUICKBOOKS_REDIRECT_URI if settings.QUICKBOOKS_SANDBOX_MODE else settings.PRODUCTION_QUICKBOOKS_REDIRECT_URI),
    )

    refresh_token = token.refresh_token
    company_id = token.realm_id

    try:
        client = QuickBooks(
            auth_client=auth_client,
            minorversion=62,
            environ=('sandbox' if settings.QUICKBOOKS_SANDBOX_MODE else 'production'),
            refresh_token=refresh_token,
            company_id=company_id,
        )
    except Exception as e:
        print(e)

    return client

# This method from a bird's-eye view does one thing:
# Syncs our Customers in QuickBooks and Quickbase.
# If a customer doesn't exist in QuickBooks but it exists in QuickBase, create that record in QuickBooks and update QuickBase with the ID of the created record.
# And vice versa.

@shared_task
def sync_customers_task():
    qb_client = get_client()
    
    # Get all the customers from QuickBase
    try:
        customers_response = qb_client.do_query(
            query="{3.GT.0}", # Select all customers
            database="your_table", # Customers
            columns=[your_fields], # RID, QuickBooks ID, DisplayName..
        )
        # data_a_set = set(item['14'] for item in customers_response)
    except Exception as e:
        print(e)
        
    # Retrieve all customers from QuickBooks using the most recent QuickBooksToken object credentials
    token = QuickBooksToken.objects.last()
    client = client_helper(token)
    limit = 1000
    offset = 0
    quickbooks_customers = []

    while True:
        # Retrieve customers with pagination parameters
        customers = Customer.query("SELECT * FROM Customer WHERE Active = True STARTPOSITION {} MAXRESULTS {}".format(offset, limit), qb=client)
        # Add the retrieved customers to the overall list
        quickbooks_customers.extend(customers)

        if len(customers) < limit:
            # Break the loop if the number of retrieved customers is less than the limit
            break

        offset += limit

    for customer in quickbooks_customers:
        # Customer from QuickBooks does not exist in QuickBase
        if (customer.DisplayName not in [item['14'] for item in customers_response]):
            print(customer.DisplayName)
            qb_client.add_record(fields={'14': customer.DisplayName, '12': customer.Id}, database='btacp5kk8')
        # if the customer exists check if it has a QuickBooks ID
        else:
            index = next((idx for idx, item in enumerate(customers_response) if item['14'] == customer.DisplayName), None)
            if index is not None:
                if customers_response[index]['12'] != customer.Id:
                    qb_client.edit_record(rid=customers_response[index]['3'], database='btacp5kk8', fields={'12': customer.Id})
    
    new_customers = []
    for customer in customers_response:
        # Customer from QuickBase does not exist in QuickBooks
        if (customer['14'] not in [item.DisplayName for item in quickbooks_customers]):
            # Create a new customer in QuickBooks
            new_customer = Customer()
            new_customer.DisplayName = customer['14']
            new_customer.CompanyName = customer['14']
            new_customers.append(new_customer)

    results = batch_create(new_customers, qb=client)

    for fault in results.faults:
        print("Operation failed on " + fault.original_object.DisplayName)
        for error in fault.Error:
            print("Error " + error.Message)
```  
