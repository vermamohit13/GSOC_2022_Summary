# GSOC_2022_Summary
Hello, I am Mohit Verma and I was selected as GSOC 2022 contributor in Open Printing @ The Linux Foundation.I have worked on the project for creating a GUI for discovering non-driverless printers and finding suitable Printer Applications for them. I have completed the task of acheiving the functionality for this GUI along with updating the discovery of devices and implementing new method for the discovery of Printer Application in cups-pk-helper. 

I have also taken an extension of One and a half month so that the discussion for the final design of GUI could be completed but it is taking more time so, it is still in the process.

cups-pk-helper - https://gitlab.freedesktop.org/cups-pk-helper/cups-pk-helper/-/merge_requests/7

Gnome Settings - https://github.com/vermamohit13/gnome-control-center/tree/cupsdev

# Project Description
Modern printers usually are driverless IPP printers, and those get discovered and set up fully automatically with CUPS, no Printer Application is required for them, so it is easy for users to get up and running with them.

Printers which do not do driverless IPP are either legacy printers, the many older printers which got developed before driverless IPP printing existed, and specialty printers. These need Printer Applications. As there will be several different Printer Applications and each one supporting another set of printers it is not trivial for the user to discover available non-IPP-driverless printers and find out which is the Printer Application to use and whether it is already installed.

So we need some guide for the user. The idea is a GUI tool which lists available, non-IPP-driverless printers, local (USB) and network devices. If the user selects one of them, all installed Printer Applications which support this printer are shown, and for each a button to open the Printer Application's web interface and also a quick auto-add-this-printer button. In addition to the list of suitable Printer Applications there should also be a button which does a fuzzy search for the printer make and model on the Snap Store/the OpenPrinting web site to find Printer Applications which are not installed on the local system. There is already a concept to implement an appropriate search index on OpenPrinting which will be used by this GUI.

# Updating discover of devices 









# Discovery of Printer Application








# GUI for Printer Application (Proposed Design)












# Temporary GUI with functionality









