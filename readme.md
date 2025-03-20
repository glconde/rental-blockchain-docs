# **SMART Contract Rental Application**
Here we will discuss the details of implementing a SMART contract rental application.

This guide will help you set up the backend for your rental smart contract dApp and integrate it with an Airbnb-like frontend. The backend will interact with your deployed smart contract on a testnet (Polygon Mumbai) to manage rental agreements, payments, security deposits, and bookings.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Setting Up Hardhat](#setting-up-hardhat)
3. [Creating the Smart Contract](#creating-the-smart-contract)
4. [Deploying to Testnet](#deploying-to-testnet)
5. [Setting Up the Backend](#setting-up-the-backend)
6. [Frontend Integration (Airbnb-like)](#frontend-integration-airbnb-like)
7. [Smart Contract Interaction](#smart-contract-interaction)
8. [Future Enhancements](#future-enhancements)

---

## **Prerequisites**

Before setting up the backend, make sure you have the following installed:

- **Node.js & npm**: For backend development.
- **Hardhat**: For smart contract deployment and management.
- **Ethers.js**: For interacting with the Ethereum blockchain.
- **Infura or Alchemy Account**: To connect to testnets like Polygon Mumbai.
- **Next.js**: To build the frontend.

---

## **Setting Up Hardhat**

First, we’ll initialize Hardhat in your project and set up the configuration for deployment.

### Step 1: Install Hardhat and Dependencies

```bash
mkdir rental-smart-contract
cd rental-smart-contract
npm init -y
npm install --save-dev hardhat
npm install @nomiclabs/hardhat-ethers ethers dotenv
```

### Step 2: Initialize a Hardhat Project
Run the following command to initialize a basic Hardhat project:
```bash
npx hardhat
```
Follow the prompts to create a basic project.

### Step 3: Configure Hardhat for Polygon Mumbai
In **`hardhat.config.js`**, set up your network configuration to deploy to Polygon Mumbai.
```javascript
require('@nomiclabs/hardhat-ethers');
require('dotenv').config();

module.exports = {
  solidity: "0.8.7",
  networks: {
    mumbai: {
      url: `https://polygon-mumbai.infura.io/v3/${process.env.INFURA_PROJECT_ID}`,
      accounts: [`0x${process.env.PRIVATE_KEY}`]
    },
  },
};

```
You’ll need to create a **`.env`** file to store sensitive information:
```env
INFURA_PROJECT_ID=your-infura-project-id
PRIVATE_KEY=your-wallet-private-key
```

## **Creating the SMART Contract**
For the impementation of the SMART contract, we will be writing the contract with Solidity, a programming language that is used to write SMART contracts on the Ethereum blockchain.

### **Step 1: Setting up a basic SMART contract with variables and data structures**
1. We will first define the SPDX license identifier for the version of solidity we will be using.

```c++
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0
;
```
2. We will then define the contract by using the keyword `contract` followed by the name of the contract. For this project, we will call the contract `RentalProperty`.
```c++
pragma solidity ^0.8.0;

// Contract definition
contract RentalProperty {

}
```
3. Next, we will define the necessary data structures and variables for the SMART rental contract. Below, we have defined:
    - The owner of the contract
    - And `enum` for the rental status. The status can be `Pending`, `Active` or `Expired`.
    - A `struct` to store rental agreement details such as:
        - Property Id `propertyId`
        - Tenant's wallet address `tenantAddress`
        - Rent amount `rentAmount`
        - Security deposit `depositAmount`
        - Late Fee `lateFee`
        - Rent Due Date `rentDueDate`
        - Rent Interval `rentInterval`
        - current status of the rent `status`
        - Start timestamp of the rental agreement `startTime`
        - End time of the rental agreement `endTime`
    - A `mapping()` data type that associates a `propertyId` with the `Rental` data

```c#
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract RentalProperty {
    // Define the owner of the contract (landlord)
    address public owner;

    // Enum for rental status
    enum RentalStatus { Pending, Active, Expired }

    // Struct to store rental agreement details
    struct Rental {
        string propertyId;   // Unique id for the property
        address tenantAddress;      // Tenant's wallet address
        uint256 rentAmount;  // Monthly rent
        uint256 depositAmount;   // Security deposit
        uint256 lateFee;         // New: Fee for late payments
        uint256 rentDueDate;     // New: Timestamp for next rent due
        uint256 rentInterval;    // New: Interval between rent payments (e.g., 30 days)
        RentalStatus status; // Current status of the rental
        uint256 startTime;   // Start timestamp
        uint256 endTime;     // End timestamp (0 if not ended)
    }

    // Mapping to store rentals by property ID and track deposits
    mapping(string => Rental) public rentals;
    mapping(address => uint256) public deposits;
}
```

4. We will create events to allow our front-end to listen for contract actions

```c#
 // Event to log actions (useful for frontend)
    event RentalCreated(string propertyId, address tenant, uint256 rentAmount);
    event RentPaid(string propertyId, uint256 amount, uint256 timestamp);
    event RentalEnded(string propertyId, uint256 timestamp);
```

5. Finally, we will create a constructor to set contract deployer as the property manager/ landlord

```c#
 // Constructor to set the contract deployer as the owner
 constructor() {
        owner = msg.sender;
    }
```
So far , here is the complete code of our progress.
```c++
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract RentalProperty {
    // Define the owner of the contract (landlord)
    address public owner;

    // Enum for rental status
    enum RentalStatus { Pending, Active, Expired }

    // Struct to store rental agreement details
    struct Rental {
        string propertyId;   // Unique id for the property
        address tenantAddress;      // Tenant's wallet address
        uint256 rentAmount;  // Monthly rent
        uint256 depositAmount;   // Security deposit
        uint256 lateFee;         // New: Fee for late payments
        uint256 rentDueDate;     // New: Timestamp for next rent due
        uint256 rentInterval;    // New: Interval between rent payments (e.g., 30 days)
        RentalStatus status; // Current status of the rental
        uint256 startTime;   // Start timestamp
        uint256 endTime;     // End timestamp (0 if not ended)
    }

    // Mapping to store rentals by property ID and track deposits
    mapping(string => Rental) public rentals;
    mapping(address => uint256) public deposits;

    // Event to log actions (useful for frontend)
    event RentalCreated(string propertyId, address tenant, uint256 rentAmount);
    event RentPaid(string propertyId, uint256 amount, uint256 timestamp);
    event RentalEnded(string propertyId, uint256 timestamp);

    // Constructor to set the contract deployer as the owner
    constructor() {
        owner = msg.sender;
    }

}
```
### **Step 2: Creating Modifiers for User Access Control**

Adding modifiers provide a layer of security by restricting certain functions to either the tenant or the property manager/ landlord.

1. `onlyOwner` - ensures only property manager / landlord can call certain functions
2. `onlyTenant` - ennsures that the tenant for a particular property can call certain functions
3. `rentlActive` - checks if rental exists and is active

```c++
// 
    modifier onlyOwner() {
        require(msg.sender == owner, "Only the owner can call this function");
        _;
    }

    modifier onlyTenant(string memory _propertyId) {
        require(msg.sender == rentals[_propertyId].tenant, "Only the tenant can call this function");
        _;
    }

    modifier rentalActive(string memory _propertyId) {
        require(rentals[_propertyId].status == RentalStatus.Active, "Rental must be active");
        _;
    }
```
### **Step 3: Creating the Rental Agreement**

We will now write a function to create a basic rental agreement.

```c#
// Create a rental with deposit and late fee
    function createRental(
        string memory _propertyId,
        address _tenantAddress,
        uint256 _rentAmount,
        uint256 _depositAmount,
        uint256 _lateFee,
        uint256 _rentInterval // e.g., 30 days in seconds
    ) public onlyOwner whenNotPaused {
        require(rentals[_propertyId].tenant == address(0), "Property already rented");

        rentals[_propertyId] = Rental({
            propertyId: _propertyId,
            tenant: _tenant,
            rentAmount: _rentAmount,
            depositAmount: _depositAmount,
            lateFee: _lateFee,
            rentDueDate: 0, // Set when activated
            rentInterval: _rentInterval,
            status: RentalStatus.Pending,
            startTime: 0,
            endTime: 0
        });

        emit RentalCreated(_propertyId, _tenant, _rentAmount, _depositAmount);
    }
```

#### **Function Details**
| Function Name |`createRental`|
|------|---|
| **Parameters** | `_propertyId`, `_tenantAddress`, `_rentAmount` |
|**Visibility**| `public`|
|**Modifier**|`onlyOwner`|
|**Storage**|Adds the rental to the `rentals` mapping with a "Pending" status|
|**Event**|Emits `RentalCreated` for front-end tracking|


### **Step 4: Activating the Rental**

We will now write a function to cactivate the rental with the deposit and initial rent amount.

```c#
// Activate rental with deposit and initial rent
    function activateRental(string memory _propertyId) public payable onlyTenant(_propertyId) whenNotPaused {
        Rental storage rental = rentals[_propertyId];
        require(rental.status == RentalStatus.Pending, "Rental must be pending");
        require(msg.value >= rental.rentAmount + rental.depositAmount, "Insufficient payment (rent + deposit)");

        // Store deposit
        deposits[msg.sender] += rental.depositAmount;

        // Activate rental
        rental.status = RentalStatus.Active;
        rental.startTime = block.timestamp;
        rental.rentDueDate = block.timestamp + rental.rentInterval;

        // Transfer rent (not deposit) to owner
        payable(owner).transfer(rental.rentAmount);

        emit RentPaid(_propertyId, rental.rentAmount, block.timestamp);
    }
```

#### **Function Details**
| Function Name |`activateRental`|
|------|---|
| **Parameters** | `_propertyId` |
|**Visibility**| `public`|
|**Modifier**|`onlyTenant`|
|**Payments**|`payable` - allows the function to receive Ether|
|Post-condition|Ensures that the rental is **"Pending"** and payment is equal to the rent amount|
|**Update**|Sets status to **"Active"** and records the start time|
|**Payment**|Transfers Ether to the owner|
|**Event**|Logs the payment|

### **Step 5: Pay Rent**

We will write a function to allow the tenant to pay rent.

```c#
 // Pay rent with late fee check
    function payRent(string memory _propertyId) public payable onlyTenant(_propertyId) rentalActive(_propertyId) whenNotPaused {
        Rental storage rental = rentals[_propertyId];
        uint256 totalDue = rental.rentAmount;

        // Check if rent is late and add late fee
        if (block.timestamp > rental.rentDueDate) {
            totalDue += rental.lateFee;
        }
        require(msg.value >= totalDue, "Insufficient payment for rent and late fee");

        // Update next due date
        rental.rentDueDate += rental.rentInterval;

        // Transfer payment to owner
        payable(owner).transfer(msg.value);

        emit RentPaid(_propertyId, msg.value, block.timestamp);
    }
```
#### **Function Details**
| Function Name |`payRent`|
|------|---|
| **Parameters** | `_propertyId` |
|**Visibility**| `public`|
|**Modifier**|`onlyTenant`, `rentalActive` - ensures rental is active|
|**Payments**|`payable` - allows the function to receive Ether|
|Post-condition|Verifies the payment is equal to the rental amount|
|**Payment**|Sends Ether to the owner|
|**Event**|Logs the payment|



### **Step 6: End the Rental**

We will create a function to allow the property manager to terminate an active rental.

```c#
  function endRental(string memory _propertyId) public onlyOwner rentalActive(_propertyId) whenNotPaused {
        Rental storage rental = rentals[_propertyId];
        rental.status = RentalStatus.Expired;
        rental.endTime = block.timestamp;

        // Refund deposit if any
        uint256 deposit = deposits[rental.tenant];
        if (deposit > 0) {
            deposits[rental.tenant] = 0;
            payable(rental.tenant).transfer(deposit);
            emit DepositRefunded(_propertyId, rental.tenant, deposit);
        }

        emit RentalEnded(_propertyId, block.timestamp);
    }
```

#### **Function Details**
| Function Name |`endRental`|
|------|---|
| **Parameters** | `_propertyId` |
|**Visibility**| `public`|
|**Modifier**|`onlyOwner`, `rentalActive` - ensures rental is active|
|**Update**|Sets status to **"Expired"** and records the end time|
|**Payment**|Transfers Ether to the owner|
|**Event**|Logs the termination|



### **The Final Contract**

``` c++
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract RentalProperty {
    address public owner;
    bool public paused; // New: Pause state for emergency stops

    enum RentalStatus { Pending, Active, Expired }

    struct Rental {
        string propertyId;
        address tenant;
        uint256 rentAmount;      // Monthly rent in wei
        uint256 depositAmount;   // New: Security deposit
        uint256 lateFee;         // New: Fee for late payments
        uint256 rentDueDate;     // New: Timestamp for next rent due
        uint256 rentInterval;    // New: Interval between rent payments (e.g., 30 days)
        RentalStatus status;
        uint256 startTime;
        uint256 endTime;
    }

    mapping(string => Rental) public rentals;

    // Mapping to store rentals by property ID and to track deposits held by the contract
    mapping(address => uint256) public deposits;

    // Events to log actions (useful for frontend)
    event RentalCreated(string propertyId, address tenant, uint256 rentAmount, uint256 depositAmount);
    event RentPaid(string propertyId, uint256 amount, uint256 timestamp);
    event RentalEnded(string propertyId, uint256 timestamp);
    event DepositRefunded(string propertyId, address tenant, uint256 amount);
    event ContractPaused(bool paused);

    constructor() {
        owner = msg.sender;
        paused = false;
    }

    // Modifiers
    modifier onlyOwner() {
        require(msg.sender == owner, "Only the owner can call this function");
        _;
    }

    modifier onlyTenant(string memory _propertyId) {
        require(msg.sender == rentals[_propertyId].tenant, "Only the tenant can call this function");
        _;
    }

    modifier rentalActive(string memory _propertyId) {
        require(rentals[_propertyId].status == RentalStatus.Active, "Rental must be active");
        _;
    }

    modifier whenNotPaused() {
        require(!paused, "Contract is paused");
        _;
    }

    // New: Toggle pause state
    function setPaused(bool _paused) public onlyOwner {
        paused = _paused;
        emit ContractPaused(_paused);
    }

    // Create a rental with deposit and late fee
    function createRental(
        string memory _propertyId,
        address _tenant,
        uint256 _rentAmount,
        uint256 _depositAmount,
        uint256 _lateFee,
        uint256 _rentInterval // e.g., 30 days in seconds
    ) public onlyOwner whenNotPaused {
        require(rentals[_propertyId].tenant == address(0), "Property already rented");

        rentals[_propertyId] = Rental({
            propertyId: _propertyId,
            tenant: _tenant,
            rentAmount: _rentAmount,
            depositAmount: _depositAmount,
            lateFee: _lateFee,
            rentDueDate: 0, // Set when activated
            rentInterval: _rentInterval,
            status: RentalStatus.Pending,
            startTime: 0,
            endTime: 0
        });

        emit RentalCreated(_propertyId, _tenant, _rentAmount, _depositAmount);
    }

    // Activate rental with deposit and initial rent
    function activateRental(string memory _propertyId) public payable onlyTenant(_propertyId) whenNotPaused {
        Rental storage rental = rentals[_propertyId];
        require(rental.status == RentalStatus.Pending, "Rental must be pending");
        require(msg.value >= rental.rentAmount + rental.depositAmount, "Insufficient payment (rent + deposit)");

        // Store deposit
        deposits[msg.sender] += rental.depositAmount;

        // Activate rental
        rental.status = RentalStatus.Active;
        rental.startTime = block.timestamp;
        rental.rentDueDate = block.timestamp + rental.rentInterval;

        // Transfer rent (not deposit) to owner
        payable(owner).transfer(rental.rentAmount);

        emit RentPaid(_propertyId, rental.rentAmount, block.timestamp);
    }

    // Pay rent with late fee check
    function payRent(string memory _propertyId) public payable onlyTenant(_propertyId) rentalActive(_propertyId) whenNotPaused {
        Rental storage rental = rentals[_propertyId];
        uint256 totalDue = rental.rentAmount;

        // Check if rent is late and add late fee
        if (block.timestamp > rental.rentDueDate) {
            totalDue += rental.lateFee;
        }
        require(msg.value >= totalDue, "Insufficient payment for rent and late fee");

        // Update next due date
        rental.rentDueDate += rental.rentInterval;

        // Transfer payment to owner
        payable(owner).transfer(msg.value);

        emit RentPaid(_propertyId, msg.value, block.timestamp);
    }

    // End rental and refund deposit
    function endRental(string memory _propertyId) public onlyOwner rentalActive(_propertyId) whenNotPaused {
        Rental storage rental = rentals[_propertyId];
        rental.status = RentalStatus.Expired;
        rental.endTime = block.timestamp;

        // Refund deposit if any
        uint256 deposit = deposits[rental.tenant];
        if (deposit > 0) {
            deposits[rental.tenant] = 0;
            payable(rental.tenant).transfer(deposit);
            emit DepositRefunded(_propertyId, rental.tenant, deposit);
        }

        emit RentalEnded(_propertyId, block.timestamp);
    }

    // Optional: Allow tenant to view their deposit
    function getDeposit(string memory _propertyId) public view onlyTenant(_propertyId) returns (uint256) {
        return deposits[msg.sender];
    }

    // Optional: Allow owner to withdraw excess funds (e.g., late fees)
    function withdrawFunds() public onlyOwner {
        uint256 balance = address(this).balance;
        require(balance > 0, "No funds to withdraw");
        payable(owner).transfer(balance);
    }
}
```
## **Deploying to Testnet**

Now that Hardhat is set up, you can deploy your contract to the Polygon Mumbai testnet.

### Step 1: Create a Deployment Script
In the **`scripts/`** folder, create a file called **`deploy.js`**:
**insert deployment code details here***

### Step 2: Deploy the SMART Contract
Run the following command to deploy your contract to the Polygon Mumbai testnet:

```bash
npx hardhat run scripts/deploy.js --network mumbai
```
Once deployed, note the contract address that’s printed in the console. You’ll use this to interact with the contract from your backend.

## Setting Up the Backend
Now, let’s set up the backend server to interact with the deployed smart contract.
### Step 1: Install Backend Dependencies
In your backend directory, install necessary packages:
```bash
npm init -y
npm install express ethers dotenv body-parser
```

### Step 2: Create an Express Server
In the root of your project, create a file called **`server.js`** to set up a basic Express server:
```javascript
const express = require('express');
const { ethers } = require('ethers');
require('dotenv').config();

const app = express();
app.use(express.json());

const provider = new ethers.JsonRpcProvider('https://polygon-mumbai.infura.io/v3/YOUR_INFURA_PROJECT_ID');
const wallet = new ethers.Wallet('YOUR_PRIVATE_KEY', provider);

const contractAddress = 'YOUR_DEPLOYED_CONTRACT_ADDRESS';
const contractABI = require('./RentalContractABI.json');

const contract = new ethers.Contract(contractAddress, contractABI, wallet);

app.get('/rentals/:id', async (req, res) => {
  const rentalId = req.params.id;
  try {
    const rentalInfo = await contract.getRentalInfo(rentalId);
    res.json(rentalInfo);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching rental info', error });
  }
});

app.listen(3000, () => {
  console.log('Backend is running on port 3000');
});

```
In the above code:

- Replace 'YOUR_INFURA_PROJECT_ID' with your Infura project ID.
- Replace 'YOUR_PRIVATE_KEY' with the private key of the account that deployed the smart contract.
- Replace 'YOUR_DEPLOYED_CONTRACT_ADDRESS' with the contract address you got from the deployment step.
- Replace RentalContractABI.json with the ABI of your deployed contract.

### Step 3: Smart Contract Interaction
You can interact with your contract from the backend by calling the appropriate methods. For example:

- Get Rental Info: You can fetch the rental information by calling getRentalInfo(rentalId) from your contract.

## **Frontend Integration using Next.js**
The frontend will be built to mimic the Airbnb experience, where users can browse, book rentals, and interact with smart contracts for payments and booking management.