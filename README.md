Configuring a JBoss Cluster in Domain mode
================================================
Author: Allen Boulom  
Level: Beginner  
Summary: This README demonstrates how to configure a JBoss cluster in domain mode.  
Target Product: JBoss EAP  
Source: https://github.com/aboulom/eap-cluster-domain

Overview
--------
Clustering allows for high availability by making your application available on secondary
servers when the primary instance is down or it lets you scale up or out by increasing
the server density on the host, or by adding servers on other hosts. It can even help to
increase performance with effective load balancing between servers based on their respective hardware.
In the domain mode it is possible to manage multiple server instances on one host.

Domain Controller Setup
-----------------------
The domain controller needs both the domain.xml and host.xml configured. In the EAP_HOME/domain/configuration
directory there are preconfigured host.xml files. You can use these instead of the host.xml for the domain 
controller (master) and host controller (slave) to use.

You can specify the configuration you want to use by adding the --host-config argument:
		
		domain.bat --host-config=host-master.xml
		
Whichever host you choose whether its host.xml or host-master.xml, make sure the empty element <local /> has
been added to the <domain-controller> section.
		
		<domain-controller>
			<local/>
		</domain-controller>
		
This is so that when WildFly looks to see which server is the domain controller, it knows to become the domain controller itself.

You can also bind the IP of the management interface so that it is not localhost by starting Jboss with this argument:
		
		domain.bat -Djboss.bind.address.management=<ip address of domain controller>
		
So now we can start the domain controller with this command:

		domain.bat --host-config=host-master.xml -Djboss.bind.address.management=<ip address>
		
Host Controller Setup
---------------------
Every host server which you want to be part of the cluster must have the host.xml file configured. In this
example you can use the host-slave.xml file since the configuration is done already.

First, to make sure the domain controller and host controller can communicate, we need a management user.
On the domain controller, run the add-user script. You will get a prompt similar to this:

	 What type of user do you wish to add?   
	  a) Management User (mgmt-users.properties)   
	  b) Application User (application-users.properties)   
	 (a): a   
	 Enter the details of the new user to add.   
	 Using realm 'ManagementRealm' as discovered from the existing property files.   
	 Username : mgmt   
	 Password recommendations are listed below. To modify these restrictions edit the add-user.properties configuration file.   
	  - The password should not be one of the following restricted values {root, admin, administrator}   
	  - The password should contain at least 8 characters, 1 alphabetic character(s), 1 digit(s), 1 non-alphanumeric symbol(s)   
	  - The password should be different from the username   
	 Password :   
	 Re-enter Password :   
	 What groups do you want this user to belong to? (Please enter a comma separated list, or leave blank for none)[ ]:   
	 About to add user 'mgmt' for realm 'ManagementRealm'   
	 Is this correct yes/no? yes   
	 Added user 'mgmt' to file '/opt/wildfly/wildfly-8.2.0.Final/standalone/configuration/mgmt-users.properties'   
	 Added user 'mgmt' to file '/opt/wildfly/wildfly-8.2.0.Final/domain/configuration/mgmt-users.properties'   
	 Added user 'mgmt' with groups to file '/opt/wildfly/wildfly-8.2.0.Final/standalone/configuration/mgmt-groups.properties'   
	 Added user 'mgmt' with groups to file '/opt/wildfly/wildfly-8.2.0.Final/domain/configuration/mgmt-groups.properties'   
	 Is this new user going to be used for one AS process to connect to another AS process?   
	 e.g. for a slave host controller connecting to the master or for a Remoting connection for server to server EJB calls.   
	 yes/no? yes   
	 To represent the user add the following to the server-identities definition <secret value="bWdtdDEyMyE=" />

Once you have the secret value, you can add it to the host.xml file. You can modify the host-slave.xml instead since much
of the configuration is done there:
		
		<?xml version='1.0' encoding='UTF-8'?>   
		<host xmlns="urn:jboss:domain:2.2">   
			<management>   
			  <security-realms>   
				<security-realm name="ManagementRealm">   
				  <server-identities>   
					 <!-- Replace this with either a base64 password of your own, or use a vault with a vault expression -->   
					 <secret value="bWdtdDEyMyE="/>   
				  </server-identities> 
				  
Another thing is to choose a unique name for each host in our domain to avoid name conflicts. Otherwise, the default is the host name of the server.

		<host name="server1" xmlns="urn:jboss:domain:1.3">
			...
		</host>

The host controller needs to know how to establish a connection to the domain controller. The following configuration on the host specifies where the domain controller is located. 
Thus, the host controller can register to the domain controller itself. 

		<host name="server1" xmlns="urn:jboss:domain:1.3">
			...
			<domain-controller>
			   <remote host="${jboss.domain.master.address}" port="${jboss.domain.master.port:9999}" security-realm="ManagementRealm"/>
			</domain-controller>
			...
		</host>

Once we've sorted the communication out, we need to tell the host controller to actually start some server instances!

At the bottom of the host-slave.xml file, there are two predefined servers to use:

		<servers>  
		  <server name="server-one" group="main-server-group"/>  
		  <server name="server-two" group="other-server-group">  
			<!-- server-two avoids port conflicts by incrementing the ports in  
			   the default socket-group declared in the server-group -->  
			<socket-bindings port-offset="150"/>   
		  </server>   
		</servers>

Note that the second server has a port offset. This means that With the port offset we can reuse the socket-binding group of 
the domain configuration for multiple server instances on one host.

That was the necessary configuration for a host. After you roll out these configuration on both hosts we can start 
the host controller on each host with the following command:

		domain.bat --host-config=host-slave.xml 
		   -Djboss.domain.master.address=<ip of domain controller>
		   -Djboss.bind.address=<ip of host controller> 
		   -Djboss.bind.address.management=<ip of host controller> 

To confirm the host controllers are started look in the host-controller.log file from the domain conroller.
We should see server1 is registered as a remote slave host.

		[Host Controller] 23:32:00,082 INFO [org.jboss.as.domain] (slave-request-threads - 1) JBAS010918: Registered remote slave host "server1", JBoss EAP 6.0.0.GA (AS 7.1.2.Final-redhat-1)