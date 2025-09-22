# Analysis performance issues Bottlencks_N_plus_One_Issues 
### note : This case will discuess in case of **place order**

# Overveiw
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



### N + 1 issue
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






