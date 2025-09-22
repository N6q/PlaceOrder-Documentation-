# 🛒 PlaceOrder Function Documentation

## 📌 Overview
This repository documents and improves the `PlaceOrder` method for an **E-Commerce System**.  
It shows the **old implementation**, identifies **performance issues**, and proposes an **optimized version** with transactions, batch updates, and custom exceptions.

---

## 1️⃣ Old Code (Original Implementation)

```csharp
public void PlaceOrder(List<OrderItemDTO> items, int uid)
{
    Product existingProduct = null;
    decimal TotalPrice, totalOrderPrice = 0;
    OrderProducts orderProducts = null;

    for (int i = 0; i < items.Count; i++)
    {
        TotalPrice = 0;
        existingProduct = _productService.GetProductByName(items[i].ProductName);
        if (existingProduct == null)
            throw new Exception($"{items[i].ProductName} not Found");
        if (existingProduct.Stock < items[i].Quantity)
            throw new Exception($"{items[i].ProductName} is out of stock");
    }

    var order = new Order { UID = uid, OrderDate = DateTime.Now, TotalAmount = 0 };
    AddOrder(order);

    foreach (var item in items)
    {
        existingProduct = _productService.GetProductByName(item.ProductName);
        TotalPrice = item.Quantity * existingProduct.Price;
        existingProduct.Stock -= item.Quantity;
        totalOrderPrice += TotalPrice;

        orderProducts = new OrderProducts { OID = order.OID, PID = existingProduct.PID, Quantity = item.Quantity };
        _orderProductsService.AddOrderProducts(orderProducts);
        _productService.UpdateProduct(existingProduct);
    }

    order.TotalAmount = totalOrderPrice;
    UpdateOrder(order);
}
```

### 🚨 Issues in Old Code
- **Duplicate DB lookups** (fetch by name twice).  
- **Multiple DB calls inside loops** (N+1 problem).  
- **No transaction safety** (risk of inconsistent data).  
- **Uses ProductName** instead of ProductId.  
- **Generic exceptions** → unclear error messages.  

---

## 2️⃣ Improved Code (Optimized Implementation)

```csharp
public void PlaceOrder(List<OrderItemDTO> items, int uid)
{
    if (items == null || items.Count == 0)
        throw new ArgumentException("Order must contain at least one item.");

    using var transaction = _dbContext.Database.BeginTransaction();
    try
    {
        var productIds = items.Select(i => i.ProductId).ToList();
        var products = _productService.GetProductsByIds(productIds)
                                      .ToDictionary(p => p.PID, p => p);

        foreach (var item in items)
        {
            if (!products.TryGetValue(item.ProductId, out var product))
                throw new ProductNotFoundException($"Product ID {item.ProductId} not found.");

            if (product.Stock < item.Quantity)
                throw new OutOfStockException($"{product.Name} is out of stock. Requested {item.Quantity}, Available {product.Stock}");
        }

        var order = new Order { UID = uid, OrderDate = DateTime.Now, TotalAmount = 0 };
        AddOrder(order);

        decimal totalOrderPrice = 0;
        var orderProductsList = new List<OrderProducts>();

        foreach (var item in items)
        {
            var product = products[item.ProductId];
            decimal totalPrice = item.Quantity * product.Price;
            totalOrderPrice += totalPrice;
            product.Stock -= item.Quantity;

            orderProductsList.Add(new OrderProducts
            {
                OID = order.OID,
                PID = product.PID,
                Quantity = item.Quantity
            });
        }

        _dbContext.Products.UpdateRange(products.Values);
        _dbContext.OrderProducts.AddRange(orderProductsList);

        order.TotalAmount = totalOrderPrice;
        UpdateOrder(order);

        _dbContext.SaveChanges();
        transaction.Commit();
    }
    catch
    {
        transaction.Rollback();
        throw;
    }
}
```

### ✅ Key Improvements
- **Single DB fetch** → all products retrieved at once.  
- **Transaction safety** → guarantees atomicity.  
- **Batch updates** with `UpdateRange` and `AddRange`.  
- **Uses ProductId** instead of ProductName.  
- **Custom exceptions** → clear error handling.  

---

## 3️⃣ Comparison Table

| Aspect           | Old Code                        | Improved Code                 |
|------------------|---------------------------------|--------------------------------|
| DB Queries       | 2× per product + updates        | 1 query + batch update         |
| Transaction      | ❌ None                         | ✅ Full transaction            |
| Scalability      | Poor (N+1 queries)              | High (bulk ops, efficient)     |
| Maintainability  | Hard (duplicate, generic errors)| Clean (custom exceptions)      |
| Correctness      | Uses ProductName (risk)         | Uses ProductId (safe, unique)  |


---

## 👨‍💻 Author
Built with ❤️ by Samir.
