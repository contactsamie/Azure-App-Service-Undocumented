Is there way to determine why Azure App Service restarted?
Ask Question
up vote
14
down vote
favorite
5
I have a bunch of websites running on a single instance of Azure App Service, and they're all set to Always On. They all suddenly restarted at the same time, causing everything to go slow for a few minutes as everything hit a cold request.

I would expect this if the service had moved me to a new host, but that didn't happen -- I'm still on the same hostname.

CPU and memory usage were normal at the time of the restart, and I didn't initiate any deployments or anything like that. I don't see an obvious reason for the restart.

Is there any logging anywhere that I can see to figure out why they all restarted? Or is this just a normal thing that App Service does from time to time?

azure azure-web-sites azure-web-app-service
shareeditflag
asked Jul 10 '17 at 21:10

Nicholas Piasecki
20.6k46383
Have you tried connecting your app service to Application Insights, which will then allow you to view detailed history and logs? – Steveland83 Jul 10 '17 at 21:20
1
I have. I can see hints from my log4net traces which forward to AI that it was a "graceful" restart as I have an Azure Functions app hosted on the instance, and its CancellationToken was called. But none of the traces reveal why they all rebooted. – Nicholas Piasecki Jul 10 '17 at 22:13 
In that case I'd hazard a guess that it was an IIS App pool recycle that did it - you can get IIS to log those if you'd like to test that: iis.net/configreference/system.applicationhost/applicationpools/… – Steveland83 Jul 10 '17 at 22:16
I'm pretty sure I can't access that confit on App Service, this is not a IaaS VM – Nicholas Piasecki Jul 10 '17 at 23:33
1
They all suddenly restarted at the same time Does the issue appear occasionally or frequently? Besides, if possible, please try to scale out to additional instances for your apps, and check if it will help to mitigate the issue. – Fei Han Jul 11 '17 at 6:32
1
Are you running your apps on Basic/ std/ premium app service plan? If yes, you can use 'Resource Health' to check the resource state and if it running as expected. It may give you some more insights. – Mihir Jul 11 '17 at 9:48
It appears occasionally -- in fact, this is the first time that I've seen it happen without a hostname change. It is a standard plan and Resource Health indicated that everything was fine. RAM is constant at 75% and CPU is constantly between 30-40%, which is normal. It's just that all 8 apps restarted at the same time and the hostname did not change, which I can tell by the Site up time in Kudu for each one. I'm fine with them restarting say at night, but 4:40 p.m. EDT is an odd time for everything to suddenly recycle. – Nicholas Piasecki Jul 11 '17 at 12:07 
I see. How about checking the 'Quota' and add alert rules on the metrics e.g. Http 40*, CPU usage, etc? If I am not wrong, web apps may restart automatically if the consumption (CPU/ Memory/ FileSystem) hits the threshold. – Mihir Jul 12 '17 at 2:37 
Azure App Service is PaaS, infrastructure is kept in healthy condition by platform itself and this process is not transparent to us, but if there is a downtime Azure notify its customers over emails. I hope this link may shed some light - docs.microsoft.com/en-in/azure/app-service-web/… – Mihir Jul 12 '17 at 2:48
add a comment
start a bounty
1 Answer
active oldest votes
up vote
17
down vote
accepted
So, it seems the answer to this is "no, you can't really know why, you can just infer that it did."

I mean, you can add some Application Insights logging like

    private void Application_End()
    {
        log.Warn($"The application is shutting down because of '{HostingEnvironment.ShutdownReason}'.");

        TelemetryConfiguration.Active.TelemetryChannel.Flush();

        // Server Channel flush is async, wait a little while and hope for the best
        Thread.Sleep(TimeSpan.FromSeconds(2)); 
    }
and you will end up with "The application is shutting down because of 'ConfigurationChange'." or "The application is shutting down because of 'HostingEnvironment'.", but it doesn't really tell you what's going on at the host level.

What I needed to accept is that App Service is going to restart things from time to time, and ask myself why I cared. App Service is supposed to be smart enough to wait for the application pool to be warmed up before sending requests to it (like overlapped recycling). Yet, my apps would sit there CPU-crunching for 1-2 minutes after a recycle.

It took me a while to figure out, but the culprit was that all of my apps have a rewrite rule to redirect from HTTP to HTTPS. This does not work with the Application Initialization module: it sends a request to the root, and all it gets its a 301 redirect from the URL Rewrite module, and the ASP.NET pipeline isn't hit at all, the hard work wasn't actually done. App Service/IIS then thought the worker process was ready and then sends traffic to it. But the first "real" request actually follows the 301 redirect to the HTTPS URL, and bam! that user hits the pain of a cold start.

I added a rewrite rule described here to exempt the Application Initialization module from needing HTTPS, so when it hits the root of the site, it will actually trigger the page load and thus the whole pipeline:

<rewrite>
  <rules>
    <clear />
    <rule name="Do not force HTTPS for application initialization" enabled="true" stopProcessing="true">
      <match url="(.*)" />
      <conditions>
        <add input="{HTTP_HOST}" pattern="localhost" />
        <add input="{HTTP_USER_AGENT}" pattern="Initialization" />
      </conditions>
      <action type="Rewrite" url="{URL}" />
    </rule>
    <rule name="Force HTTPS" enabled="true" stopProcessing="true">
      <match url="(.*)" ignoreCase="false" />
      <conditions>
        <add input="{HTTPS}" pattern="off" />
      </conditions>
      <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" appendQueryString="true" redirectType="Permanent" />
    </rule>
  </rules>
</rewrite>
It's one of many entries in a diary of moving old apps into Azure -- turns out there's a lot of things you can get away with when something's running on a traditional VM that seldom restarts, but it'll need some TLC to work out the kinks when migrating to our brave new world in the cloud....

--

UPDATE 10/27/2017: Since this writing, Azure has added a new tool under "Diagnose and solve problems". Click "Web App Restarted", and it'll tell you the reason, usually because of storage latency or infrastructure upgrades. The above still stands though, in that when moving to Azure App Service, the best way forward is you really just have to coax your app into being comfortable with random restarts.

--

UPDATE 2/11/2018: After migrating several legacy systems to a single instance of a medium App Service Plan (with plenty of CPU and memory overhead), I was having a vexing problem where my deployments from staging slots would go seamlessly, but whenever I'd get booted to a new host because of Azure infrastructure maintenance, everything would go haywire with downtime of 2-3 minutes. I was driving myself nuts trying to figure out why this was happening, because App Service is supposed to wait until it receives a successful response from your app before booting you to the new host.

I was so frustrated by this that I was ready to classify App Service as enterprise garbage and go back to IaaS virtual machines.

It turned out to be multiple issues, and I suspect others will come across them while porting their own beastly legacy ASP.NET apps to App Service, so I thought I'd run through them all here.

The first thing to check is that you're actually doing real work in your Application_Start. For example, I'm using NHibernate, which while good at many things is quite a pig at loading its configuration, so I make sure to actually create the SessionFactory during Application_Start to make sure that the hard work is done.

The second thing to check, as mentioned above, is that you don't have a rewrite rule for SSL that is interfering with App Service's warmup check. You can exclude the warmup checks from your rewrite rule as mentioned above. Or, in the time since I originally wrote that work around, App Service has added an HTTPS Only flag that allows you to do the HTTPS redirect at the load balancer instead of within your web.config file. Since it's handled at a layer of indirection above your application code, you don't have to think about it, so I would recommend the HTTPS Only flag as the way to go.

The third thing to consider is whether or not you're using the App Service Local Cache Option. In brief, this is an option where App Service will copy your app's files to the local storage of the instances that it's running on rather than off of a network share, and is a great option to enable if your app doesn't care if it loses changes written to the local filesystem. It speeds up I/O performance (which is important because, remember, App Service runs on potatoes) and eliminates restarts that are caused by any maintenance on network share. But, there is a specific subtlety regarding App Service's infrastructure upgrades that is poorly documented and you need to be aware of. Specifically, the Local Cache option is initiated in the background in a separate app domain after the first request, and then you're switched to the app domain when the local cache is ready. That means that App Service will hit a warmup request against your site, get a successful response, point traffic to that instance, but (whoops!) now Local Cache is grinding I/O in the background, and if you have a lot of sites on this instance, you've ground to a halt because App Service I/O is horrendous. If you don't know this is happening, it looks spooky in the logs because it's as if your app is starting up twice on the same instance (because it is). The solution is to follow this Jet blog post and create an application initialization warmup page to monitors for the environment variable that tells you when the Local Cache is ready. This way, you can force App Service to delay booting you to the new instance until the Local Cache is fully prepped. Here's one that I use to make sure I can talk to the database, too:

public class WarmupHandler : IHttpHandler
{
    public bool IsReusable
    {
        get
        {
            return false;
        }
    }

    public ISession Session
    {
        get;
        set;
    }

    public void ProcessRequest(HttpContext context)
    {
        if (context == null)
        {
            throw new ArgumentNullException("context");
        }

        var request = context.Request;
        var response = context.Response;

        var localCacheVariable = Environment.GetEnvironmentVariable("WEBSITE_LOCAL_CACHE_OPTION");
        var localCacheReadyVariable = Environment.GetEnvironmentVariable("WEBSITE_LOCALCACHE_READY");
        var databaseReady = true;

        try
        {
            using (var transaction = this.Session.BeginTransaction())
            {
                var query = this.Session.QueryOver<User>()
                    .Take(1)
                    .SingleOrDefault<User>();
                transaction.Commit();
            }
        }
        catch
        {
            databaseReady = false;
        }

        var result = new
        {
            databaseReady,
            machineName = Environment.MachineName,
            localCacheEnabled = "Always".Equals(localCacheVariable, StringComparison.OrdinalIgnoreCase),
            localCacheReady = "True".Equals(localCacheReadyVariable, StringComparison.OrdinalIgnoreCase),
        };

        response.ContentType = "application/json";

        var warm = result.databaseReady && (!result.localCacheEnabled || result.localCacheReady);

        response.StatusCode = warm ? (int)HttpStatusCode.OK : (int)HttpStatusCode.ServiceUnavailable;

        var serializer = new JsonSerializer();
        serializer.Serialize(response.Output, result);
    }
}
Also remember to map a route and add the application initialization your web.config:

<applicationInitialization doAppInitAfterRestart="true">
  <add initializationPage="/warmup" />
</applicationInitialization>
The fourth thing to consider is that sometimes App Service will restart your app for seemingly garbage reasons. It seems that setting the fcnMode property to Disabled can help; it prevents the runtime from restarting your app if someone diddles with configuration files or code on the server. If you're using staging slots and doing deployments that way, this shouldn't bother you. But if you expect to be able to FTP in and diddle with a file and see that change reflected in production, then don't use this option:

     <httpRuntime fcnMode="Disabled" targetFramework="4.5" />
The fifth thing to consider, and this was primarily my problem all along, is whether or not you are using staging slots with the AlwaysOn option enabled. The AlwaysOn option works by pinging your site every minute or so to make sure it's warm so that IIS doesn't spin it down. Inexplicably, this isn't a sticky setting, so you may have turned on AlwaysOn on both your production and staging slots so you don't have to mess with it every time. This causes a problem with App Service infrastructure upgrades when they boot you to a new host. Here's what happens: let's say you have 7 sites hosted on an instance, each with its own staging slot, everything with AlwaysOn enabled. App Service does the warmup and application initialization to your 7 production slots and dutifully waits for them to respond successfully before redirecting traffic over. But it doesn't do this for the staging slots. So it directs traffic over to the new instance, but then AlwaysOn kicks in 1-2 minutes later on the staging slots, so now you have 7 more sites starting up at the same time. Remember, App Service runs on potatoes, so all this additional I/O happening at the same time is going to destroy the performance of your production slots and will be perceived as downtime.

The solution is to keep AlwaysOn off on your staging slots so you don't get nailed by this simultaneous I/O frenzy after an infrastructure update. If you are using a swap script via PowerShell, maintaining this "Off in staging, On in production" is surprisingly verbose to do:

Login-AzureRmAccount -SubscriptionId {{ YOUR_SUBSCRIPTION_ID }}

$resourceGroupName = "YOUR-RESOURCE-GROUP"
$appName = "YOUR-APP-NAME"
$slotName = "YOUR-SLOT-NAME-FOR-EXAMPLE-STAGING"

$props = @{ siteConfig = @{ alwaysOn = $true; } }

Set-AzureRmResource `
    -PropertyObject $props `
    -ResourceType "microsoft.web/sites/slots" `
    -ResourceGroupName $resourceGroupName `
    -ResourceName "$appName/$slotName" `
    -ApiVersion 2015-08-01 `
    -Force

Swap-AzureRmWebAppSlot `
    -SourceSlotName $slotName `
    -ResourceGroupName $resourceGroupName `
    -Name $appName

$props = @{ siteConfig = @{ alwaysOn = $false; } }

Set-AzureRmResource `
    -PropertyObject $props `
    -ResourceType "microsoft.web/sites/slots" `
    -ResourceGroupName $resourceGroupName `
    -ResourceName "$appName/$slotName" `
    -ApiVersion 2015-08-01 `
    -Force
This script sets the staging slot to have AlwaysOn turned on, does the swap so that staging is now production, then sets the staging slot to have AlwaysOn turned off, so it doesn't blow things up after an infrastructure upgrade.

Once you get this working, it is indeed nice to have a PaaS that handles security updates and hardware failures for you. But it's a little bit more difficult to achieve in practice than the marketing materials might suggest. Hope this helps someone.

shareeditflag
edited Feb 11 at 21:16
answered Jul 21 '17 at 16:07

Nicholas Piasecki
20.6k46383
I would never have known about the new Diagnose and solve problems option. Thank you. – Neil Thompson Dec 1 '17 at 12:23
I upvoted because of your update. Did not know that was there and it's very helpful. Thanks! – madamission Feb 9 at 13:42
@NeilThompson thanks for your effort to document your findings. How is this still an open bug in Azure for more then half a year now.. isnt Microsoft aware that some people actually use Azure in production? – Simon Heinen Mar 2 at 9:52
upvote
flag
You are my hero, trying to get this setup with deployment slots since my app keeps getting taken down from storage issues and the performance is killing me, app start ups are taking forever though (3 minutes) ugg, any way to speed app start ups? I also noticed that restarting app is way quicker now, not sure what's going on behind the scenes to fix this – TWilly Apr 5 at 22:15
