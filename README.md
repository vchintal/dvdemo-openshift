# JDV Multi-Datasource Demo on Openshift 

## Setup OpenShift Environment 3.9

```sh
# Start the OpenShift cluster
oc cluster up --image registry.access.redhat.com/openshift3/ose --version v3.9.14-2

# Create a new project 
oc new-project --display-name='My Project' myproject
 
# Login to the cluster 
oc login -u <userid>
```
## Configure namespace for JDV

```sh 
# Ideally keep the service account name the same
oc create serviceaccount datavirt-service-account

# Give view access to the service account, needed for clustering
oc policy add-role-to-user view system:serviceaccount:myproject:datavirt-service-account

# Run the following commands if the keystore.jks and jgroups.jceks are not already created
export DNAME='CN=developer,O=RedHat,C=US'
export KEYPASS=mykeystorepass
export CLUSTERPASS=password
keytool -genkeypair -alias jboss -storetype JKS -storepass $KEYPASS -keypass $KEYPASS -dname $DNAME -keystore keystore.jks
keytool -genseckey -alias secret-key -storetype JCEKS -storepass $CLUSTERPASS -keypass $CLUSTERPASS -keystore jgroups.jceks

# The datavirt-app-config secret tracks all the datasources and resource adapter definitions 
# Ideally keep the secret name the same as the template references it
oc secrets new datavirt-app-config datasources.env

# keystore.jks is for SSL over JDBC and jgroups.jceks is needed for enabling SSL for JGroups clustering
oc create secret generic datavirt-app-secret --from-file=keystore.jks --from-file=jgroups.jceks

# Associate all the secrets with the serivce account 
oc secrets link datavirt-service-account datavirt-app-secret datavirt-app-config
```

### Create a MongoDB POD instance

Use the OpenShift 3.9 management UI to find/launch a MongoDB instance, provide the following as the configuration values:
* Username : dvuser
* Password : dvuser
* Admin Password : dvadmin 
* Database Name : sampledb 

### Create a MySQL POD instance

Use the OpenShift 3.9 management UI to find/launch a MongoDB instance, provide the following as the configuration values:
* Username : dvuser
* Password : dvuser
* Root Password : dvadmin 
* Database Name : sampledb 

## Add data to MySQL POD

```sh 
# Get a list of all pods and note the name of the MySQL POD
oc get pods

# Port-forward the MySQL port to your local machine on the same port
oc port-forward mysql-<id> 3306:3306

# Change directory to data folder from command line and run the 
# following command, use `dvadmin` when prompted for password 
 mysql -u root -p -h 127.0.0.1 -P 3306 < employees.sql
```
## Add data to Mongo POD

```sh 
# Get a list of all pods and note the name of the MongoDB POD
oc get pods

# Port-forward the MongoDB port to your local machine on the same port
oc port-forward mongodb-<id> 27017:27017

# Change directory to data folder from command line and run the 
# following command, use `dvadmin` when prompted for password 
mongoimport -d sampledb -c salaries --type csv --file salaries.data --headerline -u dvuser -p dvuser
```

## Create a JDV POD instance 

Use the OpenShift 3.9 management UI to find/launch a JDV 6.3 instance, provide the following as the configuration values:

* Git Repository URL : https://github.com/vchintal/dvdemo-openshift
* Context Directory : app
* Teiid Username : teiidUser
* Teiid Password : redhat1!

## Connect to JDV from outside of OpenShift 

**Step 1 : Use the keystore.jks to create a truststore for use later with JDBC client apps**

```sh 
keytool -export -alias jboss -file jdv-server.crt -keystore keystore.jks -storepass $KEYPASS
keytool -import -noprompt -trustcacerts -alias jboss -file jdv-server.crt -keystore truststore.jks -storepass $KEYPASS
```

**Step 2 : Launch your JDBC Java client with following system properties**

_Note that the keystore password is same as above in **Configure namespace for JDV** section_

```sh 
-Djavax.net.ssl.trustStore=<path-to>/truststore.jks \ 
-Djavax.net.ssl.trustStorePassword=mykeystorepass
```

**Step 3: Configure the JDBC connection to the JDV pod(s)**

Use the following specific settings for the JDBC connection
1. Port : 443
2. Host : jdbc-datavirt-app-myproject.[OpenShift Master DNS]
3. Username : teiidUser
4. Password : redhat1!
5. Enable SSL : True 

The eventual JDBC URL should look like the following :

`jdbc:teiid:dvdemo-openshift@mms://jdbc-datavirt-app-myproject.[OpenShift Master DNS]:443`

Take a note of the protocol, it should be **mms** and not **mm**.
