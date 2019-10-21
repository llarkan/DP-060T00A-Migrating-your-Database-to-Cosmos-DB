- [Lab 2: Migrate MongDB Workloads to Cosmos DB](#lab-2-migrate-mongdb-workloads-to-cosmos-db)
  - [Exercise 1: Setup](#exercise-1-setup)
    - [Task 1: Create a Resource Group and Virtual Network](#task-1-create-a-resource-group-and-virtual-network)
    - [Task 2: Create a MongoDB Database Server](#task-2-create-a-mongodb-database-server)
    - [Task 3: Configure the MongoDB Database](#task-3-configure-the-mongodb-database)
  - [Exercise 2: Populate and Query the MongoDB Database](#exercise-2-populate-and-query-the-mongodb-database)
    - [Task 1: Build and Run an App to Populate the MongoDB Database](#task-1-build-and-run-an-app-to-populate-the-mongodb-database)
    - [Task 2: Build and Run Another App to Query the MongoDB Database](#task-2-build-and-run-another-app-to-query-the-mongodb-database)
  - [Exercise 3: Migrate the MongoDB Database to Cosmos DB](#exercise-3-migrate-the-mongodb-database-to-cosmos-db)
    - [Task 1: Create a Cosmos Account and Database](#task-1-create-a-cosmos-account-and-database)
    - [Task 2: Create the Database Migration Service](#task-2-create-the-database-migration-service)
    - [Task 3: Create and Run a New Migration Project](#task-3-create-and-run-a-new-migration-project)
    - [Task 4: Verify that Migration was Successful](#task-4-verify-that-migration-was-successful)
  - [Exercise 4: Reconfigure and Run Existing Applications to Use Cosmos DB](#exercise-4-reconfigure-and-run-existing-applications-to-use-cosmos-db)
    - [Task 1: Enable MongoDB Aggregation Support](#task-1-enable-mongodb-aggregation-support)
    - [Task 2: Reconfigure the DeviceDataQuery Application](#task-2-reconfigure-the-devicedataquery-application)
  - [Exercise 5: Clean Up](#exercise-5-clean-up)

# Lab 2: Migrate MongDB Workloads to Cosmos DB

In this lab, you'll take an existing MongoDB database and migrate it to Cosmos DB. You'll use the Azure Database Migration Service. You'll also see how to reconfigure existing applications that use the MongoDB database to connect to the Cosmos DB database instead.

The lab is based around an example system that captures temperature data from a series of IoT devices. The temperatures are logged in a MongoDB database, together with a timestamp. Each device has a unique ID. You will run a MongoDB application that simulates these devices, and stores the data in the database. You will also use a second application that enables a user to query statistical information about each device. After migrating the database from MongoDB to Cosmos DB, you'll configure both applications to connect to Cosmos DB, and verify that they still function correctly.

The lab runs using the Azure Cloud shell and the Azure portal.

## Exercise 1: Setup

In the first exercise, you'll create the MongoDB database for holding the data captured form the temperature devices.

### Task 1: Create a Resource Group and Virtual Network

1. In your Internet browser, navigate to https://portal.azure.com and sign in.
2. In the Azure portal, click **Resource groups**, and then click **+Add**.
3. On the **Create a resource group page**, enter the following details, and then click **Review + Create**:

    | Property  | Value  |
    |---|---|
    | Subscription | *\<your-subscription\>* |
    | Resource Group | mongodbrg |
    | Region | Select your nearest location |

4. Click **Create**, and wait for the resource group to be created.
5. In the left-hand pane of the Azure portal, click **+ Create a resource**.
6. On the **New** page, in the **Search the Marketplace** box, type **Virtual Network**, and press Enter.
7. On the **Virtual Network** page, click **Create**.
8. On the **Create virtual network** page, enter the following details, and then click **Create**:

    | Property  | Value  |
    |---|---|
    | Name | databasevnet |
    | Address space | 10.0.0.0/24 |
    | Subscription | *\<your-subscription\>* |
    | Resource Group | mongodbrg |
    | Region | Select the same location that you specified for the resource group |
    | Subnet Name | default |
    | Subnet Address range | 10.0.0.0/28 |
    | DDos protection | Basic |
    | Service endpoints | Disabled |
    | Firewall | Disabled |

9. Wait for the virtual network to be created before continuing.

### Task 2: Create a MongoDB Database Server

1. In the left-hand pane of the Azure portal, click **+ Create a resource**.
2. In the **Search the Marketplace** box, type ***MongoDB Certified by Bitnami**, and then press Enter.
3. On the **MongoDB Certified by Bitnami** page, click **Create**.
4. On the **Create a virtual machine** page, enter the following details, and then click **Next: Disks \>**.

    | Property  | Value  |
    |---|---|
    | Subscription | *\<your-subscription\>* |
    | Resource Group | mongodbrg |
    | Virtual machine name | mongodbserver | 
    | Region | Select the same location that you specified for the resource group |
    | Availability options | No infrastructure redundancy required |
    | Image | MongoDB Certified by Bitnami |
    | Size | Standard A1 v2 |
    | Authentication type | Password |
    | Username | azureuser |
    | Password | Pa55w.rdPa55w.rd |
    | Confirm password | Pa55w.rdPa55w.rd |

5. On the **Disks** page, accept the default settings, and then click **Next: Networking \>**.
6. On the **Networking** page, enter the following details, and then click **Next: Management \>**.

    | Property  | Value  |
    |---|---|
    | Virtual network | databasevnet |
    | Subnet | default (10.0.0.0/28) |
    | Public IP | (new) mongodbserver-ip |
    | NIC network security group | Advanced |
    ! Configure network security group | (new) mongodbserver-nsg |
    | Accelerated networking | Off |
    | Load balancing | No |

7. On the **Management** page, accept the default settings, and then click **Next: Advanced \>**.
8. On the **Advanced** page, accept the default settings, and then click **Next: Tags \>**.
9. On the **Tags** page, accept the default settings, and then click **Next: Review + create \>**.
10. On the validation page, click **Create**.
11. Wait for the virtual machine to be deployed before continuing
12. In the left-hand pane of the Azure portal, click **All resources**.
13. On the **All resources** page, click **mongodbserver-nsg**.
14. On the **mongodbserver-nsg** page, under **Settings**, click **Inbound security rules**.
15. On the **mongodbserver-nsg - Inbound security rules** page, click **+ Add**.
16. In the **Add inbound security rule** pane, enter the following details, and then click **Add**:

    | Property  | Value  |
    |---|---|
    | Source | Any |
    | Source port ranges | * |
    | Destination | Any |
    | Destination port ranges | 27017 |
    | Protocol | Any |
    | Action | Allow |
    | Priority | 1020 |
    | Name | Mongodb-port |
    | Description | Port that clients use to connect to MongoDB |

### Task 3: Configure the MongoDB Database

1. In the left-hand pane of the Azure portal, click **All resources**.
2. On the **All resources** page, click **mongodbserver-ip**.
3. On the **mongodbserver-ip** page, make a note of the **IP address**.
4. In the toolbar at the top of the Azure portal, click **Cloud Shell**.
5. If the **You have no storage mounted** message box appears, click **Create storage**.
6. When the Cloud Shell starts, in the drop-down list above the Cloud Shell window, select **Bash**.
7. In the Cloud Shell, enter the following command to connect to the mongodbserver virtual machine. Replace *\<ip address\>* with the value of the **mongodbserver-ip** IP address:

    ```bash
    ssh azureuser@<ip address>
    ```

8. At the prompt, type **yes** to continue connecting.
9. Enter the password **Pa55w.rdPa55w.rd**
10. Type the following command, and make a note of the root password displayed:

    ```bash
    cat bitnami_credentials
    ```

11. Run the following command to connect to the MongoDB database. Replace *\<password\>* with the root password displayed in the previous step. Ignore the warning about using the XFS filesystem:

    ```bash
    mongo -u root -p <password>
    ```

12. At the **>** prompt, run the following commands. These commands create a new user named **deviceadmin** with password **Pa55w.rd** for a database named **DeviceData**. After running the `db.shutdownserver();` command you will receive some errors which you can ignore:

    ```mongosh
    use DeviceData;
    db.createUser(
        {
            user: "deviceadmin",
            pwd: "Pa55w.rd",
            roles: [ { role: "readWrite", db: "DeviceData" } ]
        }
    );
    use admin;
    db.shutdownServer();
    exit;
    ```

13. Run the following command restart the mongodb service. Verify that the service restarts without any error messages, and is listening on port 27017:

    ```bash
    sudo /opt/bitnami/ctlscript.sh start
    ```

14. Run the following command to verify that you can now log in to mongodb as the deviceadmin user:

    ```bash
    mongo -u "deviceadmin" -p "Pa55w.rd" --authenticationDatabase DeviceData
    ```

15. At the **>** prompt, run the following command to quit the mongo shell:

    ```mongosh
    exit;
    ```

16. At the **bitnami@mongodbserver** prompt, type the following command to disconnect from the MongoDB server and return to the Cloud Shell:

    ```bash
    exit
    ```

## Exercise 2: Populate and Query the MongoDB Database

You have now created a MongoDB server and database. The next step is to demonstrate the sample applications that can populate and query the data in this database.

### Task 1: Build and Run an App to Populate the MongoDB Database

1. In the Azure Cloud Shell, run the following command to download the sample code for this workshop:

    ```bash
    git clone https://github.com/MicrosoftLearning/DP-160T00A-Migrating-your-Database-to-Cosmos-DB migration-workshop-apps
    ```

2. Move to the **migration-workshop-apps/MongoDeviceDataCapture/MongoDeviceCapture** folder:

    ```bash
    cd ~/migration-workshop-apps/MongoDeviceDataCapture/MongoDeviceDataCapture
    ```

3. Use the **Code** editor to examine the **TemperatureDevice.cs** file:

    ```bash
    code TemperatureDevice.cs
    ```

    The code in this file contains a class named **TemperatureDevice** that simulates a temperatue device capturing data and saving it in a MongoDB database. It uses the MongoDB library for the .NET Framework. The  **TemperatureDevice** constructor connects to the database using settings stored in the application configuration file. The **RecordTemperatures** method generates a reading and writes it to the database.

4. Close the code editor, and then open the **ThermometerReading.cs** file:

   ```bash
   code ThermometerReading.cs
   ```

    This file shows the structure of the documents that the application stores in the database. Each document contains the following fields:

    - An object ID. The is the "_id" field generated by MongoDB to uniquely identify each document.
    - A device ID. Each device has a number with the prefix "Device ".
    - The temperature recorded by the device
    - The date and time when the temperature was recorded.
  
5. Close the code editor, and then open the **App.config** file:

    ```bash
    code App.config
    ```

    This file contains the settings for connecting to the MongoDB database. Set the value for the **Address** key to the IP address of the MongoDB server that you recorded earlier, and then save the file and close the editor.

6. Run the following command to rebuild the application:

    ```bash
    dotnet build
    ````

7. Run the application:

    ```bash
    dotnet run
    ```

    The application simulates 100 devices running simultaneously. Allow the application to run for a couple of minutes, and then press Enter to stop it.

### Task 2: Build and Run Another App to Query the MongoDB Database

1. Move to the **migration-workshop-apps/MongoDeviceDataCapture/DeviceDataQuery** folder:

    ```bash
    cd ~/migration-workshop-apps/MongoDeviceDataCapture/DeviceDataQuery
    ```

    This folder contains another application that you can use to analyze the data captured by each device.

2. Use the **Code** editor to examine the **Program.cs** file:

    ```bash
    code Program.cs
    ```

    The application connects to the database (using the **ConnectToDatabase** method at at te bottom of the file and then prompts the user for a device number. The application uses the MongoDB library for the .NET Framework to create and run an aggregate pipeline that calculates the following statistics for the specified device:

    - The number of readings recorded.
    - The average temperature recorded.
    - The lowest reading.
    - The highest reading.
    - The latest reading.

3. Close the code editor, and then open the **App.config** file:

    ```bash
    code App.config
    ```

    As before, set the value for the **Address** key to the IP address of the MongoDB server that you recorded earlier, and then save the file and close the editor.

4. Build and run the application:

    ```bash
    dotnet build
    dotnet run
    ```

5. At the **Enter Device Number** prompt, enter a value between 0 and 99. The application will query the database, calculate the statistics, and display the results. Press **Q** to quit the application.

## Exercise 3: Migrate the MongoDB Database to Cosmos DB

The next step is to take the MongoDB database and transfer it to Cosmos DB.

### Task 1: Create a Cosmos Account and Database

1. Return to the Azure portal.
2. In the left pane, click **+ Create a resource**.
3. On the **New** page, in the **Search the Marketplace** box, type ***Azure Cosmos DB**, end then press Enter.
4. On the **Azure Cosmos DB** page, click **Create**.
5. On the **Create Azure Cosmos DB Account** page, enter the following settings, and then click **Review + create**:

    | Property  | Value  |
    |---|---|
    | Subscription | Select your subscription |
    | Resource group | mongodbrg |
    | Account Name | mongodb*nnn*, where *nnn* is a random number selected by you |
    | API | Azure Cosmos DB for MongoDB API |
    | Location | Specify the same location that you used for the MongoDB server and virtual network |
    | Geo-Redundancy | Disable |
    | Multi-region Writes | Disable |

6. On the validation page, click **Create**, and wait for the Cosmos DB account to be deployed.
7. In the left-hand pane, click **Azure Cosmos DB**.
8. On the **Azure Cosmos DB** page, click your Cosmos DB account (**mongodb*nnn***).
9. On the **mongodb*nnn*** page, click **Data Explorer**.
10. In the **Data Explorer** pane, click **New Collection**.
11. In the **Add Collection** pane, specify the following settings, and then click **OK**:

    | Property  | Value  |
    |---|---|
    | Database id | Click **Create new**, and then type **DeviceData** |
    | Provision database throughput | selected |
    | Throughput | 1000 |
    | Collection id | Temperatures |
    | Shard Key | deviceID |

### Task 2: Create the Database Migration Service

1. In the left-hand pane, click **All services**.
2. On the **All services** page, click **Subscriptions**.
3. On the **Subscriptions** page, click your subscription.
4. On your subscription page, under **Settings**, click **Resource providers**.
5. In the **Filter by name** box, type **DataMigration**, and then click **Microsoft.DataMigration**.
6. Click **Register**, and wait for the **Status** to change to **Registered**. It might be necessary to click **Refresh** to see the status to change.
7. In the left-hand pane, click **+ Create a resource**.
8. On the **New** page, in the **Search the Marketplace** box, type **Azure Database Migration Service**, and then press Enter.
9. On the **Azure Database Migration Service** page, click **Create**.
10. On the **Create Migration Service** page, enter the following settings, and then click **Create**:

    | Property  | Value  |
    |---|---|
    | Service Name | MongoDBMigration |
    | Subscription | Select you subscription |
    | Select a resource group | mongodbrg |
    | Location | Select the same location that you used previously |
    | Virtual network | Click **Select or create virtual network**, select **databasevnet/default**, and then click **OK** |
    | Pricing Tier | Standard: 1 vCores |

11. Wait for the service to be deployed before continuing. This operation will take a few minutes.

### Task 3: Create and Run a New Migration Project

1. In the left-hand pane, click **Resource groups**.
2. In the **Resource groups** window, click **mongodbrg**.
3. In the **mongodbrg** window, click **MongoDBMigration**.
4. On the **MongoDBMigration** page, click **+ New Migration Project**.
5. On the **New migration project** page, enter the following settings, and the click **Create and run activity**:

    | Property  | Value  |
    |---|---|
    | Project name | MigrateTemperatureData |
    | Source server type | MongoDB |
    | Target server type | Cosmos DB (MongoDB API) |
    | Choose type of activity | Offline data migration |

6. When the **Migration Wizard** starts, on the **Source details** page, enter the following details, and then click **Save**:

    | Property  | Value  |
    |---|---|
    | Mode | Standard mode |
    | Source server name | Specify the value of the **mongodbserver-ip** IP address that you recorded earlier |
    | User Name | root |
    | Password | Enter the password of the root user on the mongodbserver VM that you noted earlier (from the bitnami_credentials file) |
    | Require SSL | Leave blank |

7. On the **Migration details details** page, enter the following details, and then click **Save**:

    | Property  | Value  |
    |---|---|
    | Mode | Select Cosmos DB target |
    | Subscription | Select your subscription |
    | Select Comos DB name | mongodb*nnn* |
    | Connection string | Accept the connection string generated for your Cosmos DB account |

8. On the **Map to target databases** page, enter the following details, and then click **Save**:

    | Property  | Value  |
    |---|---|
    | Source Database | DeviceData |
    | Target Database | DeviceData |
    | Throughput (RU/s) | 1000 |
    | Clean up collections | Clear this box |

9. On the **Collection setting** page, click the dropdown arrow by the DeviceData database, enter the following details, and then click **Save**:

    | Property  | Value  |
    |---|---|
    | Name | Temperatures |
    | Target Collection | Temperatures |
    | Throughput (RU/s) | 1000 |
    | Shard Key | deviceID |
    | Unique | Leave blank |

10. On the **Migration summary** page, in the **Activity name** field, enter **mongodb-migration**, select **Boost RU during initial data copy**, and then click **Run migration**.
11. On the **mongodb-migration** page, click **Refresh** every 30 seconds, until the migration has completed. Note the number of documents migrated.

### Task 4: Verify that Migration was Successful

1. In the left-hand pane, click **Azure Cosmos DB**.
2. On the **Azure Cosmos DB** page, click **mongodb*nnn***.
3. On the **mongodb*nnn** page, click **Data Explorer**.
4. In the **Data Explorer** pane, expand the **Temperatures** database, and then click **Documents**.
5. In the **Documents** pane, scroll through the list of documents. You should see a document id (**_id**) and the shard key (**/deviceID**) for each document.
6. Click any document. You should see the details of the document displayed. A typical document looks like this:

    ```JSON
    {
	    "_id" : ObjectId("5ce8104bf56e8a04a2d0929a"),
	    "deviceID" : "Device 83",
	    "temperature" : 19.65268837271849,
	    "time" : 636943091952553500
    }
    ```

7. In the toolbar in the **Document Explorer** pane, click **New Shell**.
8. In the **Shell 1** pane, at the **\>** prompt, enter the following command, and then press Enter:

    ```mongosh
    db.Temperatures.count()
    ```

    This command displays the number of documents in the Temperatures collection. It should match the number reported by the Migration Wizard .

9. Enter the following command, and then press Enter:

    ```mongosh
    db.Temperatures.find({deviceID: "Device 99"})
    ```

    This command fetches and displays the documents for Device 99.

## Exercise 4: Reconfigure and Run Existing Applications to Use Cosmos DB

The final step is to reconfigure your existing MongoDB applications to connect to Cosmos DB, and verify that they operate as before. This process requires you to modify the way in which your applications connect to the database, but the logic of your applications should remain unchanged.

### Task 1: Enable MongoDB Aggregation Support

1. In the **mongodb*nnn*** pane, under **Settings**, click **Preview Features**.
2. Next to **Aggregation Pipeline**, click **Enable**. Wait while the aggregation pipeline is enabled. This feature adds support for MongoDB aggregate and grouping operations to the Cosmos DB database. It may take several minutes before the aggregation pipeline becomes available. Refresh the browser window to see the latest status of the feature.

### Task 2: Reconfigure the DeviceDataQuery Application

1. In the **mongodb*nnn*** pane, under **Settings**, click **Connection String**.
2. On the **mongodb*nnn* Connection String** page, make a note of the following settings:

    - Host
    - Username
    - Primary Password
  
3. Return to the Cloud Shell window (reconnect if the session has timed out), and move to the **migration-workshop-apps/MongoDeviceDataCapture/DeviceDataQuery** folder:

    ```bash
    cd ~/migration-workshop-apps/MongoDeviceDataCapture/DeviceDataQuery
    ```

4. Open the App.config file in the Code editor:

    ```bash
    code App.config
    ```

5. In the **Settings for MongoDB** section of the file, comment out the existing settings.
6. Uncomment the settings in the **Settings for Cosmos DB Mongo API** section, and set the values for these settings as follows:

    | Setting  | Value  |
    |---|---|
    | Address | The Host from the **mongodb*nnn* Connection String** page |
    | Username | The Username from the **mongodb*nnn* Connection String** page |
    | Password | The Primary Password from the **mongodb*nnn* Connection String** page |

    The completed file should look similar to this:

    ```XML
    <?xml version="1.0" encoding="utf-8"?>
    <configuration>
        <appSettings>
            <add key="Database" value="DeviceData" />
            <add key="Collection" value="Temperatures" />

            <!-- Settings for MongoDB -->
            <!--add key="Address" value="168.63.99.236" />
            <add key="Port" value="27017" />
            <add key="Username" value="deviceadmin" />
            <add key="Password" value="Pa55w.rd" /-->
            <!-- End of settings for MongoDB -->

            <!-- Settings for CosmosDB Mongo API -->
            <add key="Address" value="mongodbnnn.documents.azure.com"/>
            <add key="Port" value="10255"/>
            <add key="Username" value="mongodbnnn"/>
            <add key="Password" value="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=="/>
            <!-- End of settings for CosmosDB Mongo API -->
        </appSettings>
    </configuration>
    ```

7. Save the file, and then close the Code editor.

8. Open the Program.cs file using the Code editor:

    ```bash
    code Program.cs
    ```

9. Scroll down to the **ConnectToDatabase** method.
10. Comment out the line that sets the credentials for connecting to MongoDB, and uncomment the statements that specify the credentials for connecting to Cosmos DB. The code should look like this:

    ```C#
    // Connect to the MongoDB database
    MongoClient client = new MongoClient(new MongoClientSettings
    {
        Server = new MongoServerAddress(address, port),
        ServerSelectionTimeout = TimeSpan.FromSeconds(10),

        //
        // Credential settings for MongoDB
        //

        // Credential = MongoCredential.CreateCredential(database, azureLogin.UserName, azureLogin.SecurePassword),

        //
        // Credential settings for CosmosDB Mongo API
        //

        UseSsl = true,
        SslSettings = new SslSettings
        {
            EnabledSslProtocols = SslProtocols.Tls12
        },
        Credential = new MongoCredential("SCRAM-SHA-1", new MongoInternalIdentity(database, azureLogin.UserName), new PasswordEvidence(azureLogin.SecurePassword))

        // End of Mongo API settings 
    });

    ```

    These changes are necessary because the original MongoDB database was not using an SSL connection. Cosmos DB always uses SSL.

11. Save the file, and then close the Code editor.
12. Rebuild and run the application:

    ```bash
    dotnet build
    dotnet run
    ```

13. At the **Enter Device Number** prompt, enter a device number between 0 and 99. The application should run exactly as before, except this time it is using the data held in the Cosmos DB database.
14. Test the application with other device numbers. Enter **Q** to finish.

You have successfully migrated a MongoDB database to Cosmos DB, and reconfigured an existing MongoDB application to connect to the Cosmos DB database.

## Exercise 5: Clean Up

1. Return to the Azure portal.
2. In the left-hand pane, click **Resource groups**.
3. In the **Resource groups** window, click **mongodbrg**.
4. Click **Delete resource group**.
5. On the **Are you sure you want to delete "mongodbrg"** page, in the **Type the resource group name** box, enter **mongodbrg**, and then click **Delete**.

---
© 2019 Microsoft Corporation. All rights reserved.

The text in this document is available under the [Creative Commons Attribution 3.0 License](https://creativecommons.org/licenses/by/3.0/legalcode), additional terms may apply. All other content contained in this document (including, without limitation, trademarks, logos, images, etc.) are **not** included within the Creative Commons license grant. This document does not provide you with any legal rights to any intellectual property in any Microsoft product. You may copy and use this document for your internal, reference purposes.

This document is provided "as-is." Information and views expressed in this document, including URL and other Internet Web site references, may change without notice. You bear the risk of using it. Some examples are for illustration only and are fictitious. No real association is intended or inferred. Microsoft makes no warranties, express or implied, with respect to the information provided here.
