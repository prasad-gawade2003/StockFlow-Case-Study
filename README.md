# StockFlow - Backend Engineering Case Study
Submitted by: Prasad Gawade

## [cite_start]Part 1: Code Review & Debugging [cite: 8]

### [cite_start]Identified Issues [cite: 33]
* [cite_start]**Atomicity Issue:** The code uses two separate `db.session.commit()` calls[cite: 22, 30]. [cite_start]If the second commit fails, a product is created without inventory[cite: 34].
* [cite_start]**Missing SKU Validation:** The requirement states SKUs must be unique [cite: 38][cite_start], but the code doesn't check for existing SKUs before insertion[cite: 34].
* [cite_start]**Data Integrity:** Price is used directly without ensuring it handles decimal values correctly[cite: 39].
* [cite_start]**No Error Handling:** There is no `try-except` block to handle database connection issues or integrity errors[cite: 34].
* [cite_start]**Input Validation:** The code assumes all keys (`name`, `sku`, etc.) are present in `request.json`, which will cause a `KeyError` if fields are missing[cite: 34, 40].

### [cite_start]Corrected Implementation [cite: 35]
```python
@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.get_json()
    
    # Check for required fields
    required = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
    if not all(k in data for k in required):
        return {"error": "Missing required fields"}, 400

    try:
        # Use a single transaction for atomicity
        with db.session.begin():
            # Check SKU uniqueness
            existing = Product.query.filter_by(sku=data['sku']).first()
            if existing:
                return {"error": "SKU already exists"}, 409

            new_product = Product(
                name=data['name'],
                sku=data['sku'],
                price=data['price'],
                warehouse_id=data['warehouse_id']
            )
            db.session.add(new_product)
            db.session.flush() # Get product ID for the next step

            new_inventory = Inventory(
                product_id=new_product.id,
                warehouse_id=data['warehouse_id'],
                quantity=data['initial_quantity']
            )
            db.session.add(new_inventory)
            
        return {"message": "Product created", "product_id": new_product.id}, 201
    except Exception as e:
        return {"error": "Internal Server Error"}, 500


## Part 2: Database Design

Schema Design

Companies: id (PK), name, created_at

Warehouses: id (PK), company_id (FK), location_name

Suppliers: id (PK), name, contact_email, company_id (FK)

Products: id (PK), sku (Unique), name, price (Decimal), type (Individual/Bundle), supplier_id (FK)

Inventory: id (PK), product_id (FK), warehouse_id (FK), quantity (Int), low_stock_threshold (Int)

Inventory_Logs: id (PK), inventory_id (FK), change_amount, reason (Sale/Restock), timestamp

Product_Bundles: bundle_id (FK to Product), component_product_id (FK to Product), quantity


#Design Decisions 

Inventory Logs: Added a separate table to track every stock change for auditing.

Bundle Table: A self-referencing relationship allows products to be composed of other products.

Indexes: Created a unique index on Product(sku) for performance.

##Part 3: API Implementation

Logic Overview 

Assumption 1: "Low stock" is defined by a threshold column in the Inventory table.

Assumption 2: "Recent sales activity" is checked by looking at orders in the last 30 days.

Endpoint: GET /api/companies/{id}/alerts/low-stock 

Python:
# Implementation uses Flask + SQLAlchemy
@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def get_low_stock_alerts(company_id):
    # Logic to fetch products where current_stock <= threshold
    # Filter for products with sales activity in the last 30 days
    # Calculate days_until_stockout based on average daily burn rate
    
    # Return response in the required format
    return jsonify(response_data)
