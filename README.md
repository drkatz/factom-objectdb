# factom-objectdb
An library for the Factom Blockchain, written in NodeJS. Exposes basic CRU(D) operations for JSON encoded objects stored on Factom.



The motivation for this library is to enable and demonstrate richer functionality on top of Factom's data-agnostic protocol. Distributed applications on Factom require a basic storage protocol to operate. `factom-objectdb` aims to achieve this by providing a framework for data validation and rules based on the cross platform JSON standard. 



## Prerequisites

You must have the following to write objects using this library:

-  A funded public or private Entry Credit address
-  Access to the `factomd-api` (and`walletd-api` if you want to use a public EC address)

The EC address must remain funded to continue creating entries! You may use testnet addresses and servers.

Reading stored objects is **free** and does not require an EC address.



## Installation

 Command Line:

```javascript
npm -i factom-objectdb
```



`package.json`

```javascript
"dependencies": {
    "factom-objectdb": "0.0.1",
}
```



# Examples

### Initialization

```javascript
var {FactomObjectDB} = require('factom-objectdb');
const ES = ;

var db = new FactomObjectDB({
    db_id: 'factomdbtest:0.0.1', //the ID of your database
    factomparams: {host: '88.200.170.90'},  //testnet courtesy node IP for example
    es_address: 'Es3k4L7La1g7CY5zVLer21H3JFkXgCBCBx8eSM2q9hLbevbuoL6a',  //testnet courtesy private EC address for example
});
```



### Create an Object

Lets say we have an object we want to store:

```javascript
//an example object, a person in a database

var joe = {
    _id: '134e366520a6f93265eb',
    name: 'Joe Testerson',
    age: 30,
    best_friends: []
};
```

We can store this on Factom and update the object over time. To do so safely, we must define some rules for the object and it's fields to adhere to.



#### Define Object & Field Rules

The library allows placing various restrictions on how the objects you store can be updated. The current state of an object is determined by the library using these rules when retrieving the object from Factom.

In this case, `Joe Testerson` is a user in a database. To facilitate that functionality, we should place some restrictions on what can be done with his data.

```javascript
var FieldRules = require('./src/rules/FieldRules');

var ObjectRules = require('./src/rules/ObjectRules');
declare object rules
//
var objectRules = new ObjectRules.Builder()
    .setAddFields(false) //disable adding fields to Joe's object
    .setDeleteFields(false) //disable deleting fields from to Joe's object
    .setRenameFields(false) //disable renaming fields in Joe's object

    //declare field rules:
    .addFieldRule('_id', new FieldRules.Builder().setType('string').setEditable(false).build()) //mark Joe's ID final, so it can never be changed
    .addFieldRule('name', new FieldRules.Builder().setType('string').setEditable(true).build()) //mark Joe's name editable so he can change it later
    .addFieldRule('age', new FieldRules.Builder().setType('number').setEditable(true).setMin(0).setMax(100).build()) //Joes age is, updatable, but will be locked to non negative number <= 100
    .addFieldRule('best_friends', new FieldRules.Builder().setType('array').setEditable(true).setMax(5).build()) //limit Joe's best friends to to 5 in count, non deletable
    .build();
```

See below for all Object and Field rules



#### Store The Object

Now that we've defined some rules we're ready to save the Object to Factom

```javascript
//commit the initial object and rules to Factom!
db.commitObject(joe._id, joe, objectRules, function (err, chain) {
    if (err) throw err;
});
```

 It is important to note that creation and updates to objects take up until the next block to be reflected (up to 10 Minutes).



### Read an Object

Get Joe's object using his id: `5ad28b9d18c35e2b4c000001`

```javascript
db.getObject("134e366520a6f93265eb", function (err, object) {
        if (err) {
            console.error(err);
            return;
        }
        console.log('Retrieved Object:\n' + JSON.stringify(object, undefined, 2));
});
```

Output:

```javascript
Retrieved Object:
{
  "_id": "134e366520a6f93265eb",
  "name": "Joe Testerson",
  "age": 25,
  "best_friends": []
}
```



### Update an Object

This library uses a [MongoDB inspired update syntax](https://docs.mongodb.com/manual/reference/operator/update/#id1). 

Currently, these operators are supported:

- `$set` : Set the value of a key in the object
- `$unset` : Delete the key from an object
- `$rename` : Rename the key of an object
- `$inc` : Increase the value of the key of an object by an amount
- `$mul` : Multiply the value of the key by an number
- `$push` : Add a value to the end of an array
- `$pop` : Remove a value from the end of an array



Updates to Factom objects are subject to the object's Field and Object rules. Updates that do not meet the restrictions placed on the Object will be ignored when retrieving it next time. It is important to note that updates to objects take up until the next block to be reflected using `getObject`. Changes can take up to 10 Minutes to be reflected in the final retrieval of the object.



Let's say Joe just had his 27th birthday. We want to `$set` his new age:

```javascript
var update = { //increase Joe's age by 1
        $set: {
            age: 27
        }
 };

db.commitObjectUpdate("134e366520a6f93265eb", update, function (err, entry) {
	if (err) throw err;
});
```



Lets say Joe just made a friend named Johan! We want to `$push` a friend to his best_friends array:

```javascript
var update = { //push a new friend to the best_friends array. Should be successful
        $push: {
            best_friends: {name: 'Yohan B', age: 30}
        }
};

db.commitObjectUpdate("134e366520a6f93265eb", update, function (err, entry) {
	if (err) throw err;
});
```



Lets say Joe fell into a black hole and has aged 70 years:

```javascript
var update = {
        $inc: {  //Increase Joe's age by 70!
            age: 70
        }
};

db.commitObjectUpdate("134e366520a6f93265eb", update, function (err, entry) {
	if (err) throw err;
    
});
```



Joe is now 97 years of age, and sadly all his friends are dead. Better get rid of Johan :(

```javascript
var update = { //pull a single friend from the best_friends array
        $pop: {
            best_friends: {}
        }
};

db.commitObjectUpdate("134e366520a6f93265eb", update, function (err, entry) {
	if (err) throw err;
});
```



Lets say Joe keeps falling for another 10 years:

```javascript
var update = {
        $inc: {  //Increase Joe's age by 10!
            age: 10
        }
};

db.commitObjectUpdate("134e366520a6f93265eb", update, function (err, entry) {
	if (err) throw err;
});
```

But now we have a problem! Increasing Joe's age by 10 would make him 107, which is over the maximum value we set for his age of 100. This update will be ignored the next time Joe's object is retrieved.



### Get An Object's Metadata

Let's say we want to get info on Joe's object, which is held in his first entry on Factom:

```javascript
db.getObjectMetadata("134e366520a6f93265eb", function (err, object) {
    if (err) throw err;
    console.log('GOT META!');
    console.log(JSON.stringify(object, undefined, 2));
});
```



The output illustrates how the library stores and defines rules for the Object:



```javascript
GOT META!
{
  "type": "meta",
  "protocol_version": "0.0.1",
  "timestamp": 1530488933194,
  "object": {
    "_id": "5b396865cbf4239c10000001",
    "name": "Joe Testerson",
    "age": 5,
    "best_friends": []
  },
  "rules": {
    "editfields": true,
    "addfields": false,
    "deletefields": false,
    "renamefields": false,
    "fields": {
      "_id": {
        "editable": false,
        "deletable": true,
        "renameable": true,
        "type": "string"
      },
      "name": {
        "editable": false,
        "deletable": true,
        "renameable": true,
        "type": "string"
      },
      "age": {
        "editable": true,
        "deletable": false,
        "renameable": true,
        "type": "number",
        "min": 0,
        "max": 100
      },
      "best_friends": {
        "editable": true,
        "deletable": false,
        "renameable": true,
        "type": "array",
        "max": 5
      }
    }
  }
}

```



## Security & Permissions

By default objects created using this library are publicly viewable and editable. This library offers several approaches to keeping objects permissioned and secure:



### AES Encryption

Objects and updates written using this library can be encrypted using AES256. Doing so results in object storage that can only be read and updated by the holder of the private encryption key.

To use AES, specify your key during initialization:

```javascript
var db = new FactomObjectDB({
    //... other options
    private_key : 'my awesome passphrase' //private key string or buffer
});
```



### Cryptographic Signatures (Coming Soon)

Have an object you want to be publicly readable, but only want to allow updates from authorized parties? Each update entry can be signed using asymmetric encryption keys 



### Obfuscation

The library uses deflate compression to shrink the data that is put into Factom. This means that the entries this library creates are not human readable. This will be made optional in the near future



<u>Please note that once an object initialized, AES and compression cannot be changed. Attempting to read an object using incorrect encryption and compression settings will result in an error.</u>



## Todo

- Better examples for field and object rules
- Full object and field rules table with descriptions
- Signature based validation for updates
- Make deflate compression optional for human readability
- Unit testing for many, many, many test cases