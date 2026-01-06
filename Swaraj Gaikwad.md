# StockFlow Case Study - Submission

**Candidate**: Swaraj Gaikwad 
---

## Part 1: Code Review & Debugging

### Clarifying Questions

Before fixing, I would ask:

1. **Schema Discrepancy**: The code saves `warehouse_id` into the Product table. Does Product actually have this column? If yes, this violates "Products can exist in multiple warehouses" (normalization issue). If no, this code is crashing in production.

2. **Optional Fields**: Requirements state "Some fields might be optional," but the code has hard dependency on `name`, `sku`, `price`, `warehouse_id`, and `initial_quantity`. Which are optional?

3. **Ambiguous Comment**: The line `# Update inventory count` suggests modifying an existing record, but `Inventory(...)` creates a new one. Is this a creation endpoint with outdated comment?

4. **SKU Uniqueness**: SKU is said to be unique but code doesn't check. Should duplicate SKU update existing or throw error?

5. **Transaction Behavior**: Should we create Product even if Inventory creation fails? Or should both succeed/fail together?

### Issues Identified

**Issue 1: No Transaction Safety (CRITICAL)**  
Two separate `db.session.commit()` calls. If second fails, Product exists without Inventory record.

**Issue 2: No Input Validation**  
Direct `data['field']` access without checking if fields exist or have valid values. Missing fields cause `KeyError` (500 error).

**Issue 3: No SKU Uniqueness Check**  
Requirements say "SKUs must be unique," but no validation exists. Duplicate SKUs could be inserted.

**Issue 4: No Error Handling**  
No try-except blocks. Unhandled exceptions return generic 500 errors.

**Issue 5: Inconsistent HTTP Response**  
No proper status code (should be 201 for resource creation).

**Issue 6: Warehouse Validation Missing**  
No check if `warehouse_id` exists. Foreign key constraint violation possible.

**Issue 7: `initial_quantity` Not Validated**  
No check if it exists, is a number, or is non-negative.

---

## Part 2: Database Design

### Clarifying Questions

**About Companies:**
1. What company details do we need to store?
2. Are we creating APIs for company management?

**About Warehouses:**
3. What warehouse details do we need?
4. Is a warehouse unique to each company, or shared?
5. Are we creating new warehouses via API?

**About Products & Inventory:**
6. Can products be stored in different companies' warehouses?
7. Track inventory per company or per warehouse?
8. What level of tracking - just quantity or batch/lot/expiry?

**About Bundles:**
9. When bundle sold, does bundle count decrease OR individual components?
10. When component levels change, does bundle availability change automatically?
11. Can bundles contain products from other companies?
12. Can bundles contain other bundles (nested)?

**About Suppliers:**
13. Do we need to store supplier information?
14. What supplier info - name, contact, payment terms, lead time?
15. Can one supplier serve multiple companies?
16. Does supplier supply specific products or all?

### SQL DDL Schema

```sql
-- 1. Companies
CREATE TABLE companies (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    email           VARCHAR(255) UNIQUE,
    phone           VARCHAR(20),
    address         TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active       BOOLEAN DEFAULT TRUE
);
CREATE INDEX idx_companies_name ON companies(name);

-- 2. Warehouses
CREATE TABLE warehouses (
    id              SERIAL PRIMARY KEY,
    company_id      INTEGER NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    location        VARCHAR(255),
    address         TEXT,
    capacity        INTEGER,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active       BOOLEAN DEFAULT TRUE,
    UNIQUE(company_id, name)
);
CREATE INDEX idx_warehouses_company ON warehouses(company_id);

-- 3. Products
CREATE TABLE products (
    id              SERIAL PRIMARY KEY,
    company_id      INTEGER NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    sku             VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    price           DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
    cost_price      DECIMAL(10, 2) CHECK (cost_price >= 0),
    category        VARCHAR(100),
    is_bundle       BOOLEAN DEFAULT FALSE,
    min_stock_level INTEGER DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active       BOOLEAN DEFAULT TRUE,
    UNIQUE(company_id, sku)
);
CREATE INDEX idx_products_company ON products(company_id);
CREATE INDEX idx_products_sku ON products(sku);

-- 4. Bundle Items (Self-Referencing)
CREATE TABLE bundle_items (
    id              SERIAL PRIMARY KEY,
    bundle_id       INTEGER NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    component_id    INTEGER NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    quantity        INTEGER NOT NULL CHECK (quantity > 0),
    UNIQUE(bundle_id, component_id),
    CHECK (bundle_id != component_id)
);

-- 5. Inventory
CREATE TABLE inventory (
    id              SERIAL PRIMARY KEY,
    product_id      INTEGER NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    warehouse_id    INTEGER NOT NULL REFERENCES warehouses(id) ON DELETE CASCADE,
    quantity        INTEGER NOT NULL DEFAULT 0 CHECK (quantity >= 0),
    reserved_qty    INTEGER DEFAULT 0 CHECK (reserved_qty >= 0),
    last_restocked  TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(product_id, warehouse_id)
);
CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_inventory_warehouse ON inventory(warehouse_id);

-- 6. Inventory History (Audit Trail)
CREATE TABLE inventory_history (
    id              SERIAL PRIMARY KEY,
    inventory_id    INTEGER NOT NULL REFERENCES inventory(id) ON DELETE CASCADE,
    product_id      INTEGER NOT NULL,
    warehouse_id    INTEGER NOT NULL,
    change_type     VARCHAR(50) NOT NULL,
    quantity_before INTEGER NOT NULL,
    quantity_change INTEGER NOT NULL,
    quantity_after  INTEGER NOT NULL,
    reference_id    VARCHAR(100),
    notes           TEXT,
    created_by      INTEGER,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_inv_history_product ON inventory_history(product_id);
CREATE INDEX idx_inv_history_date ON inventory_history(created_at);

-- 7. Suppliers
CREATE TABLE suppliers (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    contact_name    VARCHAR(255),
    email           VARCHAR(255),
    phone           VARCHAR(20),
    address         TEXT,
    payment_terms   VARCHAR(100),
    lead_time_days  INTEGER,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active       BOOLEAN DEFAULT TRUE
);

-- 8. Supplier-Product Relationship
CREATE TABLE supplier_products (
    id              SERIAL PRIMARY KEY,
    supplier_id     INTEGER NOT NULL REFERENCES suppliers(id) ON DELETE CASCADE,
    product_id      INTEGER NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    supplier_sku    VARCHAR(100),
    cost_price      DECIMAL(10, 2),
    min_order_qty   INTEGER DEFAULT 1,
    is_preferred    BOOLEAN DEFAULT FALSE,
    UNIQUE(supplier_id, product_id)
);

-- 9. Company-Supplier Relationship
CREATE TABLE company_suppliers (
    id              SERIAL PRIMARY KEY,
    company_id      INTEGER NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    supplier_id     INTEGER NOT NULL REFERENCES suppliers(id) ON DELETE CASCADE,
    contract_start  DATE,
    contract_end    DATE,
    is_active       BOOLEAN DEFAULT TRUE,
    UNIQUE(company_id, supplier_id)
);
```

### Design Decisions

- `DECIMAL(10,2)` for price - avoids floating point errors
- `is_active` for soft delete - preserves referential integrity
- `UNIQUE(product_id, warehouse_id)` in inventory - prevents duplicates
- `inventory_history` fulfills "Track when inventory levels change"
- `is_preferred` in supplier_products - quick lookup for reordering
- Self-referencing `bundle_items` handles "bundles containing products"

---

## Part 3: API Implementation

### Assumptions

**About "Recent Sales Activity":**
1. Products sold in last 30 days are "recent"
2. Using `inventory_history` with `change_type = 'SALE'`

**About "Low Stock Threshold":**
3. Each product has `min_stock_level` field
4. Low stock: `current_stock < min_stock_level`

**About "Days Until Stockout":**
5. Based on average daily sales over last 30 days
6. Formula: `current_stock / avg_daily_sales`

**About Supplier:**
7. Use supplier marked `is_preferred = TRUE`
8. Fallback: last supplier from whom we received product

**About Authorization:**
9. User can only view alerts for their companies
10. `company_id` validated before query

**Other:**
11. Pagination handled if needed (not implemented)
12. Results sorted by `days_until_stockout` (ascending)
13. Out-of-stock items excluded
14. Proper HTTP status codes returned

### Questions Before Implementing

1. What defines "recent sales activity"? 7/30/90 days?
2. Include zero stock or only low (non-zero)?
3. If multiple suppliers, which to return?
4. Sort by urgency?
5. Pagination needed?
6. Cache response?

### FastAPI Implementation

```python
from fastapi import FastAPI, HTTPException, Depends
from sqlalchemy.orm import Session
from sqlalchemy import func
from datetime import datetime, timedelta
from typing import Optional

app = FastAPI()

@app.get("/api/companies/{company_id}/alerts/low-stock")
async def get_low_stock_alerts(company_id: int, db: Session = Depends(database.get_db)):
    """
    Get low-stock alerts for a company.
    Business Rules:
    - Low stock: current_stock < product.min_stock_level
    - Only products with recent sales (last 30 days)
    - Returns preferred supplier OR last supplier used
    """
    
    # Validate company exists
    company = db.query(Company).filter(
        Company.id == company_id, Company.is_active == True
    ).first()
    if not company:
        raise HTTPException(status_code=404, detail="Company not found")
    
    # Get warehouses for this company
    warehouse_ids = [w.id for w in db.query(Warehouse).filter(
        Warehouse.company_id == company_id,
        Warehouse.is_active == True
    ).all()]
    
    if not warehouse_ids:
        return {"alerts": [], "total_alerts": 0}
    
    cutoff_date = datetime.utcnow() - timedelta(days=30)
    
    # Query low-stock inventory
    low_stock_items = db.query(Inventory, Product, Warehouse).join(
        Product, Inventory.product_id == Product.id
    ).join(
        Warehouse, Inventory.warehouse_id == Warehouse.id
    ).filter(
        Inventory.warehouse_id.in_(warehouse_ids),
        Inventory.quantity < Product.min_stock_level,
        Inventory.quantity > 0,
        Product.is_active == True
    ).all()
    
    alerts = []
    for inv, product, warehouse in low_stock_items:
        # Check recent sales activity
        recent_sale = db.query(InventoryHistory).filter(
            InventoryHistory.product_id == product.id,
            InventoryHistory.change_type == 'SALE',
            InventoryHistory.created_at >= cutoff_date
        ).first()
        
        if not recent_sale:
            continue
        
        # Calculate days until stockout
        total_sales = db.query(func.sum(func.abs(InventoryHistory.quantity_change))).filter(
            InventoryHistory.product_id == product.id,
            InventoryHistory.warehouse_id == warehouse.id,
            InventoryHistory.change_type == 'SALE',
            InventoryHistory.created_at >= cutoff_date
        ).scalar() or 0
        
        days_until_stockout = None
        if total_sales > 0:
            avg_daily = total_sales / 30
            days_until_stockout = int(inv.quantity / avg_daily)
        
        supplier_info = get_supplier_info(db, product.id)
        
        alerts.append({
            "product_id": product.id,
            "product_name": product.name,
            "sku": product.sku,
            "warehouse_id": warehouse.id,
            "warehouse_name": warehouse.name,
            "current_stock": inv.quantity,
            "threshold": product.min_stock_level,
            "days_until_stockout": days_until_stockout,
            "supplier": supplier_info
        })
    
    alerts.sort(key=lambda x: x['days_until_stockout'] or float('inf'))
    return {"alerts": alerts, "total_alerts": len(alerts)}


def get_supplier_info(db: Session, product_id: int) -> Optional[dict]:
    """Get preferred supplier OR last supplier used."""
    
    # Try preferred supplier
    sp = db.query(SupplierProduct).filter(
        SupplierProduct.product_id == product_id,
        SupplierProduct.is_preferred == True
    ).first()
    
    # Fallback: last supplier from inventory history
    if not sp:
        last_addition = db.query(InventoryHistory).filter(
            InventoryHistory.product_id == product_id,
            InventoryHistory.change_type == 'ADDITION'
        ).order_by(InventoryHistory.created_at.desc()).first()
        
        if last_addition:
            sp = db.query(SupplierProduct).filter(
                SupplierProduct.product_id == product_id
            ).first()
    
    if not sp:
        return None
    
    supplier = db.query(Supplier).get(sp.supplier_id)
    return {
        "id": supplier.id,
        "name": supplier.name,
        "contact_email": supplier.email
    } if supplier else None
```

### Edge Cases Handled

| Edge Case | Response |
|-----------|----------|
| Company not found | 404 error |
| No warehouses | Empty alerts array |
| No recent sales | Product skipped |
| No supplier | `supplier: null` |
| Zero stock | Excluded |
| No sales data | `days_until_stockout: null` |

---
