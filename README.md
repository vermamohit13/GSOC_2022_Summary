# GSOC_2022_Summary
Hello, I am Mohit Verma and I was selected as GSOC 2022 contributor in Open Printing @ The Linux Foundation.I have worked on the project for creating a GUI for discovering non-driverless printers and finding suitable Printer Applications for them. I have completed the task of acheiving the functionality for this GUI along with updating the discovery of devices and implementing new method for the discovery of Printer Application in cups-pk-helper. 

I have also taken an extension of One and a half month so that the discussion for the final design of GUI could be completed but it is taking more time so, it is still in the process.

cups-pk-helper - https://gitlab.freedesktop.org/cups-pk-helper/cups-pk-helper/-/merge_requests/7

Gnome Settings - https://github.com/vermamohit13/gnome-control-center/tree/cupsdev

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
void 
_cph_cups_get_printer_app_cb (gpointer        data,
                              gpointer        user_data)
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
  I and [till.kamppeter](https://github.com/tillkamppeter) has put up a request on [pappl](https://github.com/michaelrsweet/pappl) to [Add IPP operations to request discovered devices and available drivers from Printer Application as server](https://github.com/michaelrsweet/pappl/issues/214).It will be released soon. With this we will be able to make IPP requests and eventually the need for the above function will be over.


# Using the new discovery of devices
With this Approach I have updated the cups-pk-helper `_cph_cups_devices_get()` function which will now also call to pappl for the discovery of devices.

``` 
        papplDeviceList ( PAPPL_DEVTYPE_ALL,
                          _cph_cups_pappl_device_cb,                  
                           pappl_data,
                          _cph_cups_pappl_device_err_cb,
                          cups ); 
```
Along with this , we will all also run the `printer_app devices` command for all the locally available printer application.










# GUI for Printer Application (Proposed Design)












# Temporary GUI with functionality









