## Organization of Domain Logic

### 1. Transaction Scripts

**Definition**:  
a Transaction Script is essentially a procedural function that coordinates a series of steps to complete a specific transaction or task. It often calls other functions or methods to perform these steps, handling everything within that single script. The key characteristic is that all the logic needed for the transaction—like data retrieval, business logic, and database updates—is orchestrated within one procedure.

**Examples**:  
A Transaction Script for placing an order might look like this:

```python
def place_order(customer_id, product_id, quantity):
    product = get_product_by_id(product_id)
    
    if product.stock < quantity:
        raise Exception("Not enough stock")
    
    order = create_order(customer_id, product_id, quantity)
    product.stock -= quantity
    update_product(product)
    return order
```

The `place_order` function handles everything: checking stock, creating the order, and updating the product's stock. All the logic is contained within this single script.

```
def process_payment(order_id, payment_method, amount):
    # Step 1: Fetch order details
    order = get_order_by_id(order_id)
    
    # Step 2: Validate payment method
    if not is_valid_payment_method(payment_method):
        raise Exception("Invalid payment method")
    
    # Step 3: Deduct amount from customer's balance
    customer = get_customer_by_id(order.customer_id)
    if customer.balance < amount:
        raise Exception("Insufficient balance")
    
    customer.balance -= amount
    update_customer_balance(customer)
    
    # Step 4: Update order status
    order.status = "Paid"
    update_order_status(order)
    
    # Step 5: Finalize the transaction
    commit_transaction()
    return order
```

### 2. Domain Model

**Definition**:  
The Domain Layer pattern involves organizing the business logic around a **domain model** that represents the real-world entities and relationships within the application. The domain model typically consists of a set of interconnected objects that encapsulate both data and behavior relevant to the business domain.

**Example**:  
A domain model might involve classes like `Order`, `Customer`, and `Product`, each encapsulating its own logic:

```python
class Product:
    def __init__(self, product_id, name, stock):
        self.product_id = product_id
        self.name = name
        self.stock = stock
    
    def reduce_stock(self, quantity):
        if self.stock < quantity:
            raise Exception("Not enough stock")
        self.stock -= quantity

class Order:
    def __init__(self, customer, product, quantity):
        self.customer = customer
        self.product = product
        self.quantity = quantity
    
    def place(self):
        self.product.reduce_stock(self.quantity)
        # Save the order to the database
```

In this case, the `Product` class encapsulates the logic related to stock management, and the `Order` class handles the order creation process. The logic is distributed across the domain objects, making the code more modular and easier to maintain.

### 3. Table Module

**Definition**:  
The Table Module pattern is used to organize business logic around a database table.

**Example**:  
The Table Module for managing customers might look like this:

```python
class CustomerTableModule:
    def __init__(self, database_connection):
        self.db = database_connection
    
    def find_by_id(self, customer_id):
        query = "SELECT * FROM Customer WHERE id = ?"
        result = self.db.execute(query, (customer_id,))
        return result.fetchone()
    
    def update_email(self, customer_id, new_email):
        query = "UPDATE Customer SET email = ? WHERE id = ?"
        self.db.execute(query, (new_email, customer_id))
    
    def get_all_customers(self):
        query = "SELECT * FROM Customer"
        result = self.db.execute(query)
        return result.fetchall()

```

### Distinctions btwn Table Module and Domain Model

When using the **Domain Model** pattern, the business logic is organized around **domain entities** rather than being centralized in a single class that corresponds to a database table (as in the Table Module pattern). Each domain entity is an object that represents a specific concept or entity within the business domain (e.g., a `Customer`), and it encapsulates both the data and the behavior (business logic) related to that entity.

### How the Example Would Be Handled with a Domain Model Pattern

#### 1. Defining the `Customer` Domain Entity

In the Domain Model pattern, we define a `Customer` class that represents a customer in the business domain. This class would contain both the attributes (data) and methods (behavior) related to a customer.

```python
class Customer:
    def __init__(self, customer_id, name, email, balance):
        self.customer_id = customer_id
        self.name = name
        self.email = email
        self.balance = balance
    
    def update_email(self, new_email):
        # Business logic to validate and update the email
        if "@" not in new_email:
            raise ValueError("Invalid email address")
        self.email = new_email
    
    def apply_discount(self, discount_percentage):
        # Business logic to apply a discount to the customer's balance
        discount_amount = self.balance * (discount_percentage / 100)
        self.balance -= discount_amount
    
    def has_sufficient_balance(self, amount):
        return self.balance >= amount
```

### 2. Repositories for Data Access

In a Domain Model pattern, data access is typically handled by repository classes that abstract the database interactions. The repository is responsible for retrieving and saving domain objects to the database.

```python
class CustomerRepository:
    def __init__(self, database_connection):
        self.db = database_connection
    
    def find_by_id(self, customer_id):
        query = "SELECT * FROM Customer WHERE id = ?"
        result = self.db.execute(query, (customer_id,))
        row = result.fetchone()
        if row:
            return Customer(row['id'], row['name'], row['email'], row['balance'])
        else:
            return None
    
    def save(self, customer):
        query = "UPDATE Customer SET email = ?, balance = ? WHERE id = ?"
        self.db.execute(query, (customer.email, customer.balance, customer.customer_id))
```

### 3. Using the Domain Model in Application Logic

Now, the business logic can use the `Customer` domain object and interact with it directly. Here’s an example of how you might use the `Customer` domain model in an application service:

```python
def update_customer_email(customer_id, new_email, customer_repository):
    # Fetch the customer using the repository
    customer = customer_repository.find_by_id(customer_id)
    if customer is None:
        raise Exception("Customer not found")
    
    # Update the customer's email using the domain object's method
    customer.update_email(new_email)
    
    # Save the changes using the repository
    customer_repository.save(customer)
```

### Breakdown of the Example:

- **Domain Entity (`Customer`)**: The `Customer` class encapsulates both the data (e.g., `customer_id`, `name`, `email`, `balance`) and the behavior (e.g., `update_email`, `apply_discount`, `has_sufficient_balance`) associated with a customer. This makes the `Customer` object rich in behavior, representing a real-world entity.

- **Repository (`CustomerRepository`)**: The `CustomerRepository` class handles data access. It abstracts the interaction with the database and ensures that the `Customer` objects are properly loaded from and saved to the database.

- **Application Logic**: The application logic interacts with the `Customer` object through the repository, modifying the customer’s state using the methods defined within the `Customer` domain model.

### How This Differs from the Table Module Pattern:

- **Decentralization**: Unlike the Table Module, where all logic related to a table is centralized in one class, the Domain Model pattern distributes logic across multiple domain objects. Each object is responsible for its own behavior.

- **Encapsulation**: The Domain Model encapsulates data and behavior together within objects, promoting encapsulation and better modeling of real-world entities.

- **Flexibility**: The Domain Model allows for richer interactions between entities. For example, you could have methods on the `Customer` class that involve interactions with other domain objects (like `Order` or `Invoice`), allowing for more complex business logic.

### Summary

Using the Domain Model pattern, you create rich, behavior-driven objects (like `Customer`) that encapsulate both the data and the business logic relevant to that domain entity. The repository pattern complements this by handling data persistence, enabling a clear separation between the domain logic and database access. This approach is typically more flexible and scalable, especially for complex applications where the business logic is more intricate and closely tied to the domain.

"The vital difference is that a Domain Model has one instance of contract for each contract in the database whereas a Table Module has only one instance," 
