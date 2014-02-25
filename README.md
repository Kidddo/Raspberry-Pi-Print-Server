##Kidddo Print Server Raspberry Pi
This guide will help you set up a [Kidddo](http://kidddo.org) Print Server on a Raspberry Pi. Portions of this guide are adapted from another [repository](https://github.com/churchio/checkin-printer).

If you are new to the Raspberry Pi, check out the following links:

* [What is a Raspberry Pi](http://www.youtube.com/watch?v=e0wkVVVLvR8#t=49)?
* [Where to Buy a Pi](https://www.google.com/shopping/product/16525736034140563056?q=raspberry+pi&client=safari&rls=en&bav=on.2,or.r_qf.&bvm=bv.59930103,d.cGU,pv.xjs.s.en_US.eYG7PyzNpLg.O&biw=1532&bih=1011&tch=1&ech=1&psi=CBvkUqGmBcmFogTU3oHgAQ.1390680841259.3&prds=hsec:online&ei=CRvkUtXXPIT9oATDxoHIBw&ved=0CIIEENkrMAA)

===
###Requirements
* A Raspberry Pi Model B with power adapter.
* An SD card with at least 1 GB of storage.
* A network cable.
* A DYMO LabelWriter 450 (or compatible label printer) connected via USB.

===
###Setup

####Get Raspbian
* Download Raspbian "Wheezy" from [here](http://www.raspberrypi.org/downloads) (Choose Direct download YYYY-MM-DD-wheezy-raspbian.zip).
* Copy the image to your SD card using your computer. ([Guide](http://elinux.org/RPi_Easy_SD_Card_Setup))
* Insert the SD card into your Raspberry Pi and plug it in. Make sure the network cable is plugged in as well.
* Locate your PI's IP address:
	* Use a network utility like [LanScan](https://itunes.apple.com/us/app/lanscan/id472226235?mt=12):![image](http://cr8.me/u/screen_shot_2014-02-19_at_3.46.05_pm_7d9f.png)
	* OR log into your router and look at the DHCP lease table.
* SSH to your PI: `ssh pi@yourIPaddress` eg. `ssh pi@192.168.0.18`. The default password is `raspberry`.
* Change your password by typing `passwd` immediately after login.

####Install Software
First, we'll install CUPS (Common Unix Printing System), and Vim (command line editor). This may take several minutes:

    sudo aptitude install cups vim

Next, we need to install Node.js (our print server platform). Start by downloading the PI build:
    
    wget http://nodejs.org/dist/v0.10.2/node-v0.10.2-linux-arm-pi.tar.gz

Unpack it:

    tar -xvzf node-v0.10.2-linux-arm-pi.tar.gz
    
Open or create `.bash_profile`:

    vim .bash_profile

And add the following and then `:wq` write & quit it:

    NODE_JS_HOME=/home/pi/node-v0.10.2-linux-arm-pi 
    PATH=$PATH:$NODE_JS_HOME/bin 

Reboot the PI:

    sudo reboot

Log back in via ssh. If node installed correctly, we can check the version:

    node -v

The response should be something like: `v0.10.2`. 

Next, we'll use NPM to install a few more modules needed for our PrintServer script (give it a few more minutes):
    
    npm install ipp pdfkit ibtrealtimesjnode


Ok, now let's make a new directory in your PI's home folder:

    mkdir kidddo

Change Directory to "kidddo":

    cd kidddo

Download Kidddo Print Server:

    git clone git://github.com/Kidddo/Raspberry-Pi-Print-Server

Change Directory to "Rasberry-Pi-Print-Server":

    cd Raspberry-Pi-Print-Server

Open PrintServer.js in Vim editor to add your organization's token:

    vim PrintServer.js

Replace `abc` on line 4 with your organization's token (found in Settings area of your Admin site: Settings>Set Printer>Server) and save the file. If you are new to Vim, here's a command cheat sheet: [http://www.fprintf.net/vimCheatSheet.html](http://www.fprintf.net/vimCheatSheet.html).

===


Next we'll install a package that will automatically run our print server every time the the PI boots-up (this will take a while):

    npm install -g forever

And finally, open rc.local in Vim:

    sudo vim /etc/rc.local

And add the following just before exit:

    sudo -u pi forever start -l /home/pi/kidddo/Raspberry-Pi-Print-Server/PrintServer.log -a /home/pi/kidddo/Raspberry-Pi-Print-Server/PrintServer.js

####Configure Printers
Add your user (pi) to the to the lpadmin group (so we can manage printers):

    sudo usermod -aG lpadmin pi

Next, we'll make a few changes to the CUPS Configuration:

    sudo vim /etc/cups/cupsd.conf

Change `Listen localhost:631` to `Listen 0.0.0.0:631`:

    # Only listen for connections from the local machine.
    Listen 0.0.0.0:631
    Listen /var/run/cups/cups.sock
    
Add `Allow @LOCAL` to both the `<Location />` and `<Location /admin>` sections:

    # Restrict access to the server...
    <Location />
      Allow @LOCAL
      Order allow,deny
    </Location>
    
    # Restrict access to the admin pages...
    <Location /admin>
      Allow @LOCAL
      Order allow,deny
    </Location>

Save (write & quit `:wq`) cupsd.conf

Restart CUPS:

    sudo service cups restart

Now we can leave the command line and open your web browser. Access the CUPS admin interface via the URL: `http://yourIPaddress:631/admin` eg. `http://192.168.0.18:631/admin`:

![image](http://cr8.me/u/screen_shot_2014-02-21_at_9.53.34_am_c578.png)

Click the "Add Printer" button. If prompted, enter your user info (default "pi"/"raspberry").

Select Your label printer and click "Continue":

![image](http://cr8.me/u/screen_shot_2014-02-21_at_9.57.56_am_ac13.png)

Enter a unique name for this printer and click "Continue". This is the name you will enter on all devices that print to this printer:

![image](http://cr8.me/u/screen_shot_2014-02-21_at_9.59.29_am_9d61.png)

Click "Add Printer" on the Confirmation Screen.

In "Set Default Options" change the Media Size to "Shipping Address" and click "Set Default Options" (If you have modified the label template in PrintServer.js or are using a printer other than DYMO LabelWriter, adjust these options accordingly):

![image](http://cr8.me/u/screen_shot_2014-02-21_at_10.04.57_am_b6a9.png)

Test your printer by selecting "Print Test Page" from the Maintenance Dropdown:

![image](http://cr8.me/u/screen_shot_2014-02-21_at_10.11.16_am_636d.png)

Congrats! Repeat the process to add any additional printers. When finished, reboot your PI:

    sudo reboot

In your Kidddo Admin Settings tab, Enable "Use Label Printers" and click "Save Settings". Then click "Set Printer" and enter your selected printer name eg. `DYMO_1` and click Print Test Label:

![image](http://cr8.me/u/screen_shot_2014-02-21_at_10.14.51_am_f18b.png)

