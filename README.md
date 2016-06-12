# Akka.NET-on-Azure
Information for running Akka.NET on Microsoft Azure

## Running your site as an App Service

Application pool recycling will cause issues when running as an App Service (Web App). A [Stack Overflow post](http://stackoverflow.com/questions/27421024/disable-pool-recycling-on-azure-websites) describes what to do. In a nutshell:

1. Navigate to your Site (or specific slot if testing in a slot) within the new Azure Portal.
2. Click on "Tools" in the header of the app
3. Click on "Kudu" in the menu, then click "Go". This will launch a new window, opening up https://yoursite.scm.azurewebsites.net/ or https://yoursite-slotname.scm.azurewebsites.net/ if you're testing this out in a slot.
4. Click the "Debug Console" menu, then click "CMD". 
5. Ignore the command prompt; instead, you'll be using the file navigation above it.
6. Click the world icon (Site Root), then click on the "Config" folder.
7. Edit the file `applicationHost.config`
8. Ctrl+F and search for the string `~1`. You will see something like: 
   
   ```xml
   <applicationPools>
     <add name="YOUR_SITE_NAME" managedRuntimeVersion="v4.0">
       <processModel identityType="ApplicationPoolIdentity" />
     </add>
     <add name="~1YOUR_SITE_NAME" managedRuntimeVersion="v4.0" managedPipelineMode="Integrated">
       <processModel identityType="ApplicationPoolIdentity" />
     </add>
   </applicationPools>
   ```
9. Without a `<recycling ...>` section, this means your app will recycle. To change this, we need a config transformation.
10. Create a new file called `applicationHost.xdt` and use the following:
   
   ```xml
   <?xml version="1.0"?>
   <configuration xmlns:xdt="http://schemas.microsoft.com/XML-Document-Transform">>
     <system.applicationHost>
       <applicationPools>
         <add name="YOUR_SITE_NAME" xdt:Locator="Match(name)">
           <recycling disallowOverlappingRotation="true" xdt:Transform="Insert" />
         </add>
         <add name="~1YOUR_SITE_NAMEd" xdt:Locator="Match(name)">
           <recycling disallowOverlappingRotation="true" xdt:Transform="Insert" />
         </add>
       </applicationPools>
     </system.applicationHost>
   </configuration>
   ```
11. Replace `YOUR_SITE_NAME` and `~1YOUR_SITE_NAME` with the corresponding names that you found within `applicationHost.config`
12. Upload this file to your site. Your normal site lives at `\home\site\wwwroot\` and you want the `applicationHost.xdt` file to live at `\home\site`. Once uploaded, go back to Kudu and click the "Home" (house) icon and navigate to the `site` directory. You should see your `applicationHost.xdt` file there.
13. Now back to the Azure Portal, click "Restart" for your site.
14. To verify that this transformation took place, repeat steps 6 through 8. Your `applicationHost.config` file should now display something like:
   
   ```xml
   <applicationPools>
     <add name="YOUR_SITE_NAME" managedRuntimeVersion="v4.0">
       <processModel identityType="ApplicationPoolIdentity" />
       <recycling disallowOverlappingRotation="true" />
     </add>
     <add name="~1YOUR_SITE_NAME" managedRuntimeVersion="v4.0" managedPipelineMode="Integrated">
       <processModel identityType="ApplicationPoolIdentity" />
       <recycling disallowOverlappingRotation="true" />
     </add>
   </applicationPools>
   ```

Your app should now no longer recycle itself automatically.
