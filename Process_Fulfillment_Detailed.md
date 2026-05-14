# GUIDEBOOK FULFILLMENT: PROCESS LOGIC

---

## STAGE 1: PICKLIST GENERATION (SYSTEM DECISIONS)

### 1.1 Finding and Grouping Orders
*   The system scans `OrderHeader` and `OrderItem` for all orders that have a `statusId` of "Approved" (from `StatusItem`).
*   It groups these orders by `productId` to make picking faster. Orders for the same guidebook model stay together in one `Picklist`.
*   Within each group, the system sorts by `orderDate` to maintain FIFO priority for customer orders.

### 1.2 Picking the Right Batch (FIFO)
*   For each guidebook, the system suggests which physical `InventoryItem` should be picked.
*   It checks the `datetimeReceived` of all stock in that `Facility` and selects the oldest `Lot` (FIFO). 
*   The suggested `lotId` is assigned to each `PicklistItem` task to ensure stock rotation.

### 1.3 Managing Cart Capacity
*   Each picking cart (`FixedAsset`) has a `maxUnitCapacity` stored in the `FixedAssetAttribute` table.
*   The system fills a new `Picklist` order by order. Both the guidebook and its manual (accessory) count as units against the cart's limit.
*   If adding a full order exceeds the `maxUnitCapacity`, the system skips it for the next cart. Orders are never split across multiple carts to keep things simple.

### 1.4 Linking Accessories
*   The system automatically checks the `ProductAssoc` table for any manuals or assembly parts linked to the guidebook.
*   It creates a separate work line in the `Picklist` for each accessory. 
*   These accessory lines are linked to the same `PicklistBin` as the main guidebook so they stay grouped together.

### 1.5 Separating Orders on the Cart (Bins)
*   The system assigns a unique `PicklistBin` (Slot) on the cart for every order in the picklist.
*   This ensures that the guidebook, manuals, and assembly parts for "Order A" are kept in Bin #1, separate from "Order B" in Bin #2. This fulfills the requirement of making accessories easily separable.

---

## STAGE 2: PICKING OPERATIONS (WAREHOUSE)

### 2.1 Starting the job
*   The picker logs in with their `UserLogin` (acting as a `PartyRole` of PICKER).
*   The system suggests a `FixedAsset` (Cart ID) and the picker scans its barcode to confirm the `fixedAssetId`. This marks the cart as "In Use".

### 2.2 Picking items (by location)
*   The app displays a list sorted by `FacilityLocation` (Aisle, Shelf, Bin) for efficient walking.
*   The screen shows the Product Name and the specific `PicklistBin` (Slot) on the cart where the item must be placed.

### 2.3 Scanning and batch check
*   At each location, the picker scans the unique `serialNumber` of the guidebook.
*   The system validates the serial against the `InventoryItem` and `Lot` suggested in Stage 1.
*   If the picker scans a serial from a newer batch, a warning shows: *"Please pick from Batch [lotId] instead"*.

### 2.4 Picking manuals and assembly parts
*   After the guidebook is scanned, the app automatically prompts for the linked Manuals or Assembly items.
*   The picker scans these parts and places everything into the same assigned `PicklistBin` slot.

### 2.5 Finishing the job
*   When all `PicklistItem` entries for the job are scanned, the app confirms completion.
*   The picker moves the cart to the "Docking Station" area.
*   The system updates the `Picklist` status to "Picked" and records the completion timestamp.

---

## STAGE 3: PACKING & INVOICING (BEFORE DOCKING)

### 3.1 Verification & Box Selection
*   The packer (acting as a `PartyRole` of PACKER) scans the `fixedAssetId` to start.
*   For every line item in the `PicklistBin`, the screen shows clear packing instructions retrieved from `ProductAttribute` or `ProductContent`.
*   The system suggests a `ShipmentBoxType` based on item dimensions and quantity.
*   **Instructions**: The screen shows exactly where each part goes:
    *   Place **Main Item** in the center compartment.
    *   Place **Assembly parts** and **Manuals** in the side slots.

### 3.2 Real-time Invoicing
*   As the items are verified and placed in the box, the system creates the `Shipment`, `ShipmentItem`, and `ShipmentPackage`.
*   The unique `InventoryItem` (Serial Number) is linked to the `ShipmentPackageContent` for exact tracking.
*   The system automatically generates the `Invoice` (and `InvoiceItem`) on the fly.

### 3.3 Finalizing the Box
*   The packer prints the `Invoice` and places it **inside** the `ShipmentPackage`.
*   The box is sealed and the shipping label is applied.
*   **Move to Docking**: Only after the invoice is inside and the box is ready, the picker/packer moves the finished box to the final "Docking Station" for pickup.

### 3.4 System Completion
*   The system records the `ItemIssuance` to officially deduct the stock from the warehouse.
*   The `OrderHeader` status is updated to "Completed" and the `Shipment` status is moved to "Shipped".
