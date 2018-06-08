# Buzzy for Connections On-Prem Help Guide
Basic instructions for deploying buzzy into IBM Spectrum for on-premise IBM Connections environments


### ISW PROD Spectrum CFC Console
console accessible under: `<server-ip>:8443`  
if not running:  
```
cd /opt/ibm-cloud-private-ce-1.2.1/
sudo systemctl stop firewalld
sudo docker run -e LICENSE=accept --net=host --rm -t -v "$(pwd)/cluster":/installer/cluster ibmcom/cfc-installer:1.2.0 install
```


### MongoDB
install mongodb: https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/  
enable on startup  
`sudo systemctl enable mongod`  
secure with user credentials:  

1) Start MongoDB without access control.  
```
sudo systemctl start mongod
```
2) Connect to the instance.  
```
mongo
```
3) Create the user administrator (in the admin authentication database).  
```
use admin
db.createUser(
  {
    user: "admin",
    pwd: "xxx",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
  }
)
```  
4) Re-start the MongoDB instance with access control.  
```
sudo systemctl stop mongod
```
add:  
```
security:
    authentication: enabled
```
add:
```
net:
    bindIpAll: true
```
to /etc/mongodb.conf  

```
sudo systemctl start mongod
```
5) Connect and authenticate as the user administrator.  
```
mongo -u "admin" -p "xxx" --authenticationDatabase "admin"
```
6) Create db "buzzy-db" and add user "buzzyuser"  
```
use buzzy-db
db.createUser(
  {
    user: "buzzyuser",
    pwd: "xxx",
    roles: [ "readWrite", "dbAdmin" ]
  }
)
```
7) Test to connect and authenticate as buzzyuser.  
```
mongo -u "buzzyuser" -p "xxx" --authenticationDatabase "buzzy-db"
```


### Pre-Requisites
1. IBM Spectrum CFC is installed and running
2. WebSphere environment with Web Server
3. [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) is installed
4. mongodb is installed and db and user is configured


### Login to kubectl
1. Open Spectrum Console
2. Go to `Admin` (top right)
3. Click `Config Client`
4. Copy the contents shown
5. Open your command line (cmd)
6. Paste the commands copied earlier and press enter


### Deploy Buzzy into CFC
1. create namespace   
`kubectl create namespace buzzy`
2. create secret (docker)   
`kubectl create secret docker-registry dockerhub --docker-server=docker.io --docker-username=<REPLACE> --docker-password=<REPLACE> --docker-email=<REPLACE> --namespace=buzzy`   
3. edit buzzy.yml and enter details at the following lines:	 
* 50: URL you are deploying Buzzy to  
* 54: Enter your mongoDB credentials  
* 56: Enter your mongoDB debug credentials  
* 58: Enter your SMTP details  
* 68-74: Your individual AWS details for file storage  
* 179: More AWS details for files  
* 301: Under Buzzy Custom are lots of settings to change to make Buzzy your own  
  * 302: Company Name  
  * 305: URL of your main logo  
  * 306: URL of us in Email  
  * 309: Email Footer  
  * 312: Splash image  
  * 313: Welcome Image
		
4. create your services   
`kubectl apply -f buzzy.yml`   
service has NodePort (ie 30289 which maps to 30buz on keypads)   
accessible on `<server-ip>:30289`
5. add new DNS record for Buzzy URL   
ie `buzzy.isw.net.au`
6. Open WebSphere ISC -> open web server -> edit `http.conf` -> add another VirtualHost:   

```
<VirtualHost *:443>
	ServerName buzzy.isw.net.au

	#Buzzy On-prem
	ProxyPreserveHost On
	ProxyPass / http://<server-ip>:30289/
	ProxyPassReverse / http://<server-ip>:30289/
	#End Buzzy On-prem

	SSLEnable
    # Disable SSLv2
    SSLProtocolDisable SSLv2
    # Set strong ciphers
    SSLCipherSpec TLS_RSA_WITH_AES_128_CBC_SHA
    SSLCipherSpec TLS_RSA_WITH_AES_256_CBC_SHA
    SSLCipherSpec SSL_RSA_WITH_3DES_EDE_CBC_SHA

</VirtualHost>
```

7. Setup Community Widget

	Check out the widgets-config.xml file.
	
		execfile("profilesAdmin.py")
		ProfilesConfigService.checkOutWidgetConfig("/LCCheckedOut", AdminControl.getCell())

	Edit the widgets-config.xml file. Under the <resource type="community"> section, then under <widgets>, then within <definitions> add the following.
	
		```
		<!-- BUZZY -->
		<widgetDef defId="Buzzy" modes="view fullpage" url="https://<BUZZY_URL>/assets/connections/communityWidget.xml" themes="wpthemeNarrow wpthemeWide wpthemeBanner" uniqueInstance="true">
			<itemSet>
				<item name="resourceId" value="{resourceId}"/>
			</itemSet>
		</widgetDef>
		<!-- END BUZZY -->
		```
		
	Check in the widgets-config.xml file.	
		
		```
		ProfilesConfigService.checkInWidgetConfig()
		```
		
8. Setup Activity Stream widget

	Access Homepage->Administrator

	Select the following:
		• Open Social Gadget
		• Trusted and Use SSO
		• Show for Activity Stream events
		• All servers
	Click the Add Mapping button.

	Add a Mapping for the Buzzy service to the Kudos client. Ensure OAuth Client is set to conn-ee and the Service name is Buzzy.
	Click the Ok button

	Enter a Widget Title and Description.
		For example:
			Buzzy Activity Stream
		For the URL Address and SECURE URL Address you must enter:
			http://<BUZZY_URL>/assets/connections/gadget.xml
			https://<BUZZY_URL>/assets/connections/gadget.xml
		For the ICON URL and ICON SECURE URL you must enter:
			http://<BUZZY_URL>/assets/ico/favicon-32x32.png
			https://<BUZZY_URL>/assets/ico/favicon-32x32.png
	Select:
		• Use IBM Connections specific tags
		• Opened by default

	Select the following Prerequisites:
		• oauthprovider
		• webresources
		• oauth
		• opensocial
	Scroll down and Click Save		
		
9. Register Widgets (IBM Connections 6.0 CR1 onwards)

	```
	execfile("newsAdmin.py")
	
	NewsWidgetCatalogService.addWidget(title="Buzzy", url="http://<BUZZY_URL>/assets/connections/communityWidget.xml" ,secureUrl="https://<BUZZY_URL>/assets/connections/communityWidget.xml", categoryName=WidgetCategories.NONE, isHomepageSpecific=0, isDefaultOpened=0, multipleInstanceAllowed=0, isGadget=0, policyFlags=[GadgetPolicyFlags.TRUSTED], prereqs=['communities'], appContexts=["IWIDGETS"])
	NewsWidgetCatalogService.enableWidget("<ID_RETURNED>")
	
	NewsWidgetCatalogService.clearWidgetCaches()
	```

### Notes on IBM Spectrum CFC Naming
- Application (aka Deployments)
- PetSet (aka Replica sets)
