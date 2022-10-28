# GSOC_2022_Summary
Hello, I am Mohit Verma and I was selected as GSOC 2022 contributor in Open Printing @ The Linux Foundation.I have worked on the project for creating a GUI for discovering non-driverless printers and finding suitable Printer Applications for them. I have completed the task of acheiving the functionality for this GUI along with updating the discovery of devices and implementing new method for the discovery of Printer Application in cups-pk-helper. 

I have also taken an extension of One and a half month so that the discussion for the final design of GUI could be completed but it is taking more time so, it is still in the process.

cups-pk-helper - https://gitlab.freedesktop.org/cups-pk-helper/cups-pk-helper/-/merge_requests/7

Gnome Settings - https://github.com/vermamohit13/gnome-control-center/tree/cupsdev-main

# Project Description
Modern printers usually are driverless IPP printers, and those get discovered and set up fully automatically with CUPS, no Printer Application is required for them, so it is easy for users to get up and running with them.

Printers which do not do driverless IPP are either legacy printers, the many older printers which got developed before driverless IPP printing existed, and specialty printers. These need Printer Applications. As there will be several different Printer Applications and each one supporting another set of printers it is not trivial for the user to discover available non-IPP-driverless printers and find out which is the Printer Application to use and whether it is already installed.

So we need some guide for the user. The idea is a GUI tool which lists available, non-IPP-driverless printers, local (USB) and network devices. If the user selects one of them, all installed Printer Applications which support this printer are shown, and for each a button to open the Printer Application's web interface and also a quick auto-add-this-printer button. In addition to the list of suitable Printer Applications there should also be a button which does a fuzzy search for the printer make and model on the Snap Store/the OpenPrinting web site to find Printer Applications which are not installed on the local system. There is already a concept to implement an appropriate search index on OpenPrinting which will be used by this GUI.

This GUI will finally be the part of GNOME-CONTROL-CENTER.

# Updating discovery of devices 

The current discovery of devices by `get_cups_devices_async()` in `pp-utils.c` in [GNOME Settings]() happens by running cups backends. Since, CUPS 3.x these backends will mostly go down the drain. So, we need to find new methods of discovering devices. Hence,for the discovery of devices , I and [till.kamppeter](https://github.com/tillkamppeter)  has thought of the following three calls for the discovery of non-driverless printer discovery:

1) Call all already installed Printer Applications with `PRINTER_APP devices`. This is the only future-proof (works with New Architecture/CUPS 3.x) way to discover specialty printers which PAPPL's own backends do not discover. This means also that in the future there are some specialty printers like dye-subs (Gutenprint) or Braille embossers which need their Printer Application already installed in order to get auto-discovered by printer setup tools.

2) Call device discovery of PAPPL, this discovers at least classic USB and network (port 9100) printers.                                                                
3) Do classic CUPS (<= 2.x) discovery. Principally, this is already implemented in the G-C-C module you are working on, but you have to change this implementation, so that G-C-C builds with both libcups2 and libcups3, so you have to remove convenience API calls which most probably go away in libcups3. Safest bet is what I already taold, call lpinfo -v and fail silently if (most probably due to CUPS 3.x in use) you get an error or an empty response.

This way we are safe for everything: pure CUPS 2.x, pure CUPS 3.x, CUPS 2.x but already Printer Applications installed.

# Discovery of Printer Application

To call all the already installed printer application via the `PRINTER_APP devices`, we first need to discover them. All the printer application advertise themselves as IPP System Service, `_ipps-system._tcp` and also with a web interface, Web Site. So, I used Avahi via DBUS calls to scan DNS-SD records for the discovery of printer application. The `printer_app_discovery()` method is implemented in cups-pk-helper which is now able to do the required task. 

``` 
    void 
printer_app_discovery (gpointer user_data)
{
        Avahi *printer_app_backend = user_data;
        printer_app_backend->avahi_cancellable = g_cancellable_new ();
        printer_app_backend->dbus_connection = g_bus_get_sync (G_BUS_TYPE_SYSTEM, printer_app_backend->avahi_cancellable, NULL);

        avahi_create_browsers (printer_app_backend);

        g_list_foreach (printer_app_backend->services,(GFunc) _cph_cups_get_printer_app_cb, printer_app_backend);       

        return;
}
```
The APIs which were created for this method are -
```
static void
avahi_create_browsers ( gpointer    user_data)

static void
avahi_service_browser_new_cb (GVariant*     output,
                              gpointer      user_data)
static void
avahi_service_browser_signal_handler (GDBusConnection *connection,
                                      const char      *sender_name,
                                      const char      *object_path,
                                      const char      *interface_name,
                                      const char      *signal_name,
                                      GVariant        *parameters,
                                      gpointer         user_data)
static gboolean
unsubscribe_general_subscription_cb (gpointer user_data)

static void
avahi_service_resolver_cb (GVariant*     output,
                           gpointer      user_data)
```
Along with these few more helper functions were created.

With the discovery of printer applications, we now have to run the devices method of the printer apps. From the DNS-SD records we have the URI for the printer applications but not their binary paths. Since, Printer Apps are full emulation of an IPP network printer we can make IPP requests for our task. But the problem is there is no interface for the discovery of devices in the Printer Apps. Hence, I had to make a slightly inefficient method 

```
void 
_cph_cups_get_printer_app_cb (gpointer        data,
                              gpointer        user_data)
```
to get the binary paths using several system calls. However this gets the job done.

 Update -
  I and [till.kamppeter](https://github.com/tillkamppeter) has put up a request on [pappl](https://github.com/michaelrsweet/pappl) to [Add IPP operations to request discovered devices and available drivers from Printer Application as server](https://github.com/michaelrsweet/pappl/issues/214). It will be released soon. With this we will be able to make IPP requests and eventually the need for the above function will be over.


# Using the new discovery of devices
In cups-pk-helper's `_cph_cups_devices_get()` method will call the below call for the discovery of devices via PAPPL. Also, with the binary path of printer apps from `printer_app_discovery()` method we will now be able to complete the approach we discussed in Updating discovery of devices section.

``` 
        papplDeviceList ( PAPPL_DEVTYPE_ALL,
                          _cph_cups_pappl_device_cb,                  
                           pappl_data,
                          _cph_cups_pappl_device_err_cb,
                          cups ); 
```
The Printer Application are called by the below code segment -

```
for (int i = 0; i < g_list_length (printer_app_backend->services); i++)
   	  { 
             AvahiData *printer_app_data = g_list_nth_data (printer_app_backend->services,i); 
             char *cmd = g_strdup_printf ("%s devices",printer_app_data->binary_path); 
             
             if ((fp = exec_command (cups,"cmd")) == NULL)
             {
                return FALSE;
             }

             if (_parse_app_devices (cups, data, fp))
             {
                 _cph_cups_set_internal_status ( cups, 
                                 g_strdup_printf ("Error while getting devices from %s.",printer_app_data->binary_path));
                 return FALSE;
             }
```
It returns false to indicate a failure if the printer app commands cannot discover devices.

Update - 
 With the request to [Add IPP operations to request discovered devices and available drivers from Printer Application as server](https://github.com/michaelrsweet/pappl/issues/214) . We just now have to make a IPP request to the printer Apps and call the new created method `_cph_cups_get_devices_cb` with a new field of URI as a parameter to acheive functionality to open web interface of printer application. This method will be implemented as soon as the PAPPL v1.3 will be released.
 
 
# GUI for Printer Application 
In order to get this GUI implemented the I and [till.kamppeter](https://github.com/tillkamppeter) has created a new request on the [GNOME Settings Gitlab Page](https://gitlab.gnome.org/GNOME/gnome-control-center/-/issues/1878)

The discussions are still going on about the GUI. We have also contacted Elio at Canonical to get his opinion too. As this process was taking more time than expected. I proceeded and developed a temporary GUI which holds the following buttons to obtain the aim of the project -

1) Search - This button will query Openprinting Web Server ( see https://openprinting.github.io/OpenPrinting-News-November-2021/#printer-querying-on-the-  openprinting-web-server )   via a cul_easy API call provided by  libcurl about the selected printer. This query will return the list of supported         Printer Application along with the internal driver name of the Printer Application for the printer, a human-readable description (the one which you     also see in the web interface of the Printer Application), and the device ID as it is registered for this printer in the Printer Application. This is already implemented and is working well in the temporary UI [see](https://drive.google.com/file/d/1eSJinN_NxyimeTPr_ZQDc0omeI-0lZwH/view?usp=sharing) . It will also have a Recommended tag along with the name of the most appropriate Printer Application.

2) Web Interface Button - It is supposed to open the web interface of the Printer Application for the selected Printer after it is installed. As per Marek , this button should be in the Printer Details Dialog. This button will work similar to the link to Address : ‘localhost’ in the Printer Details Dialog.
 
3) Install - This button will become sensitive when a supported Printer Application (which is not Installed otherwise it should be insensitive)  is selected in the window where supported Printer Application is displayed. This button will allow the user for an automatic Installation of the Printer Applications. This button will be working once the GNOME Settings team and my organization decides the exact method for its working.

4) Auto-Add - This button will add the current printer in the selected Printer Application of the user's choice. Also, the display section of Printer Application will have the most appropriate App preselected along with a "Recommended" tag.

The Temporary dialog is designed in [Camabalanche](https://gitlab.gnome.org/jpu/cambalache) 

![Design of Temporary GUI](https://github.com/vermamohit13/GSOC_2022_Summary/blob/main/Images/Screenshot%20from%202022-10-10%2021-35-30.png)

To test this GUI I have also made few changes and integrated it into G-C-C. I have used a sample "Device-id" to query the Open Printing Web Server and test the functionality. The button works well and is able to query the server and get the supported Application name along with few other details.

![Testing the GUI](https://github.com/vermamohit13/GSOC_2022_Summary/blob/main/Images/Screenshot%20from%202022-10-19%2015-58-29.png)

# Update
With our request to [Add IPP operations to request discovered devices and available drivers from Printer Application as server](https://github.com/michaelrsweet/pappl/issues/214) completed. New IPP methods will be implemented to query Printer Application for Discovery of devices.


