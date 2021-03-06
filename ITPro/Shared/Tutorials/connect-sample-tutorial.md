<properties linkid="dev-net-e2e-multi-tier" urldisplayname="Multi-Tier Application" headerexpose pagetitle=".NET Multi-Tier Application" metakeywords="" footerexpose metadescription="" umbraconavihide="0" disquscomments="1"></properties>

# .NET Multi-Tier Application Using Service Bus Queues

Developing for Windows Azure is easy using Visual Studio 2010 and the
free Windows Azure SDK for .NET. If you do not already have Visual
Studio 2010, the SDK will automatically install Visual Web Developer
2010 Express, so you can start developing for Windows Azure entirely for
free. This guide assumes you have no prior experience using Windows
Azure. On completing this guide, you will have an application that uses
multiple Windows Azure resources running in your local environment and
demonstrating how a multi-tier application works.

You will learn:

-   How to enable your computer for Windows Azure development with a
    single download and install.
-   How to use Visual Studio to develop for Windows Azure.
-   How to create a multi-tier application in Windows Azure using web
    and worker roles.
-   How to communicate between tiers using Service Bus Queues.

You will build a front-end ASP.NET MVC web role that uses a back-end
worker role to process long running jobs. You will learn how to create
multi-role solutions, as well as how to use Service Bus Queues to enable
inter-role communication. A screenshot of the completed application is
shown below:

![][0]

## Scenario Overview: Inter-Role Communication

To submit an order for processing, the front end UI component, running
in the web role, needs to interact with the middle tier logic running in
the worker role. This example uses Service Bus brokered messaging for
the communication between the tiers.

Using brokered messaging between the web and middle tiers decouples the
two components. In contrast to direct messaging (that is, TCP or HTTP),
the web tier does not connect to the middle tier directly; instead it
pushes units of work, as messages, into the Service Bus, which reliably
retains them until the middle tier is ready to consume and process them.

The Service Bus provides two entities to support brokered messaging,
queues and topics. With queues, each message sent to the queue is
consumed by a single receiver. Topics support the publish/subscribe
pattern in which each published message is made available to each
subscription registered with the topic. Each subscription logically
maintains its own queue of messages. Subscriptions can also be
configured with filter rules that restrict the set of messages passed to
the subscription queue to those that match the filter. This example uses
Service Bus queues.

![][1]

This communication mechanism has several advantages over direct
messaging, namely:

-   **Temporal decoupling.** With the asynchronous messaging pattern,
    producers and consumers need not be online at the same time. Service
    Busreliably stores messages until the consuming party is ready to
    receive them. This allows the components of the distributed
    application to be disconnected, either voluntarily, for example, for
    maintenance, or due to a component crash, without impacting the
    system as a whole. Furthermore, the consuming application may only
    need to come online during certain times of the day.

-   **Load leveling**. In many applications, system load varies over
    time whereas the processing time required for each unit of work is
    typically constant. Intermediating message producers and consumers
    with a queue means that the consuming application (the worker) only
    needs to be provisioned to accommodate average load rather than peak
    load. The depth of the queue will grow and contract as the incoming
    load varies. This directly saves money in terms of the amount of
    infrastructure required to service the application load.

-   **Load balancing.** As load increases, more worker processes can be
    added to read from the queue. Each message is processed by only one
    of the worker processes. Furthermore, this pull-based load balancing
    allows for optimum utilization of the worker machines even if the
    worker machines differ in terms of processing power as they will
    pull messages at their own maximum rate. This pattern is often
    termed the competing consumer pattern.

    ![][2]

The following sections discuss the code that implements this
architecture.

## Set Up the Development Environment

Before you can begin developing your Windows Azure application, you need
to get the tools and set-up your development environment.

1.  To install the Windows Azure SDK for .NET, click the button below:

    [Get Tools and SDK][]

    When prompted to run or save WindowsAzureSDKForNet.exe, click
    **Run**:

    ![][3]

2.  Click **Install** in the installer window and proceed with the
    installation:

    ![][4]

3.  Once the installation is complete, you will have everything
    necessary to start developing. The SDK includes tools that let you
    easily develop Windows Azure applications in Visual Studio. If you
    do not have Visual Studio installed, it also installs the free
    Visual Web Developer Express.

## Create a Windows Azure Account

1.  Open a web browser, and browse to [http://www.windowsazure.com][].

    To get started with a free account, click **free trial** in the upper
    right corner and follow the steps.

    ![][5]

2.  Your account is now created. You are ready to deploy your
    application to Windows Azure!

## <a name="create-namespace"> </a>Set up the Service Bus Namespace

The next step is to create a service namespace, and to obtain a shared
secret key. A service namespace provides an application boundary for
each application exposed through Service Bus. A shared secret key is
automatically generated by the system when a service namespace is
created. The combination of service namespace and shared secret key
provides a credential for Service Bus to authenticate access to an
application.

1.  Log into the [Windows Azure Management Portal][].

2.  In the lower left navigation pane of the Management Portal, click
    **Service Bus, Access Control & Caching**.

3.  In the upper left pane of the Management Portal, click the **Service
    Bus** node, then click **New**.

    ![][6]

4.  In the **Create a new Service Namespace** dialog, enter a namespace,
    and then to make sure that it is unique, click **Check
    Availability**.   
    ![][7]

5.  After making sure the namespace name is available, choose the
    country or region in which your namespace should be hosted (make
    sure you use the same country/region in which you are deploying your
    compute resources), and then click **Create Namespace**. Also,
    choose a country/region from the dropdown, a connection pack size,
    and the name of the subscription you want to use:

    IMPORTANT: Pick the **same region** that you intend to choose for
    deploying your application. This will give you the best performance.

6.  Click **Create Namespace**. The system now creates your service
    namespace and enables it. You might have to wait several minutes as
    the system provisions resources for your account.

7.  In the main window, click the name of your service namespace.

8.  In the **Properties** pane on the right-hand side, find the
    **Default Key** entry.

9.  In Default Key, click **View**. Make a note of the key, or copy it
    to the clipboard.

## Create a Web Role

In this section, you will build the front end of your application. You
will first create the various pages that your application displays.
After that, you will add the code for submitting items to a Service Bus
Queue and displaying status information about the queue.

### Create the Project

1.  Using administrator privileges, start either Microsoft Visual Studio
    2010 or Microsoft Visual Web Developer Express 2010. To start Visual
    Studio with administrator privileges, right-click **Microsoft Visual
    Studio 2010 (or Microsoft Visual Web Developer Express 2010)** and
    then click Run as administrator. The Windows Azure compute emulator,
    discussed later in this guide, requires that Visual Studio be
    launched with administrator privileges.

    In Visual Studio, on the **File** menu, click **New**, and then
    click **Project**.

    ![][8]

2.  From **Installed Templates**, under **Visual C\#**, click Cloud and
    then click **Windows Azure Project**. Name the project
    **MultiTierApp**. Then click **OK**.

    ![][9]

3.  From **.NET Framework 4** roles, double-click **ASP.NET MVC 3 Web
    Role**.

    ![][10]

4.  Hover over **MvcWebRole1** under **Windows Azure solution**, click
    the pencil icon, and rename the web role to **FrontendWebRole**. Then Click **OK**.

    ![][11]

5.  From the **Select a template** list, click **Internet Application**,
    then click **OK**.

    ![][12]

6.  In **Solution Explorer**, right-click **References**, then click
    **Manage NuGet Packages...** or **Add Library Package Reference**.

7.  Select **Online** on the left-hand side of the dialog. Search for
    ‘**WindowsAzure.ServiceBus**’ and select the **Windows Azure.Service
    Bus** item. Then complete the installation and close this dialog.

    ![][13]

8.  Note that the required client assemblies are now referenced and some
    new code files have been added.

9.  In **Solution Explorer**, right click **Models** and click **Add**,
    then click **Class**. In the Name box, type the name
    **OnlineOrder.cs**. Then click **Add**.

### Write the Web Code for Your Web Role

In this section, you will create the various pages that your application
displays.

1.  In the **OnlineOrder.cs** file in Visual Studio, replace the
    existing namespace definition with the following code:

        namespace FrontendWebRole.Models
        {
            public class OnlineOrder
            {
                public string Customer { get; set; }
                public string Product { get; set; }
            }
        }

2.  In the **Solution Explorer**, double-click
    **Controllers\\HomeController.cs**. Add the following **using**
    statements at the top of the file to include the namespaces for the
    model you just created, as well as Service Bus:

        using FrontendWebRole.Models;
        using Microsoft.ServiceBus.Messaging;
        using Microsoft.ServiceBus;

3.  Also in the **HomeController.cs** file in Visual Studio, replace the
    existing namespace definition with the following code. This code
    contains methods for handling the submission of items to the queue:

        namespace FrontendWebRole.Controllers
        {
            public class HomeController : Controller
            {
                public ActionResult Index()
                {
                    // Simply redirect to Submit, since Submit will serve as the
                    // front page of this application
                    return RedirectToAction("Submit");
                }

                public ActionResult About()
                {
                    return View();
                }

                // GET: /Home/Submit
                // Controller method for a view you will create for the submission
                // form
                public ActionResult Submit()
                {
                    // Will put code for displaying queue message count here.

                    return View();
                }

                // POST: /Home/Submit
                // Controler method for handling submissions from the submission
                // form 
                [HttpPost]
                public ActionResult Submit(OnlineOrder order)
                {
                    if (ModelState.IsValid)
                    {
                        // Will put code for submitting to queue here.
                    
                        return RedirectToAction("Submit");
                    }
                    else
                    {
                        return View(order);
                    }
                }
            }
        }

4.  Build the application (F6).

5.  Now, you will create the view for the **Submit()** method you
    created above. Right-click within the Submit() method, and choose
    **Add View**

    ![][14]

6.  A dialog appears for creating the view. Click the checkbox for
    **Create a strongly-typed view**. In addition, select the
    **OnlineOrder** class in the **Model class** dropdown, and choose
    **Create** under the **Scaffold template** dropdown.

    ![][15]

7.  Click **Add**.

8.  Now, you will change the displayed name of your application. In the
    **Solution Explorer**, double-click the
    **Views\\Shared\\_Layout.cshtml**file to open it in the Visual
    Studio editor.

9.  Locate **My MVC Application**, and replace it with
    **LITWARE'S Awesome Products**:

    ![][16]

10. Finally, tweak the submission page to include some information about
    the queue. In the **Solution Explorer**, double-click the
    **Views\\Home\\Submit.cshtml** file to open it in the Visual Studio
    editor. Add the following line after `<h2>Submit</h2>`. For now,
    the **ViewBag.MessageCount** is empty. You will populate it later.

        <p>Current Number of Orders in Queue Waiting to be Processed: @ViewBag.MessageCount</p>
             

11. You now have implemented your UI. You can press **F5** to run your
    application and confirm it looks as expected.

    ![][17]

### Write the Code for Submitting Items to a Service Bus Queue

Now, you will add code for submitting items to a queue. You will first
create a class that contains your Service Bus Queue connection
information. Then, you will initialize your connection from
**Global.aspx.cs**. Finally, you will update the submission code you
created earlier in **HomeController.cs** to actually submit items to a
Service Bus Queue.

1.  In Solution Explorer, right-click **FrontendWebRole** (right-click the class, not the role). Click **Add**, and then click **Class**.

2.  Name the class **QueueConnector.cs**. Click **Add** to create the class.

3.  You will now paste in code that encapsulates your connection
    information and contains methods for initializing the connection to
    a Service Bus Queue. In QueueConnector.cs, paste in the following code, and enter in
    values for **Namespace**, **IssuerName**, and **IssuerKey**. You can
    find these values in the [Management Portal][Windows Azure Management Portal].

        using System;
        using System.Collections.Generic;
        using System.Linq;
        using System.Web;
        using Microsoft.ServiceBus.Messaging;
        using Microsoft.ServiceBus;

        namespace FrontendWebRole
        {
            public static class QueueConnector
            {
                // Thread-safe. Recommended that you cache rather than recreating it
                // on every request.
                public static QueueClient OrdersQueueClient;

                // Obtain these values from the Management Portal
                public const string Namespace = "your service bus namespace";
                public const string IssuerName = "issuer name";
                public const string IssuerKey = "issuer key";

                // The name of your queue
                public const string QueueName = "OrdersQueue";

                public static NamespaceManager CreateNamespaceManager()
                {
                    // Create the namespace manager which gives you access to
                    // management operations
                    var uri = ServiceBusEnvironment.CreateServiceUri(
                        "sb", Namespace, String.Empty);
                    var tP = TokenProvider.CreateSharedSecretTokenProvider(
                        IssuerName, IssuerKey);
                    return new NamespaceManager(uri, tP);
                }

                public static void Initialize()
                {
                    // Using Http to be friendly with outbound firewalls
                    ServiceBusEnvironment.SystemConnectivity.Mode = 
                        ConnectivityMode.Http;

                    // Create the namespace manager which gives you access to 
                    // management operations
                    var namespaceManager = CreateNamespaceManager();

                    // Create the queue if it does not exist already
                    if (!namespaceManager.QueueExists(QueueName))
                    {
                        namespaceManager.CreateQueue(QueueName);
                    }

                    // Get a client to the queue
                    var messagingFactory = MessagingFactory.Create(
                        namespaceManager.Address, 
                        namespaceManager.Settings.TokenProvider);
                    OrdersQueueClient = messagingFactory.CreateQueueClient(
                        "OrdersQueue");
                }
            }
        }

4.  Now, you will ensure your **Initialize** method gets called. In the
    **Solution Explorer**, double-click **Global.asax\\Global.asax.cs**.

5.  Add the following line to the bottom of the **Application\_Start**
    method:

        QueueConnector.Initialize();

6.  Finally, you will update your web code you created earlier, to
    submit items to the queue. In the **Solution Explorer**,
    double-click **Controllers\\HomeController.cs** that you created
    earlier.

7.  Update the **Submit()** method as follows to get the message count
    for the queue:

        public ActionResult Submit()
        {            
            // Get a NamespaceManager which allows you to perform management and
            // diagnostic operations on your Service Bus Queues.
            var namespaceManager = QueueConnector.CreateNamespaceManager();

            // Get the queue, and obtain the message count.
            var queue = namespaceManager.GetQueue(QueueConnector.QueueName);
            ViewBag.MessageCount = queue.MessageCount;

            return View();
        }

8.  Update the **Submit(OnlineOrder order)** method as follows to submit
    order information to the queue:

        public ActionResult Submit(OnlineOrder order)
        {
            if (ModelState.IsValid)
            {
                // Create a message from the order
                var message = new BrokeredMessage(order);
                
                // Submit the order
                QueueConnector.OrdersQueueClient.Send(message);
                return RedirectToAction("Submit");
            }
            else
            {
                return View(order);
            }
        }

9.  You can now run your application again. Each time you submit an
    order, the message count increases.

    ![][18]

## Create the Worker Role

You will now create the worker role that processes the order
submissions. This example uses the **Worker Role with Service Bus Queue** Visual Studio project template. First, you will use Server Explorer in Visual Studio to obtain the required credentials.

1. From the menu bar in Visual Studio, choose **View**, and then click **Server Explorer**. A **Windows Azure Service Bus** node appears within the Server Explorer hierarchy, as in the following figure.

	![][21]

2. In Server Explorer, right-click **Windows Azure Service Bus**, then click **Add New Connection**.

3. In the **Add Connection** dialog, type the name of the service namespace, the issuer name, and the issuer key. Then click **OK** to connect.

	![][22]

4.  In Visual Studio, in **Solution Explorer** right-click the
    **Roles** folder under the **MultiTierApp** project.

5.  Click **Add**, and then click **New Worker Role Project**. The **Add New Role Project** dialog appears.

	![][26]

6.  In the **Add New Role Project dialog**, click **Worker Role with Service Bus Queue**, as in the following figure:

	![][23]

7.  In the **Name** box, name the project **OrderProcessingRole**. Then click **Add**.

8.  In Server Explorer, right-click the name of your service namespace, then click **Properties**. In the Visual Studio **Properties** pane, the first entry contains a connection string that is populated with the service namespace endpoint containing the required authorization credentials. For example, see the following figure. Double-click **ConnectionString**, and then press **Ctrl+C** to copy this string to the clipboard.

	![][24]

9.  In Solution Explorer, right-click the **OrderProcessingRole** you created in step 7, then click **Properties**.

10.  In the **Settings** tab of the **Properties** dialog, click inside the **Value** box for **Microsoft.ServiceBus.ConnectionString**, and then paste the endpoint value you copied in step 8.

	![][25]

11.  Create an **OnlineOrder** class to represent the orders as you process them from the queue. You can reuse a class you have already created. In Solution Explorer, right-click the **OrderProcessingRole** project (right-click the project, not the role). Click **Add**, then click **Existing Item**.

12. Browse to the subfolder for **FrontendWebRole\Models**, and double-click **OnlineOrder.cs** to add it to this project.

13. Replace the value of the **QueueName** variable in **WorkerRole.cs** from `“ProcessingQueue”` to `“OrdersQueue”` as in the following code:

		// The name of your queue
		const string QueueName = "OrdersQueue";

14. Add the following using statement at the top of the WorkerRole.cs file:

		using FrontendWebRole.Models;

15. In the `Run()` function, add the following code inside the `if (receivedMessage != null)` loop, below the `Trace` statement:

		if (receivedMessage != null)
    	{
        	// Process the message
        	Trace.WriteLine("Processing", receivedMessage.SequenceNumber.ToString());

        	// Add these two lines of code
        	// View the message as an OnlineOrder
        	OnlineOrder order = receivedMessage.GetBody<OnlineOrder>();
        	Trace.WriteLine(order.Customer + ": " + order.Product, "ProcessingMessage");

        	receivedMessage.Complete();
    	}
![this is a vm image][imageID]

16.  You have completed the application. You can test the full
    application as you did earlier, by pressing F5. Note that the message count does not increment, because the worker role processes items from the queue and marks them as complete. You can see the trace output of your
    worker role by viewing the Windows Azure Compute Emulator UI. You
    can do this by right-clicking the emulator icon in the notification
    area of your taskbar and selecting **Show Compute Emulator UI**.

    ![][19]

    ![][20]

  [0]: ../Media/getting-started-multi-tier-01.png
  [1]: ../Media/getting-started-multi-tier-100.png
  [2]: ../Media/getting-started-multi-tier-101.png
  [Get Tools and SDK]: http://go.microsoft.com/fwlink/?LinkID=234939&clcid=0x409
  [3]: ../Media/getting-started-3.png
  [4]: ../Media/getting-started-4.png
  [http://www.windowsazure.com]: http://www.windowsazure.com
  [5]: ../Media/getting-started-12.png
  [Windows Azure Management Portal]: http://manage.windowsazure.com
  [6]: ../Media/sb-queues-03.png
  [7]: ../Media/sb-queues-04.png
  [8]: ../Media/getting-started-multi-tier-09.png
  [9]: ../Media/getting-started-multi-tier-10.jpg
  [10]: ../Media/getting-started-multi-tier-11.png
  [11]: ../Media/getting-started-multi-tier-02.png
  [12]: ../Media/getting-started-multi-tier-12.png
  [13]: ../Media/getting-started-multi-tier-13.png
  [14]: ../Media/getting-started-multi-tier-33.png
  [15]: ../Media/getting-started-multi-tier-34.png
  [16]: ../Media/getting-started-multi-tier-35.png
  [17]: ../Media/getting-started-multi-tier-36.png
  [18]: ../Media/getting-started-multi-tier-37.png
  [19]: ../Media/getting-started-multi-tier-38.png
  [20]: ../Media/getting-started-multi-tier-39.png
  [21]: ../Media/SBExplorer.jpg
  [22]: ../Media/SBExplorerAddConnect.jpg
  [23]: ../Media/SBWorkerRole1.jpg
  [24]: ../Media/SBExplorerProperties.jpg
  [25]: ../Media/SBWorkerRoleProperties.jpg
  [26]: ../Media/SBNewWorkerRole.jpg
[imageID]: ../Media/NewImage.png