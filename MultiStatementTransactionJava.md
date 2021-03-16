Multi-Statement Transactions and Upsert Examples in Java

*Reference:* https://docs.aws.amazon.com/qldb/latest/developerguide/driver.best-practices.html
*Intention:* To add a multi-code tab selection and expand the Running multiple statements per transaction section.


Multi-Statement Insertions within a Transaction

```Java
    public static class Person {
        String firstName;
        String lastName;
        int age;
        IonStruct ionPerson;

        public Person(String firstName, String lastName, int age) {
            this.firstName = firstName;
            this.lastName = lastName;
            this.age = age;

            ionPerson = ionSys.newEmptyStruct();
            ionPerson.put("firstName").newString(firstName);
            ionPerson.put("lastName").newString(lastName);
            ionPerson.put("age").newInt(age);
        }
    }

    public static class Vehicle {
        String personId;
        String brand;
        String model;
        String vin;
        Boolean insured;
        IonStruct ionVehicle;

        public Vehicle(String personId, String brand, String model, String vin, Boolean insured) {
            this.personId = personId;
            this.brand = brand;
            this.model = model;
            this.vin = vin;
            this.insured = insured;

            ionVehicle = ionSys.newEmptyStruct();
            ionVehicle.put("brand").newString(brand);
            ionVehicle.put("model").newString(model);
            ionVehicle.put("vin").newString(vin);
            ionVehicle.put("insured").newBool(insured);
            ionVehicle.put("personId").newString(personId);
        }
    }

        // Multi-line insert transaction
        final String[] documentIdJohn = {""};
        qldbDriver.execute(txn -> {
            System.out.println(
                    "Registering a new person for auto insurance and making sure the vehicle is insured.");

            // Create person John
            Person personJohn = new Person("John", "Doe", 32);

            // Registering John for a new insurance policy
            Result result = txn.execute("INSERT INTO Person ?", personJohn.ionPerson);
            IonStruct ionPersonJohn = (IonStruct) result.iterator().next();
            IonString ionDocumentIdJohn = (IonString) ionPersonJohn.get("documentId");
            documentIdJohn[0] = ionDocumentIdJohn.stringValue();
            System.out.println("Person's documentId = " + documentIdJohn[0]);

            // Create a new vehicle record for John
            Vehicle vehicleAudi = new Vehicle(documentIdJohn[0],
                    "Audi", "A4", "3C4PDCBG7ET280296", true);

            // Register the vehicle under John
            Result resultVehicle = txn.execute("INSERT INTO Vehicles ?", vehicleAudi.ionVehicle);
            IonStruct ionVehicle = (IonStruct) resultVehicle.iterator().next();
            IonString ionDocumentIdVehicle = (IonString) ionVehicle.get("documentId");
            String documentIdVehicle = ionDocumentIdVehicle.stringValue();
            System.out.println("Inserted Audi with document Id " + documentIdVehicle);
        });

```
    

Upserts within a Transaction

```Java
// Insures John's vehicle(s).
// An example UPSERT workflow where it will modify the insurance of any of
// John's existing vehicles or adds and insures a new vehicle owned by John.
public static void upsertVehicle(QldbDriver qldbDriver, Vehicle vehicle) {
    qldbDriver.execute(txn -> {
        // Query the Vehicles table using the vin
        Result resultVehicle = txn.execute(
                "SELECT brand, model, insured FROM Vehicles WHERE vin = ?",
                vehicle.ionVehicle.get("vin"));

        if (resultVehicle.iterator().hasNext()) {
            // Update the car's insurance status if existing
            Result resultUpdate = txn.execute(
                    "UPDATE Vehicles SET insured = ? WHERE vin = ?",
                    vehicle.ionVehicle.get("insured"), vehicle.ionVehicle.get("vin"));

            IonStruct ionVehicleUpdated = (IonStruct) resultUpdate.iterator().next();
            IonString ionDocumentIdVehicle = (IonString) ionVehicleUpdated.get("documentId");
            System.out.format("Updated vehicle %s %s with document Id '%s' %n",
                    vehicle.brand, vehicle.model, ionDocumentIdVehicle.stringValue());
        } else {
            // Insert the car record if not existing
            Result resultInsert = txn.execute(
                    "INSERT INTO Vehicles ?", vehicle.ionVehicle);

            IonStruct ionVehicleInserted = (IonStruct) resultInsert.iterator().next();
            IonString ionDocumentIdVehicle = (IonString) ionVehicleInserted.get("documentId");
            System.out.format("Inserted vehicle %s %s with document Id '%s' %n",
                    vehicle.brand, vehicle.model, ionDocumentIdVehicle.stringValue());
        }
    });
}

public class Main {
    //...
    
    // John bought a new vehicle Rivian, so he's going 
    // to unregister the Audi and register the Rivian.

    // Update vehicle Audi for not being registered anymore
    Vehicle vehicleAudi = new Vehicle(documentIdJohn[0], "Audi", "A4", "3C4PDCBG7ET280296", false);
    upsertVehicle(qldbDriver, vehicleAudi);

    // Update vehicle Rivian as a new vehicle and registered
    Vehicle vehicleRivian = new Vehicle(documentIdJohn[0], "Rivian", "R1S", "4A3AE85H61E295903", true);
    upsertVehicle(qldbDriver, vehicleRivian);
}
```
