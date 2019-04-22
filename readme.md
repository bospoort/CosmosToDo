# Azure Monitoring Challenge

This set of challenges will provide you an overview of most of the major features of Azure Monitor with a focus on Application Insights. 


## Challenge 0: Setup

1. Launch the Azure Cloud Shell (Hint: https://shell.azure.com) in the bash environment. This will provide you a bash console with the latest version of the Azure CLI installed and logged into your Azure Subscription. (Note: you can do this from your local machine if you have the latest version of the CLI installed locally.)

2. Ensure that you are logged into the correct subscription

``` bash
az account list -o table
```

If you are not logged into the right subscription you can change your subscription with the following command

``` bash
az account set -s <SubscriptionId>
```

3. Setup some variables to deploy our sample environment

``` bash
# 5 to 8 character prefix to identify you (all lowercase, no special characters)
prefix=<your prefix>

# setup some more variables
gitrepo=https://github.com/shawnweisfeld/CosmosToDo
webappname=$prefix$RANDOM
rg=$prefix-monitoringlab
location='eastus'
accountName=$prefix$RANDOM
```

4. Create our resource group and deploy our web app and cosmos db

``` bash
# Create a resource group.
az group create --location $location --name $rg

# Create an App Service plan
az appservice plan create --name $webappname --resource-group $rg --sku S1

# Create a web app.
az webapp create --name $webappname --resource-group $rg --plan $webappname

# Deploy sample code to our webapp
az webapp deployment source config --name $webappname --resource-group $rg --repo-url $gitrepo --branch master

# Create a SQL API Cosmos DB account
az cosmosdb create --resource-group $rg --name $accountName --default-consistency-level "Session"

## Get the endpoint and key of the cosmos db that we just created
endpoint=`az cosmosdb show -n $accountName -g $rg --query 'documentEndpoint' --output tsv`
primaryKey=`az cosmosdb list-keys -n $accountName -g $rg --query 'primaryMasterKey' --output tsv`

# connect our webapp to our cosmosdb
az webapp config appsettings set -g $rg -n $webappname --settings endpoint=$endpoint primaryKey=$primaryKey

```

## Challenge 1: Application Insights: Deploy and Overview

1. Using the Azure portal create a new instance of Application Insights in the resource group that we created earlier.

2. After it is created get the Instrumentation Key for our instance of Application Insights from the portal.

3. Use the following command to update the web application with our instrumentation key.

``` bash
az webapp config appsettings set -g $rg -n $webappname --settings InstrumentationKey=<your key>
```

4. Navigate to your deployed website (i.e. https://yoursite.azurewebsites.net), create 10 todo items, then go back in and edit some and mark some completed.

5. Create 5 errors in your site by going to the "error" controller (i.e. https://yoursite.azurewebsites.net/Error). This page throws an "not implemented" exception. 

6. Using the Azure Portal create an Application Insights "application dashboard". This will provide us a bunch of default metrics to keep an eye on our application. 

7. Using the "application dashboard" you just created, drill into the failures widget, and the details for the request that generated the 500 error. What information can you find out about the cause of the error?

8. Using the "View in Analytics" button pull up the Kusto query behind the "Requests" chart. Review the query. 

9. Go back to the "application dashboard" and look at the "Application Map". What components do you see? What was the slowest call from our website to cosmos db?

## Challenge 2: Application Insights Availability Test
1. Using Application Insights, create a URL Ping Test to check on the availability of our site from 5 locations around the US every 5 minutes. The url ping test should hit our availability test controller (i.e. https://yoursite.azurewebsites.net/AvailabilityTest). 
2. Review the code for the availability controller ([here](https://github.com/shawnweisfeld/CosmosToDo/blob/master/src/Controllers/AvailabilityTestController.cs)). List some of the dependencies from the applications you work on.
3. Do you see the failed availability tests on the "application dashboard"?

## Challenge 3: Application Insights Live Metrics Stream
1. Lets start by configuring our website to perform a "scale out" auto scale to up to 3 nodes, when the cpu hits 20%. NOTE: this is really low to ensure we can generate enough load to make the service scale out.
2. Next lets create a VM to generate some load on our web server, and installing the load test tool, you can do this from the Azure Cloud Shell

``` bash
# 5 to 8 character prefix to identify you (all lowercase, no special characters)
# this should match the one from earlier
prefix=<your prefix>

# setup some more variables
vmname=$prefix$RANDOM
rg=$prefix-monitoringlab

# create a VM, we will use this to generate artifical traffic for our website 
az vm create -g $rg --name $vmname --image UbuntuLTS --admin-username vmadmin --generate-ssh-keys

# SSH into the VM
ssh vmadmin@<public IP Address of your vm>

# update the vm and install the apache utilities
sudo apt-get -y update
sudo apt-get install apache2-utils
```

2. Open the Live metrics stream from your "application dashboard", how many web servers are currently deployed? Leave this tab open. 

3. Use the following command from Generate Load on your VM

``` bash
ab -n 10000 -c 10 https://<webappname>.azurewebsites.net/Load
```

4. Flip back to the tab you have open with the live metrics, do you see the spike in requests and CPU load? How many web servers do you have now?

## Challenge 4: Application Insights: Alerts with Kusto

The Analytics tool in Application Insights allows you to write your own Kusto queries ([Hint](https://docs.microsoft.com/en-us/azure/azure-monitor/log-query/query-language)) over the data that is being collected. Use this tool and write the following:

1. Draw a line chart showing the count of requests over the last 3 hours, grouped into 15 minute buckets.
2. Draw a bar chart showing the number of requests over the last 3 hours, grouped by performanceBucket.
3. What are the top 10 pages that failed on our site in the past 3 hours, how many times did each fail?


## Challenge 5: Application Insights: Workbooks

More Info on Workbooks can be found [here](https://docs.microsoft.com/en-us/azure/azure-monitor/app/usage-workbooks)

1. Create a few workbooks using the templates in the gallery
2. Create a new or modify an existing workbook to add a new chart you created from one of the kusto queries you wrote.
3. Pin your workbook to your "application dashboard".