Multi-Statement Transactions and Upsert Examples in NodeJs-TypeScript

*Reference:* https://docs.aws.amazon.com/qldb/latest/developerguide/driver.best-practices.html
*Intention:* To add a multi-code tab selection and expand the Running multiple statements per transaction section.

Multi-Statement Insertions within a Transaction

```typescript
    type Person = Required<{
        lastName: string;
        firstName: string;
        age: number;
    }>;

    type Vehicle = Required<{
        personId: string;
        brand: string;
        model: string;
        vin: string;
        insured: boolean;
    }>;

    console.log("Registering a new person for auto insurance and making sure the vehicle is insured.");

    var documentIdJohn: string = "";
    await driver.executeLambda(async (txn: TransactionExecutor) => {
        // Create person John
        const personJohn: Person = {
            lastName: "Doe", firstName: "John", age: 32
        }
        // Registering John for a new insurance policy
        const results: dom.Value[] = (await txn.execute("INSERT INTO People ?", personJohn)).getResultList();
        if (results.length == 1) {
            documentIdJohn = results.pop().get('documentId').stringValue();
            console.log("John's documentId: %s", documentIdJohn);
        }
            
        // Create a new vehicle record for John
        const vehicleAudi: Vehicle = {
            personId: documentIdJohn, brand: "Audi", model: "A4", vin: "3C4PDCBG7ET280296", insured: true
        };
        // Register the vehicle under John
        const resultsVehicle: dom.Value[] = (await txn.execute("INSERT INTO Vehicles ?", vehicleAudi)).getResultList();
        if (resultsVehicle.length == 1) {
            const documentIdVehicle: string = resultsVehicle.pop().get('documentId').stringValue();
            console.log("Inserted Audi with document Id %s", documentIdVehicle);
        }
    });
```

Upserts within a Transaction

```typescript
// Insures John's vehicle(s).
// An example UPSERT workflow where it will modify the insurance of any of
// John's existing vehicles or adds and insures a new vehicle owned by John.
async function upsertVehicle(driver: QldbDriver, vehicle: Vehicle): Promise<void> {
    await driver.executeLambda(async (txn: TransactionExecutor) => {
        const results: dom.Value[] = (await txn.execute(
            "SELECT brand, model, insured FROM Vehicles WHERE vin = ?", vehicle.vin)).getResultList();
        if (results.length > 0) {
            // Update the car's insurance status if existing
            const resultUpdate: dom.Value[] = (await txn.execute(
                "UPDATE Vehicles SET insured = ? WHERE vin = ?", vehicle.insured, vehicle.vin)).getResultList();
            if (resultUpdate.length == 1) {
                const documentIdUpdate: string = resultUpdate.pop().get('documentId').stringValue();
                console.log("Updated vehicle %s %s with document Id %s",
                    vehicle.brand, vehicle.model, documentIdUpdate);
            }
        } else {
            // Insert the car record if not existing
            const resultInsert: dom.Value[] = (await txn.execute(
                "INSERT INTO Vehicles ?", vehicle)).getResultList();
            if (resultInsert.length == 1) {
                const documentIdInsert: string = resultInsert.pop().get('documentId').stringValue();
                console.log("Inserted vehicle %s %s with document Id %s",
                    vehicle.brand, vehicle.model, documentIdInsert);
            }
        }
    });
};

async function main(): Promise<void> {
    //...
    
    // John bought a new vehicle Rivian, so he's going 
    // to unregister the Audi and register the Rivian.

    // Update vehicle Audi for not being registered anymore
    const vehicleAudi: Vehicle = {
        personId: documentIdJohn, brand: "Audi", model: "A4", vin: "3C4PDCBG7ET280296", insured: false
    };
    await upsertVehicle(driver, vehicleAudi);

    // Insert vehicle Rivian as a new vehicle and registered
    const vehicleRivian: Vehicle = {
        personId: documentIdJohn, brand: "Rivian", model: "R1S", vin: "4A3AE85H61E295903", insured: true
    };
    await upsertVehicle(driver, vehicleRivian);

    //...
};
```
