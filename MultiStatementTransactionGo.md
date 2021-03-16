Multi-Statement Transactions and Upsert Examples in Go

*Reference:* https://docs.aws.amazon.com/qldb/latest/developerguide/driver.best-practices.html
*Intention:* To add a multi-code tab selection and expand the Running multiple statements per transaction section.


Multi-Statement Insertions within a Transaction

```Go
type Person struct {
	FirstName string `ion:"firstName"`
	LastName  string `ion:"lastName"`
	Age       int    `ion:"age"`
}

type Vehicle struct {
	Brand string `ion:"brand"`
	Model string `ion:"model"`
	Vin string `ion:"vin"`
	Insured bool `ion:"insured"`
	personId string `ion:"personId"`
}
...
    // Multi-line insert transaction
	var documentId string
	fmt.Println("Registering a new person for auto insurance and making sure the vehicle is insured.")
    _, err = driver.Execute(context.Background(), func(txn qldbdriver.Transaction) (interface{}, error) {

		// Create person John
		personJohn := Person{"John", "Doe", 32}
		// Registering John for a new insurance policy
		resultInsert, err := txn.Execute("INSERT INTO People ?", personJohn)
		if err != nil {
			panic(err)
		}
		// Get the document Id of record John Doe
		var decodedResult map[string]interface{}
		for resultInsert.Next(txn) {
			ionBinary := resultInsert.GetCurrentData()
			err = ion.Unmarshal(ionBinary, &decodedResult)
			if err != nil {
				return nil, err
			}
		}
		documentId = decodedResult["documentId"].(string)
		fmt.Println("Inserted John with document Id: %s", documentId)

		// Create a new vehicle record for John
		vehicleAudi := Vehicle{"Audi", "A4", "3C4PDCBG7ET280296", true, documentId}
        // Register the vehicle under John
		_, err =  txn.Execute("INSERT INTO Vehicles ?", vehicleAudi)
		if err != nil {
			panic(err)
		}

		return nil, nil
	})
	if err != nil {
		panic(err)
	}
```
    

Upserts within a Transaction

```Go
// Insures John's vehicle(s).
// An example UPSERT workflow where it will modify the insurance of any of
// John's existing vehicles or adds and insures a new vehicle owned by John.
func upsert(driver *qldbdriver.QLDBDriver, vehicle *Vehicle) error {
	// Multi update statements
	_, err := driver.Execute(context.Background(), func(txn qldbdriver.Transaction) (interface{}, error) {

		// Query the Vehicles table using the vin
		result, err := txn.Execute("SELECT brand, model, vin FROM Vehicles WHERE vin = ?", vehicle.Vin)
		if err != nil {
			panic(err)
		}
		hasNext := result.Next(txn)
		if !hasNext && result.Err() != nil {
			return nil, result.Err()
		}

		if hasNext {
			returnedVehicle := new(Vehicle)
			ionBinaryVehicle := result.GetCurrentData()
			err = ion.Unmarshal(ionBinaryVehicle, returnedVehicle)
			if err != nil {
				return nil, err
			}

			// Update the car's insurance status if existing
			_, err = txn.Execute("UPDATE Vehicles SET insured = ? WHERE vin = ? ", vehicle.Insured, vehicle.Vin)
			if err != nil {
				panic(err)
			}
			fmt.Printf("Updated vehicle %s %s %s\n", vehicle.Model, vehicle.Brand, vehicle.Vin)
		} else {
			// Insert the car record if not existing
			_, err =  txn.Execute("INSERT INTO Vehicles ?", vehicle)
			if err != nil {
				panic(err)
			}
			fmt.Printf("Inserted vehicle %s %s %s\n", vehicle.Model, vehicle.Brand, vehicle.Vin)
		}
		return nil, nil
	})
	if err != nil {
		panic(err)
	}
	return err
}

func main() {
    //...
    
    // John bought a new vehicle Rivian, so he's going 
    // to unregister the Audi and register the Rivian.

    // Update vehicle Audi for not being registered anymore 
    vehicleAudi := Vehicle{"Audi", "Q5", "3C4PDCBG7ET280296", false, documentId}
	upsert(driver, &vehicleAudi)

    // Update vehicle Rivian as a new vehicle and registered
    vehicleRivian := Vehicle{"Rivian", "R1S", "4A3AE85H61E295903", true, documentId}
	upsert(driver, &vehicleRivian) 
}
```
