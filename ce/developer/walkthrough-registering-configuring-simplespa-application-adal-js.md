---
title: "Walkthrough: Registering and configuring SimpleSPA application with adal.js (Developer Guide for Dynamics 365 Customer Engagement)| MicrosoftDocs"
description: "This walkthrough describes the process of registering and configuring the simplest Single Page Application (SPA) to access data in Dynamics 365 Customer Engagement using adal.js and Cross-origin Resource Sharing (CORS)"
ms.custom: ""
ms.date: 10/31/2017
ms.reviewer: ""
ms.service: "crm-online"
ms.suite: ""
ms.tgt_pltfrm: ""
ms.topic: "article"
applies_to: 
  - "Dynamics 365 (online)"
ms.assetid: 0346a66d-147b-40e0-a1b6-0c30815043b4
caps.latest.revision: 14
author: "JimDaly"
ms.author: "jdaly"
search.audienceType: 
  - developer
search.app: 
  - D365CE
---
# Walkthrough: Registering and configuring SimpleSPA application with adal.js

[!INCLUDE[](../includes/cc_applies_to_update_9_0_0.md)]

This walkthrough describes the process of registering and configuring the simplest Single Page Application (SPA) to access data in [!INCLUDE[pn_dynamics_crm](../includes/pn-dynamics-crm.md)] Customer Engagement using adal.js and Cross-origin Resource Sharing (CORS). [!INCLUDE[proc_more_information](../includes/proc-more-information.md)] [Use OAuth with Cross-Origin Resource Sharing  to connect a Single Page Application  to Dynamics 365 (online)](oauth-cross-origin-resource-sharing-connect-single-page-application.md).  
  
## Prerequisites  
  
- [!INCLUDE[pn_dynamics_crm](../includes/pn-dynamics-crm.md)]  
  
- You must have a [!INCLUDE[pn_dynamics_crm_online](../includes/pn-dynamics-crm-online.md)] system user account with administrator role for the [!INCLUDE[pn_MS_Office_365](../includes/pn-ms-office-365.md)].  
  
- A [!INCLUDE[pn_Windows_Azure](../includes/pn-windows-azure.md)] subscription for application registration. A trial account will also work.  
  
- [!INCLUDE[pn_microsoft_visual_studio_2015](../includes/pn-microsoft-visual-studio-2015.md)]  
  
<a name="bkmk_goal"></a>   
## Goal of this walkthrough  
 When you complete this walkthrough you will be able to run a simple SPA application in Visual Studio that will provide the ability for a user to authenticate and retrieve data from [!INCLUDE[pn_dynamics_crm_online](../includes/pn-dynamics-crm-online.md)]. This application consists of a single HTML page.  
  
 When you debug the application initially there will only be a **Login** button.  
  
 Click **Login** and you will be re-directed to a sign-in page to enter your credentials.  
  
 After you enter your credentials you will be directed back to the HTML page where you will find the **Login** button is hidden and a **Logout** button and a **Get Accounts** button are visible. You will also see a greeting using information from your user account.  
  
 Click the **Get Accounts** button to retrieve 10 account records from your [!INCLUDE[pn_dynamics_crm](../includes/pn-dynamics-crm.md)] organization. The **Get Accounts** button is disabled as shown in the following screenshot:  
  
 ![The SimpleSPA page](media/simple-spa.png "The SimpleSPA page")  
  
> [!NOTE]
>  The initial load of data from [!INCLUDE[pn_dynamics_crm](../includes/pn-dynamics-crm.md)] may be slow as the operations to support authentication take place, but subsequent operations are much faster.  
  
 Finally, you can click the **Logout** button to logout.  
  
> [!NOTE]
>  This SPA application is not intended to represent a pattern for developing robust SPA applications. It is simplified to focus on the process of registering and configuring the application.  
  
### Create a web application project  
  
1. Using [!INCLUDE[pn_microsoft_visual_studio_2015](../includes/pn-microsoft-visual-studio-2015.md)], create a new **ASP.NET Web Application** project and use the **Empty** template. You can name the project whatever you like.  
  
    You should be able to use earlier versions of [!INCLUDE[pn_Visual_Studio](../includes/pn-visual-studio.md)] as well, but these steps will describe using [!INCLUDE[pn_visual_studio_2015](../includes/pn-visual-studio-2015.md)].  
  
2. Add a new HTML page named SimpleSPA.html to the project and paste in the following code:  
  
   ```html  
   <!DOCTYPE html>  
   <html>  
   <head>  
    <title>Simple SPA</title>  
    <meta charset="utf-8" />  
    <script src="https://secure.aadcdn.microsoftonline-p.com/lib/1.0.0/js/adal.min.js"></script>  
    <script type="text/javascript">  
     "use strict";  
  
     //Set these variables to match your environment  
     var organizationURI = "https:// [organization name].crm.dynamics.com"; //The URL to connect to CRM (online)  
     var tenant = "[xxx.onmicrosoft.com]"; //The name of the Azure AD organization you use  
     var clientId = "[client id]"; //The ClientId you got when you registered the application  
     var pageUrl = "http://localhost: [PORT #]/SimpleSPA.html"; //The URL of this page in your development environment when debugging.  
  
     var user, authContext, message, errorMessage, loginButton, logoutButton, getAccountsButton, accountsTable, accountsTableBody;  
  
     //Configuration data for AuthenticationContext  
     var endpoints = {  
      orgUri: organizationURI  
     };  
  
     window.config = {  
      tenant: tenant,  
      clientId: clientId,  
      postLogoutRedirectUri: pageUrl,  
      endpoints: endpoints,  
      cacheLocation: 'localStorage', // enable this for IE, as sessionStorage does not work for localhost.  
     };  
  
     document.onreadystatechange = function () {  
      if (document.readyState == "complete") {  
  
       //Set DOM elements referenced by scripts  
       message = document.getElementById("message");  
       errorMessage = document.getElementById("errorMessage");  
       loginButton = document.getElementById("login");  
       logoutButton = document.getElementById("logout");  
       getAccountsButton = document.getElementById("getAccounts");  
       accountsTable = document.getElementById("accountsTable");  
       accountsTableBody = document.getElementById("accountsTableBody");  
  
       //Event handlers on DOM elements  
       loginButton.addEventListener("click", login);  
       logoutButton.addEventListener("click", logout);  
       getAccountsButton.addEventListener("click", getAccounts);  
  
       //call authentication function  
       authenticate();  
  
       if (user) {  
        loginButton.style.display = "none";  
        logoutButton.style.display = "block";  
        getAccountsButton.style.display = "block";  
  
        var helloMessage = document.createElement("p");  
        helloMessage.textContent = "Hello " + user.profile.name;  
        message.appendChild(helloMessage)  
  
       }  
       else {  
        loginButton.style.display = "block";  
        logoutButton.style.display = "none";  
        getAccountsButton.style.display = "none";  
       }  
  
      }  
     }  
  
     // Function that manages authentication  
     function authenticate() {  
      //OAuth context  
      authContext = new AuthenticationContext(config);  
  
      // Check For & Handle Redirect From AAD After Login  
      var isCallback = authContext.isCallback(window.location.hash);  
      if (isCallback) {  
       authContext.handleWindowCallback();  
      }  
      var loginError = authContext.getLoginError();  
  
      if (isCallback && !loginError) {  
       window.location = authContext._getItem(authContext.CONSTANTS.STORAGE.LOGIN_REQUEST);  
      }  
      else {  
       errorMessage.textContent = loginError;  
      }  
      user = authContext.getCachedUser();  
  
     }  
  
     //function that logs in the user  
     function login() {  
      authContext.login();  
     }  
     //function that logs out the user  
     function logout() {  
      authContext.logOut();  
      accountsTable.style.display = "none";  
      accountsTableBody.innerHTML = "";  
     }  
  
   //function that initiates retrieval of accounts  
     function getAccounts() {  
  
      getAccountsButton.disabled = true;  
      var retrievingAccountsMessage = document.createElement("p");  
      retrievingAccountsMessage.textContent = "Retrieving 10 accounts from " + organizationURI + "/api/data/v8.0/accounts";  
      message.appendChild(retrievingAccountsMessage)  
  
      // Function to perform operation is passed as a parameter to the aquireToken method  
      authContext.acquireToken(organizationURI, retrieveAccounts)  
  
     }  
  
   //Function that actually retrieves the accounts  
     function retrieveAccounts(error, token) {  
      // Handle ADAL Errors.  
      if (error || !token) {  
       errorMessage.textContent = 'ADAL error occurred: ' + error;  
       return;  
      }  
  
      var req = new XMLHttpRequest()  
      req.open("GET", encodeURI(organizationURI + "/api/data/v8.0/accounts?$select=name,address1_city&$top=10"), true);  
      //Set Bearer token  
      req.setRequestHeader("Authorization", "Bearer " + token);  
      req.setRequestHeader("Accept", "application/json");  
      req.setRequestHeader("Content-Type", "application/json; charset=utf-8");  
      req.setRequestHeader("OData-MaxVersion", "4.0");  
      req.setRequestHeader("OData-Version", "4.0");  
      req.onreadystatechange = function () {  
       if (this.readyState == 4 /* complete */) {  
        req.onreadystatechange = null;  
        if (this.status == 200) {  
         var accounts = JSON.parse(this.response).value;  
         renderAccounts(accounts);  
        }  
        else {  
         var error = JSON.parse(this.response).error;  
         console.log(error.message);  
         errorMessage.textContent = error.message;  
        }  
       }  
      };  
      req.send();  
     }  
     //Function that writes account data to the accountsTable  
     function renderAccounts(accounts) {  
      accounts.forEach(function (account) {  
       var name = account.name;  
       var city = account.address1_city;  
       var nameCell = document.createElement("td");  
       nameCell.textContent = name;  
       var cityCell = document.createElement("td");  
       cityCell.textContent = city;  
       var row = document.createElement("tr");  
       row.appendChild(nameCell);  
       row.appendChild(cityCell);  
       accountsTableBody.appendChild(row);  
      });  
      accountsTable.style.display = "block";  
     }  
  
    </script>  
    <style>  
     body {  
      font-family: 'Segoe UI';  
     }  
  
     table {  
      border-collapse: collapse;  
     }  
  
     td, th {  
      border: 1px solid black;  
     }  
  
     #errorMessage {  
      color: red;  
     }  
  
     #message {  
      color: green;  
     }  
    </style>  
   </head>  
   <body>  
    <button id="login">Login</button>  
    <button id="logout" style="display:none;">Logout</button>  
    <button id="getAccounts" style="display:none;">Get Accounts</button>  
    <div id="errorMessage"></div>  
    <div id="message"></div>  
    <table id="accountsTable" style="display:none;">  
     <thead><tr><th>Name</th><th>City</th></tr></thead>  
     <tbody id="accountsTableBody"></tbody>  
    </table>  
   </body>  
   </html>  
  
   ```  
  
3. Set this page as the start page for the project  
  
4. In the properties of the project, select **Web** and under **Servers** note the **Project URL**. It should be something like `http://localhost:46575/`. Note the port number that is generated. You will need this in the next step.  
  
5. Within the SimpleSPA.html page, locate the following configuration variables and set them accordingly. You will be able to set the `clientId` after you complete the next part of the walkthrough.  
  
   ```javascript  
   //Set these variables to match your environment  
   var organizationURI = "https://[organization name].crm.dynamics.com"; //The URL to connect to CRM (online)  
   var tenant = "[xxx.onmicrosoft.com]"; //The name of the Azure AD organization you use  
   var clientId = "[client id]"; //The ClientId you got when you registered the application  
   var pageUrl = "http://localhost:[PORT #]/SimpleSPA.html"; //The URL of this page in your development environment when debugging.  
  
   ```  
  
### Register the application  
  
1. [Sign in](http://manage.windowsazure.com) to the [!INCLUDE[pn_Windows_Azure](../includes/pn-windows-azure.md)] management portal by using an account with administrator permission. You must use an account in the same [!INCLUDE[pn_Office_365](../includes/pn-office-365.md)] subscription (tenant) as you intend to register the app with. You can also access the [!INCLUDE[pn_Windows_Azure](../includes/pn-windows-azure.md)] portal through the [!INCLUDE[pn_Office_365](../includes/pn-office-365.md)] admin center by expanding the **ADMIN** item in the left navigation pane and selecting **Azure AD**.  
  
    If you don’t have an [!INCLUDE[pn_azure_shortest](../includes/pn-azure-shortest.md)] tenant (account) or you do have one but your [!INCLUDE[pn_Office_365](../includes/pn-office-365.md)] subscription with [!INCLUDE[pn_dynamics_crm_online](../includes/pn-dynamics-crm-online.md)] is not available in your [!INCLUDE[pn_azure_shortest](../includes/pn-azure-shortest.md)] subscription, following the instructions in the topic [Set up Azure Active Directory access for your Developer Site](https://msdn.microsoft.com/office/office365/HowTo/setup-development-environment) to associate the two accounts.  
  
    If you don’t have an account, you can sign up for one by using a credit card. However, the account is free for application registration and your credit card won’t be charged if you only follow the procedures called out in this topic to register one or more apps. [!INCLUDE[proc_more_information](../includes/proc-more-information.md)] [Active Directory Pricing Details](http://azure.microsoft.com/pricing/details/active-directory/)  
  
2. Click **Active Directory** in the left column of the page. You may need to scroll the left column to see the **Active Directory** icon and label.  
  
3. Click the desired tenant directory in the directory list.  
  
   ![List of available Active Directory entries](media/azure-active-directory.PNG "List of available Active Directory entries")  
  
    If your [!INCLUDE[pn_crm_shortest](../includes/pn-crm-shortest.md)] tenant directory isn’t shown in the directory list, click **Add**, and then select **Use existing directory** in the dialog box. Follow the prompts and instructions provided, and then go back to step 1.  
  
4. With the target directory selected, click **Applications** (near the top of the page), and then click **Add**.  
  
5. In the **What do you want to do?** dialog box, click **Add an application my organization is developing**.  
  
6. When prompted, enter a name for your application, for example ‘SimpleSPA’, pick the type: **Web Application and/or Web API**, and then click the right arrow to continue. Click a question mark **?** for more information on the appropriate values for each input field.  
  
7. Enter the following information :  
  
   **Sign-on URL**  
    This is the URL which the user should be redirected to after they sign in. For debugging purposes in [!INCLUDE[pn_Visual_Studio_short](../includes/pn-visual-studio-short.md)] it should be  `http://localhost:####/SimpleSPA.html` where #### represents the port number you got from step 4 of the **Create a web application project** procedure.  
  
   **APP ID URI**  
    This must be a unique identifier for the application. Use `https://XXXX.onmicrosoft.com/SimpleSPA` where XXXX is the Active Directory tenant.  
  
8. With the tab of the newly registered app selected, click **Configure**, locate the **Client id** and copy it.  
  
    Set the `clientId` variable in the SimpleSPA.html page to this value. Refer to step 5 of the **Create a web application project** procedure.  
  
9. Scroll to the bottom of the page and click **Add application**. In the dialog box select **Dynamics 365 Online** and close the dialog box.  
  
10. Under permissions to other applications, you will find a row for **Dynamics 365 Online** and **Delegated Permissions: 0**. Select this and add **Access Dynamics 365 (online) as organization users**.  
  
11. Save the application registration  
  
12. At the bottom, select **Manage Manifest** and choose **Download Manifest**.  
  
13. Open the JSON file you downloaded and locate the line: `"oauth2AllowImplicitFlow": false,` and change `false` to `true` and save the file.  
  
14. Go back to **Manage Manifest** again. Choose **Upload Manifest** and upload the JSON file you just saved.  
  
### Debugging the application  
  
1. Set the browser to use [!INCLUDE[pn_microsoft_edge](../includes/pn-microsoft-edge.md)], [!INCLUDE[tn_Google_Chrome](../includes/tn-google-chrome.md)], or [!INCLUDE[tn_Mozilla_Firefox](../includes/tn-mozilla-firefox.md)].  
  
   > [!NOTE]
   > [!INCLUDE[pn_Internet_Explorer](../includes/pn-internet-explorer.md)] will not work for debugging in this situation.  
  
2. Press F5 to start debugging. You should expect the behavior described in [Goal of this walkthrough](walkthrough-registering-configuring-simplespa-application-adal-js.md#bkmk_goal).  
  
    If you don’t get the results you expect, double-check the values you set when registering the application and configuring the SimpleSPA.html code.  
  
### See also  
 [Use OAuth with Cross-Origin Resource Sharing  to connect a Single Page Application  to Dynamics 365 (online)](oauth-cross-origin-resource-sharing-connect-single-page-application.md)
