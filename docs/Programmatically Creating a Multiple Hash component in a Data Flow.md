The following is a bit of "sample" code that can be used to create an SQL 2008 SSIS package that reads from master.sys.databases, and then passes each row into Multiple Hash, and generates a single hash from the entire input row.  There is no additional components to do anything with the output from the Multiple Hash component shown in the example.

For an SQL2012 sample please download the attached sample [CreateMHashPackage.zip](Programmatically Creating a Multiple Hash component in a Data Flow_CreateMHashPackage.zip).
This sample is an improvement on the one shown below, as it utilises the classes available within Multiple Hash to ensure that the data saved with the component is correct.

You will need to add references to:
Microsoft.SqlServer.DTSPipelineWrap
Microsoft.SQLServer.DTSRuntimeWrap
Microsoft.SQLServer.ManagedDTS
in your project.

{code:c#}
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using Microsoft.SqlServer.Dts.Runtime;
using Microsoft.SqlServer.Dts.Pipeline;
using Microsoft.SqlServer.Dts.Pipeline.Wrapper;

namespace GenerateSSISPackage
{
    class Program
    {
        static void Main(string[] args)
        {
            String lineageIDs = string.Empty;
            Package package;
            Application app = new Application();
            package = new Package();

            // Add a data flow task to the package.
            Executable dataFlowTask = package.Executables.Add("STOCK:PipelineTask");
            TaskHost taskHost = dataFlowTask as TaskHost;

            // Name the task
            taskHost.Name = "Data Flow Task";
            // We need a reference to the InnerObject to add items to the data flow
            MainPipe pipeline = taskHost.InnerObject as MainPipe;

            // Create a OLEDB connection to the master database (on an instance called SQL2008R2)
            ConnectionManager connection = package.Connections.Add("OLEDB");
            connection.Name = "Source Connection";
            connection.ConnectionString = @"Data Source=.\SQL2008R2;Initial Catalog=master;Provider=SQLNCLI10.1;Integrated Security=SSPI;Auto Translate=False;";

            // Provide the required properties for the OLEDB connection
            IDTSComponentMetaData100 srcComponent = pipeline.ComponentMetaDataCollection.New();
            srcComponent.ComponentClassID = "DTSAdapter.OleDbSource";
            srcComponent.ValidateExternalMetadata = true;
            IDTSDesigntimeComponent100 srcDesignTimeComponent = srcComponent.Instantiate();
            srcDesignTimeComponent.ProvideComponentProperties();
            srcComponent.Name = "OleDb Source";

            // Configure it to read from sys.databases
            srcDesignTimeComponent.SetComponentProperty("AccessMode", 0);
            srcDesignTimeComponent.SetComponentProperty("OpenRowset", "[sys].[databases]");

            // Set the connection manager
            srcComponent.RuntimeConnectionCollection[0].ConnectionManager = DtsConvert.GetExtendedInterface(connection);
            srcComponent.RuntimeConnectionCollection[0].ConnectionManagerID = connection.ID;

            // Retrieve the column metadata
            srcDesignTimeComponent.AcquireConnections(null);
            srcDesignTimeComponent.ReinitializeMetaData();
            srcDesignTimeComponent.ReleaseConnections();


            //
            // Add the Multiple Hash
            //

            // If you want to see all the components on your machine, uncomment the following code.  The CreationName is what you place into the PipelineComponentInfos
            //foreach (PipelineComponentInfo pci in app.PipelineComponentInfos)
            //{
            //    Console.WriteLine(String.Format("{0}, {1}, {2}", pci.CreationName, pci.Description, pci.FileName));
            //}

            // Create a new object in the Pipeline
            IDTSComponentMetaData100 metaDataMultipleHash = pipeline.ComponentMetaDataCollection.New();
            // Name it
            metaDataMultipleHash.Name = "Multiple Hash Test";
            // Assign it to the Multiple Hash component.
            metaDataMultipleHash.ComponentClassID = app.PipelineComponentInfos["Martin.SQLServer.Dts.MultipleHash, MultipleHash2008, Version=1.0.0.0, Culture=neutral, PublicKeyToken=51c551904274ab44"].CreationName;
            // Instantiate the component, so that we can configure it.
            CManagedComponentWrapper instance = metaDataMultipleHash.Instantiate();

            // Initialize the component
            instance.ProvideComponentProperties();
            instance.ReinitializeMetaData();

            // Set suitable defaults
            metaDataMultipleHash.CustomPropertyCollection["MultipleThreads"].Value = 1; // Auto
            metaDataMultipleHash.CustomPropertyCollection["SafeNullHandling"].Value = 1; // True

            // Create a path from the source component to the new Multiple Hash component
            IDTSPath100 path = pipeline.PathCollection.New();
            path.AttachPathAndPropagateNotifications(srcComponent.OutputCollection[0], metaDataMultipleHash.InputCollection[0]);

            // Select which columns are to be hashed

            // Grab the input from the Multiple Hash component
            IDTSInput100 destInput = metaDataMultipleHash.InputCollection[0];
            IDTSVirtualInput100 destVirInput = destInput.GetVirtualInput();

            // This just selects all the columns!
            // If you were to be doing named columns, then you would find them in the collection.
            // And you would only SetUsageType to those specific columns.
            // Iterate through the virtual input column collection.
            foreach (IDTSVirtualInputColumn100 vColumn in destVirInput.VirtualInputColumnCollection)
            {
                // Call the SetUsageType method of the destination
                //  to add each available virtual input column as an input column.
                instance.SetUsageType(
                   destInput.ID, destVirInput, vColumn.LineageID, DTSUsageType.UT_READONLY);

                // As we are only going to do a single output in this example, just string all the input's ID's into the lineage string...
                if (string.IsNullOrEmpty(lineageIDs))
                    lineageIDs = "#" + vColumn.LineageID.ToString();
                else
                    lineageIDs += ",#" + vColumn.LineageID.ToString();
            }


            // Create a new output column
            IDTSOutputColumn100 outputColumn = instance.InsertOutputColumnAt(metaDataMultipleHash.OutputCollection[0].ID, 0, "OutputCol1", "Output Column 1");
            // Set the Hash Type.  The available hashes are 1 = MD5, 2 = RipeMD160, 3 = SHA1, 4 = SHA256, 5 = SHA384, 6 = SHA512, 7 = CRC32, 8 = CRC32C, 9 = FNV1a32, 10 = FNV1a64
            outputColumn.CustomPropertyCollection["HashType"].Value = 1; // MD5

            // Set the list of lineage id's that will be hashed for this column.
            outputColumn.CustomPropertyCollection["InputColumnLineageIDs"].Value = lineageIDs;

            // You should add a destination and another Path here from Multiple Hash to the destination.
            // Details for this are http://msdn.microsoft.com/en-us/library/ms136086.aspx

            // Save the package.
            app.SaveToXml(@"E:\Temp\Test.dtsx", package, null);

        }
    }
}
{code:c#}
