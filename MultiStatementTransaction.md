Multi-Statement Transactions and Upsert Examples in Dotnet

*Reference:* https://docs.aws.amazon.com/qldb/latest/developerguide/driver.best-practices.html
*Intention:* To add a multi-code tab selection and expand the Running multiple statements per transaction section.


Multi-Statement Insertions within a Transaction

```C#
    public class Person
    {
        public string firstName;
        public string lastName;
        public int age;
        public IIonValue ionPerson;

        public Person(string firstName, string lastName, int age)
        {
            this.firstName = firstName;
            this.lastName = lastName;
            this.age = age;
            
            // Create ion values
            ValueFactory valueFactory = new ValueFactory();
            ionPerson = valueFactory.NewEmptyStruct();
            ionPerson.SetField("firstName", valueFactory.NewString(firstName));
            ionPerson.SetField("lastName", valueFactory.NewString(lastName));
            ionPerson.SetField("age", valueFactory.NewInt(age));
        }
    };

    public class Vehicle
    {
        public string personId;
        public string brand;
        public string model;
        public string vin;
        public bool insured;
        public IIonValue ionVehicle;

        public Vehicle(string personId, string brand, string model, string vin, bool insured)
        {
            this.personId = personId;
            this.brand = brand;
            this.model = model;
            this.vin = vin;
            this.insured = insured;
            
            // Create ion values
            ValueFactory valueFactory = new ValueFactory();
            ionVehicle = valueFactory.NewEmptyStruct();
            ionVehicle.SetField("personId", valueFactory.NewString(personId));
            ionVehicle.SetField("brand", valueFactory.NewString(brand));
            ionVehicle.SetField("model", valueFactory.NewString(model));
            ionVehicle.SetField("vin", valueFactory.NewString(vin));
            ionVehicle.SetField("insured", valueFactory.NewBool(insured));
        }
    };

    // Multi-line insert transaction
    string documentIdJohn = "";
    Console.WriteLine("Registering a new person for auto insurance and making sure the vehicle is insured.");
    driver.Execute(txn =>
    {
        try
        {
            // Create person John
            var personJohn = new Person("John", "Doe", 32);
            
            // Registering John for a new insurance policy
            IResult resultPeople = txn.Execute("INSERT INTO People ?", personJohn.ionPerson);
            var rowInsertedPerson = resultPeople.Cast<IIonValue>().First();
            documentIdJohn = rowInsertedPerson.GetField("documentId").StringValue;
            Console.WriteLine($"Inserted John with document Id: {documentIdJohn}");

            // Create a new vehicle record for John
            var vehicleAudi = new Vehicle(documentIdJohn, "Audi", "A4", "3C4PDCBG7ET280296", true);

            // Register the vehicle under John
            IResult resultVehicle = txn.Execute("INSERT INTO Vehicles ?", vehicleAudi.ionVehicle);
            var rowInsertedVehicle = resultVehicle.Cast<IIonValue>().First();
            Console.WriteLine($"Inserted Audi with document Id {rowInsertedVehicle.GetField("documentId").StringValue}");
        }
        catch (InvalidOperationException)
        {
            Console.WriteLine("Unable to insure the vehicle due to a system error, please try again.");
            return;
        }
    });

```
    

Upserts within a Transaction

```C#
// Insures John's vehicle(s).
// An example UPSERT workflow where it will modify the insurance of any of
// John's existing vehicles or adds and insures a new vehicle owned by John.
public static void UpsertVehicles(IQldbDriver driver, Vehicle vehicle)
{
    driver.Execute(txn =>
    {
        try
        {
            // Query the Vehicles table using the vin
            IResult resultVehicles = txn.Execute(
                "SELECT brand, model, insured FROM Vehicles WHERE vin = ?", 
                vehicle.ionVehicle.GetField("vin"));

            if (resultVehicles.Count() > 0)
            {
                // Update the car's insurance status if existing
                IResult resultUpdate = txn.Execute("UPDATE Vehicles SET insured = ? WHERE vin = ?",
                    vehicle.ionVehicle.GetField("insured"), vehicle.ionVehicle.GetField("vin"));
                var rowUpdated = resultUpdate.Cast<IIonValue>().First();
                Console.WriteLine(
                        $"Updated vehicle {vehicle.brand} {vehicle.model} {vehicle.insured}, " +
                        $"document Id: {rowUpdated.GetField("documentId").StringValue}");
            }
            else
            {
                // Insert the car record if not existing
                IResult resultInsert = txn.Execute("INSERT INTO Vehicles ?", vehicle.ionVehicle);
                var rowInserted = resultInsert.Cast<IIonValue>().First();
                Console.WriteLine($"Inserted {vehicle.brand} {vehicle.model}, document Id {rowInserted.GetField("documentId").StringValue}");
            }
        }
        catch (InvalidOperationException)
        {
            Console.WriteLine("Unable to update the vehicle registration due to a system error, please try again.");
            return;
        }
    });
}

static void Main(string[] args)
{
    //...
    
    // John bought a new vehicle Rivian, so he's going 
    // to unregister the Audi and register the Rivian.

    // Update vehicle Audi for not being registered anymore 
    var vehicleAudi = new Vehicle(documentIdJohn, "Audi", "a4", "3C4PDCBG7ET280296", false);
    UpsertVehicles(driver, vehicleAudi);

    // Update vehicle Rivian as a new vehicle and registered
    var vehicleRivian = new Vehicle(documentIdJohn, "Rivian", "R1S", "4A3AE85H61E295903", true);
    UpsertVehicles(driver, vehicleRivian);   
}
```
