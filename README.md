Good catch 👍 — the question you shared clearly talks about products in inventory (bikes and scooters), not vehicles.
So the design should revolve around Product → BikeProduct / ScooterProduct.

I will correct the entire design and code accordingly so it matches the problem statement exactly and is clean for interviews.

1️⃣ Functional Requirements

Add product to inventory

Remove product from inventory

Add customer

Rent a product

Track inventory

Track rented products

Calculate customer balance

Track overdue rentals

Return rented product

Calculate rental charges

Products supported:

Bike

Scooter

Bike attribute

size → SMALL, MEDIUM, LARGE

Scooter attribute

style → ELECTRIC, GAS

2️⃣ Non Functional Requirements

System must support:

• Concurrent rentals
• Thread safe inventory updates
• Extensible product types
• High availability
• Fast product lookup

3️⃣ Core Entities
Product (abstract)
  ├── BikeProduct
  └── ScooterProduct

Customer
Rental
Charge
Inventory
4️⃣ Class Diagram
                    +---------------------+
                    |    RentalService    |
                    |     (Singleton)     |
                    +---------------------+
                        |            |
                        v            v
                ProductRepository  RentalRepository
                        |
                        v
                ProductAssignStrategy
                        |
                -----------------------
                |                     |
         BikeSizeStrategy      ScooterStyleStrategy


Entities
--------
Product
  ├── BikeProduct
  └── ScooterProduct

Customer
Rental
Charge
5️⃣ Product Model
Product (Abstract)
abstract class Product {

    String id;

    boolean available;

    Product(String id) {
        this.id = id;
        this.available = true;
    }
}
BikeProduct
class BikeProduct extends Product {

    BikeSize size;

    BikeProduct(String id, BikeSize size) {
        super(id);
        this.size = size;
    }
}
ScooterProduct
class ScooterProduct extends Product {

    ScooterStyle style;

    ScooterProduct(String id, ScooterStyle style) {
        super(id);
        this.style = style;
    }
}
Enums
enum BikeSize {
    SMALL,
    MEDIUM,
    LARGE
}

enum ScooterStyle {
    ELECTRIC,
    GAS
}
6️⃣ Customer
class Customer {

    String id;

    String name;

    double balance;

    Customer(String id, String name) {

        this.id = id;
        this.name = name;
        this.balance = 0;
    }
}
7️⃣ Rental Entity
class Rental {

    String id;

    String productId;

    String customerId;

    long startTime;

    long expectedReturnTime;

    boolean returned;
}
8️⃣ Charge Entity
class Charge {

    String id;

    String rentalId;

    double amount;

    boolean paid;
}
9️⃣ Product Repository (Thread Safe Inventory)
import java.util.concurrent.ConcurrentHashMap;
import java.util.Collection;

class ProductRepository {

    ConcurrentHashMap<String, Product> products = new ConcurrentHashMap<>();

    void addProduct(Product p) {
        products.put(p.id, p);
    }

    void removeProduct(String id) {
        products.remove(id);
    }

    Product getProduct(String id) {
        return products.get(id);
    }

    Collection<Product> getAllProducts() {
        return products.values();
    }
}
🔟 Rental Repository
import java.util.concurrent.ConcurrentHashMap;

class RentalRepository {

    ConcurrentHashMap<String, Rental> rentals = new ConcurrentHashMap<>();

    void save(Rental r) {
        rentals.put(r.id, r);
    }

    Rental get(String id) {
        return rentals.get(id);
    }
}
1️⃣1️⃣ Strategy Pattern (Product Assignment)

Different logic for:

• Bike size
• Scooter style

Strategy Interface
interface ProductAssignStrategy {

    Product assignProduct(ProductRepository repo);
}
Strategy 1 — Bike Size Strategy
class BikeSizeStrategy implements ProductAssignStrategy {

    BikeSize size;

    BikeSizeStrategy(BikeSize size) {
        this.size = size;
    }

    public Product assignProduct(ProductRepository repo) {

        for(Product p : repo.getAllProducts()) {

            if(p instanceof BikeProduct) {

                BikeProduct bike = (BikeProduct)p;

                if(bike.size == size && bike.available) {

                    synchronized (bike) {

                        if(bike.available) {

                            bike.available = false;

                            return bike;
                        }
                    }
                }
            }
        }

        return null;
    }
}
Strategy 2 — Scooter Style Strategy
class ScooterStyleStrategy implements ProductAssignStrategy {

    ScooterStyle style;

    ScooterStyleStrategy(ScooterStyle style) {
        this.style = style;
    }

    public Product assignProduct(ProductRepository repo) {

        for(Product p : repo.getAllProducts()) {

            if(p instanceof ScooterProduct) {

                ScooterProduct scooter = (ScooterProduct)p;

                if(scooter.style == style && scooter.available) {

                    synchronized (scooter) {

                        if(scooter.available) {

                            scooter.available = false;

                            return scooter;
                        }
                    }
                }
            }
        }

        return null;
    }
}
1️⃣2️⃣ Factory Pattern

Used to create products.

class ProductFactory {

    static Product createBike(String id, BikeSize size) {

        return new BikeProduct(id, size);
    }

    static Product createScooter(String id, ScooterStyle style) {

        return new ScooterProduct(id, style);
    }
}
1️⃣3️⃣ Rental Service (Thread Safe Singleton)
import java.util.UUID;

class RentalService {

    private static volatile RentalService INSTANCE;

    ProductRepository productRepo;

    RentalRepository rentalRepo;

    private RentalService() {

        productRepo = new ProductRepository();

        rentalRepo = new RentalRepository();
    }

    public static RentalService getInstance() {

        if(INSTANCE == null) {

            synchronized(RentalService.class) {

                if(INSTANCE == null) {

                    INSTANCE = new RentalService();
                }
            }
        }

        return INSTANCE;
    }

    Rental rentProduct(ProductAssignStrategy strategy,
                       Customer customer) {

        Product product = strategy.assignProduct(productRepo);

        if(product == null)
            return null;

        Rental rental = new Rental();

        rental.id = UUID.randomUUID().toString();

        rental.productId = product.id;

        rental.customerId = customer.id;

        rental.startTime = System.currentTimeMillis();

        rental.expectedReturnTime =
                rental.startTime + (24 * 60 * 60 * 1000);

        rental.returned = false;

        rentalRepo.save(rental);

        return rental;
    }
}
1️⃣4️⃣ API Design
Add Product
POST /products

Request

{
 "type":"BIKE",
 "size":"MEDIUM"
}
Add Customer
POST /customers
Rent Product
POST /rentals

Request

{
 "customerId":"C1",
 "productType":"BIKE",
 "size":"MEDIUM"
}
Return Product
POST /rentals/return
Get Customer Balance
GET /customers/{id}/balance
1️⃣5️⃣ Database Schema
products
id (PK)
type
size
style
available
customers
id (PK)
name
balance
rentals
id (PK)
product_id
customer_id
start_time
expected_return_time
returned
charges
id
rental_id
amount
paid
1️⃣6️⃣ SQL Queries
Fetch available products
SELECT *
FROM products
WHERE available = true;
Fetch booked products
SELECT *
FROM rentals
WHERE returned = false;
Identify overdue rentals
SELECT *
FROM rentals
WHERE returned = false
AND expected_return_time < NOW();
1️⃣7️⃣ Concurrency Handling

Two important mechanisms used:

1️⃣ ConcurrentHashMap

Thread-safe inventory storage.

2️⃣ synchronized(product)

Prevents two users renting the same product.

Example race condition:

Thread A → sees product available
Thread B → sees product available

Lock acquired → only one rents
1️⃣8️⃣ Why This Design Is Strong for Interviews

Demonstrates:

✔ Strategy Pattern
✔ Factory Pattern
✔ Singleton Pattern
✔ Repository Pattern
✔ Thread Safety
✔ Inventory management
✔ Rental lifecycle
✔ SQL querying

If you'd like, I can also show 3 very important improvements interviewers expect next (this will make your LLD much stronger):

1️⃣ Complete Spring Boot version (Controller → Service → Repository)
2️⃣ Sequence Diagram for Rent + Return flow
3️⃣ Scaling design (Redis inventory cache + distributed locks)

These are very commonly asked follow-ups in LLD interviews.
