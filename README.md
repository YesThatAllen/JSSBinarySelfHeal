# JSSBinarySelfHeal

Overview of JSS Binary Self Heal

What do we do when computers stop communicating with the JSS?  The computer is in another city, state or country and I have no way to get my hands on this device today.  A Self Heal workflow can help repair the broken communication and get that computer talking to the JSS again.  This is an automated process to verify client to server communication and repair any detection of failure.

How this works

1. Uses a launch daemon that runs on a scheduled time each day that runs a script to perform a series of steps to verify that the computer is managed.  I have just added a radom delay of ten minutes to the launch daemon.  I do have a policy that is being executed in the self heal, and I wanted to eliminate all managed clients from running a policy at the same time(LCV 11-19-15).
2. The script that is ran is performing the following taks:
    a. Does this computer have the JAMF binary
    b. Can this computer check the JSS connection
    c. Are we able to log this computers IP address to the JSS
    d. Are we able to execute a policy
    e. Is the MDM profile installed on this computer
3. Each computer that is using this Self Heal mechanism will have a local log that is tracking the verification checks.  These entries are logged with the current date and time the check was performed.
4. This log is then submitted ot the devices inventory record within the JSS
5. If a detected failure is found, the client is repaired in one of the following ways:
    a. Either a quickadd is downloaded from the JSS via curl and installed
    b. The client is enrolled into the JSS via invitation

Lets open Recon and create a quickadd package.  Save this quickadd package to your desktop.  Before we move on to the next step, please perform the following steps, so we can get the invitation id to update our SelfHeal.sh script:
    a. Use pacifist so we can view the quickadd package contents
    b. expand Resources and extract the postinstall file
    c. Open the postinstall file with TextWrangler or any text editor we prefer 
    d. Search for this line: $jamfCLIPath enroll -invitation and copy the number string after -invitation.  This is our invitation id that we will input into our SelfHeal.sh as the following variable: enrollInv=""
    e. Save changes to the SelfHeal.sh

Once the quickadd is created, lets compress this as a .zip.  We will want to rename the quickadd package to quickadd.zip.  Once we have the quickadd compressed, lets move it to our JSS in the following location:

Linux: /usr/local/jss/tomcat/webapps/ROOT/
Mac OS: /Library/JSS/tomcat/webapps/ROOT/
Windows: C:\Program Files\JSS\Tomcat\webapps\ROOT\

Lets now package our SelfHeal script and our Launch Daemon using Composer.  First we will place the files in the proper locations:

SelfHeal.sh: /Library/Scripts/
com.management.selfheal.plist: /Library/LaunchDaemons

Lets open Composer and perform the following tasks please:

1. Click New
2. Select User Environment
3. Click on Dashboard
4. Click Next
5. Type in administrator password when prompted
6. Right click on “Dashboard” under “Sources” and rename this package to SelfHeal
7. On the right pane, right click on the “Users” folder and delete it
8. Navigate to /Library/LaunchDaemons/ and drag "com.management.selfheal.plist" into the right pane of Composer
9. Expand all of the folders until we see “com.management.selfheal.plist" and click on it.  Change the owner of this file to "ROOT" and the group to "WHEEL"
10. Leave the permissions at the default
11. Navigate to /Library/Scripts/ and drag “SelfHeal.sh" into the right pane of Composer
12. Expand all of the folders until wee “SelfHeal.sh” and click on it.  Change the owner of this file to “ROOT” and the group to “WHEEL"
13. Set the permissions for this file to 744
14. Click build as DMG
15. Select our save location
16. Add the DMG to Casper Admin

We will also need to create an additional policy that our Self Heal will verify the client can run

1. Click the gear in the upper right corner of the Web Application
2. Click on "Computer Management"
3. Click the "Scripts" payload
4. Click "New"
5. Type in a display name for this script; something like SelfHeal
6. Click on the "Script" payload
7. Create a script that looks like this: 
8.  #!/bin/sh

    echo "heal"
8. Save this script
9. Click "Computers"
10. Click "Policies"
11. Create a New policy
12. Give this policy a display name
13. Assign this policy a category
14. Check the Custom Trigger box
    a. Name the Custom Event heal
15. Set our execution frequency to "Ongoing"
16. Click the "Scripts" paylod
17. Click on "Configure"
18. Choose our Script we just created
19. Scope this policy to all computers
20. Save the policy

We can now create a policy to push out our DMG to end users.  Our policy should look like this:

1. General payload filled out setting our trigger, execution frequency and the name of our policy
2. Add our DMG from above to install our LaunchDaemon and Script on the client
3. Click on the “Files and Process” payload and click edit
4. Scroll all the way down on the right pane and type the following under “Execute Command"
    a. launchctl load /Library/LaunchDaemons/com.management.selfheal.plist
5. Scope the policy to our end users

We now have a SelfHeal solution in place that will help keep our clients communicating with the JSS. The script that we designed will go through a serious of checks, if any of the checks fail, a patch is performed and the script exits.  There is logging for this process that can be found on the client at the following location:

/Users/Shared/enrollLog.log

This log is set to rollover once it reaches/exceeds 10MB.  The log will be archived in /Users/Shared/log_archive.  The Self Heal is going to archive 5 logs.  Once it reaches 5, the oldest 4 logs will be deleted keeping the newest archive log on the client.

If we wish to have transparency of the log from the JSS, we can download enrollLog.xml and upload it to the JSS.

1. Click the gear in the upper right corner
2. Click on "Computer Management"
3. Click on "Extension Attributes"
4. Click "Upload"
5. Locate enrollLog.xml and select it to be uploaded
6. Save

CAVEATS:

Each time the JSS is upgraded, we will need to create a new quickadd package.  We will want to create it the same way we did above and place it in the same directory as we did above.  When we make the new quickadd package, we will want to also update the invitation variable in the SelfHeal.sh.  We can then package up a new SelfHeal.sh with this update and push that out to the clients.  The script will overwrite the existing one in /Library/Scripts.

We can make a DMG using Composer that will lay down the SelfHeal.sh in /Library/Scripts.  Set the same ownership (root:wheel) and permissions 755 before building the DMG.
Add the modified SelfHeal.sh to Casper Admin and use a policy to push this out.
