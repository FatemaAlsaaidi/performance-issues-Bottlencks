# Analysis performance issues Bottlencks_N_plus_One_Issues 
### note : This case will discuess in case of **place order**

## Overveiw
- In this artical will disccues performance issues Bottlencks, which also called N + 1 issue, this issue happen when write code that has for loop for every item of list in every item must go to database and get the information for that single item , this will take less of memory (the **Stack** and the **Heap** primary memory storage areas) but will be slow in performance .


## Code which has performance issues Bottlencks, which is function of place order feature in E-CommerceSystem 

### Has New Place Order:

- Tables: 
1. order model 
2. product model 
3. OrderProduct model 


- Logic:

1. First want to check if every product exist in the product table (** need the data from product table**))
2. Check if the stoke number of product is available or no (** need the data from product table agin to check the stok number of very product**), if availavle want to :

    1. Add new order to order table, such as (order id, user id, order date, total amount)
    2. Add information of every product in the order to OrderProduct table, such as (order id, product id, quantity)
    3. Update the stock number of every product in the product table



### Issues In Place Order Function:
```sql
//Places an order for the given list of items and user ID.
        public void PlaceOrder( List<OrderItemDTO> items, int uid)
        {
            // Temporary variable to hold the currently processed product
            Product existingProduct = null;
            decimal TotalPrice, totalOrderPrice = 0; // Variables to hold the total price of each item and the overall order
            OrderProducts orderProducts = null;

            // Validate all items in the order
            for (int i = 0; i < items.Count; i++)
            {
                TotalPrice = 0;
                existingProduct = _productService.GetProductByName(items[i].ProductName);
                if (existingProduct == null)
                    throw new Exception($"{items[i].ProductName} not Found");
                if (existingProduct.Stock < items[i].Quantity)
                    throw new Exception($"{items[i].ProductName} is out of stock");
            }
            // Create a new order for the user
            var order = new Order { UID = uid, OrderDate = DateTime.Now, TotalAmount = 0 };
            AddOrder(order); // Save the order to the database
            // Process each item in the order
            foreach (var item in items)
            {
                // Retrieve the product by its name
                existingProduct = _productService.GetProductByName(item.ProductName);
                // Calculate the total price for the current item
                TotalPrice = item.Quantity * existingProduct.Price;
                // Deduct the ordered quantity from the product's stock
                existingProduct.Stock -= item.Quantity;
                // Update the overall total order price
                totalOrderPrice += TotalPrice;
                // Create a relationship record between the order and product
                orderProducts = new OrderProducts {OID = order.OID, PID = existingProduct.PID, Quantity = item.Quantity  };
                _orderProductsService.AddOrderProducts(orderProducts);
                // Update the product's stock in the database
                _productService.UpdateProduct(existingProduct);
            }
            // Update the total amount of the order
            order.TotalAmount = totalOrderPrice;
            UpdateOrder(order);
        }

``` 

**Issue**
- N+1 Queries x 2 (Validation + Processing)

    - What happens: For each item you call _productService.GetProductByName(...) twice (once during validation, once during processing).

    - Runtime effect:

      1. High latency (many round trips to DB).

      2. Lower throughput under concurrency.

      3. DB connection pool pressure (threads wait for free connections).

    - Why it’s bad: With 20 items, you do 40 product lookups + per-item updates. That’s classic N+1 explosion.
**Solve**
    - Batch Fetch Products Upfront
      - How: Query all products needed in one go before validation/processing.
      - Benefit: Single DB round trip, leverage set-based operations.
      - Example:
        ```csharp
        var productNames = items.Select(i => i.ProductName).ToList();
        var products = _productService.GetProductsByNames(productNames);
        var productDict = products.ToDictionary(p => p.Name, p => p);
        ```
      - Then use productDict in validation and processing loops.

**Notes, There are other issues**

1. Per-Item Writes

    - What happens: ```Each _orderProductsService.AddOrderProducts(...) and _productService.UpdateProduct(...)``` likely emits its own DB command.

    - Runtime effect:

      1. More locks, more log writes, more contention.

      2. Transaction time increases → higher probability of deadlocks/timeouts.

    - Why it’s bad: Databases are fast at set operations, slow at chatty loops.
**Solve**
    - Batch Writes
      - How: Accumulate changes and write in bulk.
      - Benefit: Fewer commands, reduced overhead.
      - Example:
        ```csharp
        var orderProductsList = new List<OrderProducts>();
        foreach (var item in items)
        {
            // ... existing logic ...
            orderProductsList.Add(new OrderProducts { OID = order.OID, PID = existingProduct.PID, Quantity = item.Quantity });
            // Update stock in-memory
            existingProduct.Stock -= item.Quantity;
        }
        _orderProductsService.AddOrderProductsBulk(orderProductsList);
        _productService.UpdateProductsBulk(productsToUpdate);
        ```
2) Missing Explicit Transaction Boundaries

    - What happens: If AddOrder, UpdateOrder, and per-item updates each run in their own implicit transaction, you risk partial commits.
    - Runtime effect:

      1. Inconsistency under failures (order created but stock not reduced for all items).

      2. Harder retries; longer lock durations scattered across statements.

    - Why it’s bad: ACID work should happen in a single, short transaction.
**Solve**
    - Use Explicit Transactions
      - How: Wrap the entire place order logic in a transaction scope.
      - Benefit: Atomicity, consistency, easier error handling.
      - Example:
        ```csharp
        using (var transaction = _dbContext.Database.BeginTransaction())
        {
            try
            {
                // Place order logic here
                transaction.Commit();
            }
            catch
            {
                transaction.Rollback();
                throw;
            }
        }
        ```

3) Stock Races (Concurrency)

    - What happens: Two users can pass validation concurrently and both deduct stock, leading to negative stock or overselling.

    - Runtime effect:

      1. Correctness bug first, then performance issues from retries/compensations.

    - Why it’s bad: Validation uses stale reads; update isn’t guarded by a concurrency check.

**Solve**
    - Use Concurrency Controls
      - How: Implement optimistic concurrency (e.g., row versioning) or pessimistic locks during stock checks and updates.
      - Benefit: Prevents overselling, ensures data integrity.
      - Example:
        ```csharp
        // Optimistic concurrency example
        var product = _dbContext.Products.Single(p => p.PID == existingProduct.PID);
        if (product.Stock < item.Quantity)
            throw new Exception("Insufficient stock");
        product.Stock -= item.Quantity;
        _dbContext.SaveChanges();
        ```

4) Lookup by Product Name

    - What happens: Repeated “get by name” is sensitive to missing indexes and collation issues.

    - Runtime effect:

        1. Table scans → CPU spikes, IO overhead.

    - Why it’s bad: Names aren’t great keys; they change and may be non-unique in the wild.

**Solve**
    - Use IDs for Lookups
      - How: Pass product IDs in OrderItemDTO instead of names.
      - Benefit: Faster lookups, simpler queries.
      - Example:
        ```csharp
        public class OrderItemDTO
        {
            public int ProductId { get; set; }
            public int Quantity { get; set; }
        }
        ```

5) Synchronous IO

    - What happens: Method is void and services appear synchronous.

    - Runtime effect:

        1. Thread-pool threads block waiting on IO → fewer requests served concurrently.

    - Why it’s bad: ASP.NET Core scales better with async IO.

**Solve**
    - Make Async
      - How: Use async/await and return Task.
      - Benefit: Frees up threads during IO waits.
      - Example:
        ```csharp
        public async Task PlaceOrderAsync(List<OrderItemDTO> items, int uid)
        {
            // Async logic here
        }
        ```







