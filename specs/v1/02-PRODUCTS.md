# 02 — Product Management

**Requirement 2** — covers all 9 acceptance criteria for product CRUD, outlet assignment, variations, stock linkage, unique SKU/barcode, and performance analysis.

> **User Story:** As a Business Owner, I want to manage products and link them to inventory, so that stock is deducted automatically and I can analyse product performance.

## Acceptance Criteria

1. WHEN the Business_Owner submits a valid new product, THE System SHALL create the Product with its name, category, price, cost, SKU, and barcode.
2. WHEN the Business_Owner submits an edit to an existing Product, THE System SHALL update the Product with the submitted values.
3. WHEN the Business_Owner requests deletion of a Product, THE System SHALL remove the Product from the catalogue.
4. WHEN the Business_Owner assigns a Product to one or more Outlets, THE System SHALL make the Product available for sale only at the assigned Outlets.
5. WHEN the Business_Owner creates a Product_Variation for a Product, THE System SHALL store the Product_Variation and make it available for sale under that Product.
6. WHERE a Product is linked to a Stock_Item, WHEN a Transaction that includes the Product is completed, THE System SHALL deduct the sold quantity from the linked Stock_Item.
7. IF the Business_Owner submits a Product with a SKU or barcode that already exists, THEN THE System SHALL reject the submission and return a descriptive error.
8. WHEN a user with product access requests product sales performance, THE System SHALL display the quantity sold and revenue generated for each Product over the selected period.
9. WHEN a user with product access requests best-selling products, THE System SHALL display Products ranked in descending order of quantity sold over the selected period.

## Module Interface

```typescript
interface ProductService {
  createProduct(input: ProductInput): Promise<Product>;        // 2.1, rejects dup SKU/barcode 2.7
  updateProduct(productId: ID, input: ProductInput): Promise<Product>; // 2.2
  deleteProduct(productId: ID): Promise<void>;                 // 2.3
  assignToOutlets(productId: ID, outletIds: ID[]): Promise<void>; // 2.4
  createVariation(productId: ID, input: VariationInput): Promise<ProductVariation>; // 2.5
  getSalesPerformance(filter: PeriodFilter): Promise<ProductPerformance[]>; // 2.8
  getBestSellers(filter: PeriodFilter): Promise<ProductPerformance[]>;      // 2.9
}
```

## Data Model Entities

Monetary fields are integers in minor currency units.

```typescript
interface Product {
  id: ID; name: string; category: string;
  price: Money; cost: Money;
  sku: string;        // unique (2.7)
  barcode: string;    // unique (2.7)
  linkedStockItemId?: ID; // (2.6)
}

interface ProductVariation { id: ID; productId: ID; name: string; priceDelta: Money; sku?: string; barcode?: string; }
interface ProductOutlet { productId: ID; outletId: ID; } // availability (2.4)
```

## Notes

- **Unique SKU/barcode (2.7):** Enforced via database unique constraints on `Product.sku` and `Product.barcode`. A duplicate submission is rejected with a descriptive error rather than persisted.
- **Stock deduction on sale (2.6):** This is not a direct `ProductService` method call. It is performed by the **Sales Module** within the sale transaction for any sold product linked to a stock item. See `03-SALES-TRANSACTIONS.md` and `01-INVENTORY.md`.
- **Outlet availability (2.4):** A Product is sellable only at the Outlets recorded in `ProductOutlet`. The POS search (4.1) filters by the Cashier's assigned Outlet.
