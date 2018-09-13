
# Build a .NET CLI app with Microsoft Graph

In this lab you will create a .NET application, configured with Azure Active Directory (Azure AD) for authentication and authorization using Microsoft Authentication Library (MSAL).

## In this lab
- [Exercise 1: Create a CLI application](#create-cli-application)
- [Exercise 2: Register a native application with the Application Registration Portal](#exercise-2-register-a-native-application-with-the-application-registration-portal)
- [Exercise 3: Extend the app for Azure AD Authentication](#exercise-3-extend-the-app-for-azure-ad-authentication)
- [Exercise 4: Extend the app for Microsoft Graph](#exercise-4-extend-the-app-for-microsoft-graph)

## Prerequisites

To complete this lab, you need the following:

* [Visual Studio](https://visualstudio.microsoft.com/vs/) installed on your development machine.(**Note**: This tutorial was written with Visual Studio 2017 version 15.8.3. 
The steps in this guide may work with other versions, but that has not been tested).

If you don't have a Microsoft account, there are a couple of options to get a free account:
* You can [sign up for a new personal Microsoft account](https://signup.live.com/signup?wa=wsignin1.0&rpsnv=12&ct=1454618383&rver=6.4.6456.0&wp=MBI_SSL_SHARED&wreply=https://mail.live.com/default.aspx&id=64855&cbcxt=mai&bk=1454618383&uiflavor=web&uaid=b213a65b4fdc484382b6622b3ecaa547&mkt=E-US&lc=1033&lic=1)
* You can [sign up for the Office 365 Developer Program](https://developer.microsoft.com/office/dev-program) to get  a free Office 365 subscription

## Exercise 1: Create a .NET application

Open Visual Studio, and select **File > New > Project.** In the **New Project** dialog, do the following:
1. Select **Visual C# > Get Started**.
2. Select **Console App**.
3. Enter **Calendar** for the name of the project.


#### Important

Ensure that you enter the exact name for the Visual Studio Project that is specified in these lab instructions. The project name will also be the **namespace**. If you used a different project name
adjust all the namespaces to match your project name.

Press **Ctrl + F5** to run the application or **Debug > Start Debugging** to run the application in debug mode. If everything is working fine a terminal window will open.

Before moving on, install additional NuGet packages that you will use later.

Select **Tools > Nuget Package Manager > Package Manager Console**. In the console window, run the following commands:
```powershell
Install-Package "Microsoft Graph"
Install-Package "Microsoft.Identity.Client" -pre
Install-Package "System.Configuration.ConfigurationManager"
```

## Exercise 2: Register a native application with the Application Registration Portal
In this exercise, you will create a new Azure AD native application using the Application Registry Portal.
1. Open a browser and navigate to the [Application Registration Portal](). Login using a **personal account**(aka: Microsoft Account) or **Work or School Account**
2. Select **Add an app** at the top of the page
3. On the Register your application page, set the **Applcaiton Name** to **Calendar** and select **Create**
4. On the **Calendar Registration** page, under the **Properties** section, copy the **Application Id** as you will need it later
5. Select **Add Platform** button on the registration page. Choose **Native Application** in the dialog box.
6. Once completed, move to the bottom of the page and select **Save**

## Exercise 3: Extend the app for Azure AD Authentication
In this exercise you will extend the application from **Exercise 1** to support authentication with Azure AD. This is required to obtain the necessary OAuth token
to call the Microsoft Graph.

Edit the **app.config** file, and immeadiately before the `/configuration` element, add the following element:
```xml
<appSettings>
    <add key="clientId" value="THE_APPLICATION_ID_YOU_COPIED_IN_EXERCISE_2">
</appSettings>
```

## Add GraphClientServiceProvider.cs
1. Add a class to the project named **GraphClientServiceProvider** This class will be responsible for authenticating using the Microsoft Authentication Library (MSAL), which is the
Microsoft.Identity.Client package that we installed. For separation of concerns, change the **namespace** of this class to **Helpers**

2. Replace the `using` statement at the top of the file
```csharp
using Microsoft.Graph;
using Microsoft.Identity.Client;
using System;
using System.Collections.Generic;
using System.Configuration;
using System.Diagnostics;
using System.Net.Http.Headers;
using System.Threading.Tasks;
```

3. Replace the `class` declaration with the following
```csharp
class GraphClientServiceProvider
    {
        // The client ID is used by the application to uniquely identify itself to the authentication endpoint.
        private static string clientId = ConfigurationManager.AppSettings["clientId"].ToString();
        private static string[] scopes = {
            "https://graph.microsoft.com/User.Read"
        };

        private static PublicClientApplication identityClientApp = new PublicClientApplication(clientId);
        private static GraphServiceClient graphClient = null;

        // Get an access token for the given context and resourceId. An attempt is first made to acquire the token silently.
        // If that fails, then we try to acquire the token by prompting the user.
        public static GraphServiceClient GetAuthenticatedClient()
        {
            if (graphClient == null)
            {
                try
                {
                    graphClient = new GraphServiceClient(
                        "https://graph.microsoft.com/v1.0",
                        new DelegateAuthenticationProvider(
                                async (requestMessage) =>
                                {
                                    var token = await getTokenForUserAsync();
                                    requestMessage.Headers.Authorization = new AuthenticationHeaderValue("bearer", token);
                                }
                            ));
                    return graphClient;
                } 
                catch(Exception error)
                {
                    Debug.WriteLine($"Could not create a graph client {error.Message}");
                }
            }
            return graphClient;
        }

        /// <summary>
        /// Get token for User
        /// </summary>
        /// <returns>Token for User</returns>
        private static async Task<string> getTokenForUserAsync()
        {
            AuthenticationResult authResult = null;

            try
            {
                IEnumerable<IAccount> account = await identityClientApp.GetAccountsAsync();
                authResult = await identityClientApp.AcquireTokenSilentAsync(scopes, account as IAccount);
                return authResult.AccessToken;
            }
            catch(MsalUiRequiredException error)
            {
                // This means the AcquireTokenSilentAsync threw an exception. 
                // This prompts the user to log in with their account so that we can get the token.
                authResult = await identityClientApp.AcquireTokenAsync(scopes);
                return authResult.AccessToken;
            }
        }

    }
```

## Exercise 4: Extend the app for Microsoft Graph
In this exercise you will incorporate the Microsoft Graph into the application. For this application, you will use the [Microsoft Graph Client Library for .NET](https://github.com/microsoftgraph/msgraph-sdk-dotnet) to make calls to Microsoft Graph.
Add the following code snippet below **Main** in `Program.cs`

```csharp
        /// <summary>
        /// Gets a User from Microsoft Graph
        /// </summary>
        /// <returns>A User object</returns>
        public static async Task<User> GetMeAsync()
        {
            User currentUser = null;
            try
            {
                var graphClient = Authentication.GetAuthenticatedClient();

                // Request to get the current logged in user object from Microsoft Graph
                currentUser = await graphClient.Me.Request().GetAsync();

                return currentUser;
            }

            catch (ServiceException e)
            {
                Debug.WriteLine("We could not get the current user: " + e.Error.Message);
                return null;
            }
        }

        static async Task RunAsync()
        {
            var me = await GetMeAsync();

            Console.WriteLine($"{me.DisplayName} logged in.");
            Console.WriteLine();
        }
```

Replace the **Main** function with the following
```csharp
        static void Main(string[] args)
        {
            Console.WriteLine("Welcome to the Calendar CLI");

            RunAsync().GetAwaiter().GetResult();
            Console.ReadKey();
        }
```

Press **Ctrl + F5** to run the application or **Debug > Start Debugging** to run the application in debug mode. If everything is working fine a terminal window will show and prompt you to log in.
