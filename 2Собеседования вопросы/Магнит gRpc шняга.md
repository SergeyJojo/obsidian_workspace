
## **🔥 Разбираем и исправляем реализацию gRPC-сервера для корзины**

### **🛠️ Проблемы в коде**

1. ❌ **Отсутствует проверка статуса корзины** – нельзя добавлять товары, если `Status == "ordered"`.
    
2. ❌ **Не обновляется сумма заказа** (`basket.Total` не пересчитывается).
    
3. ❌ **Добавление дубликатов товаров** – товары должны быть уникальны по `ProductID`.
    
4. ❌ **Метод AddItemAndOrder() совмещает две разные функции** – лучше разделить на **добавление** и **оформление заказа**.
    

---

## **🔹 Правильная реализация**

### **1. Добавляем товар в корзину**

```go
func (bs *BasketServer) AddItem(ctx context.Context, req *AddItemRequest) (*EmptyResponse, error) {
    basket, err := bs.repo.Load(ctx, req.UserID)
    if err != nil {
        return nil, err
    }

    // ❌ Нельзя изменять оформленную корзину
    if basket.Status == "ordered" {
        return nil, status.Errorf(codes.FailedPrecondition, "Корзина уже оформлена и не может быть изменена")
    }

    // ✅ Проверяем, есть ли товар уже в корзине
    found := false
    for i, item := range basket.Items {
        if item.ProductID == req.ProductID {
            basket.Items[i].Count += req.Count
            found = true
            break
        }
    }

    // ✅ Если товара нет – добавляем новый
    if !found {
        basket.Items = append(basket.Items, BasketItem{
            BasketID:  basket.ID,
            ProductID: req.ProductID,
            Count:     req.Count,
            Price:     req.Price,
        })
    }

    // ✅ Пересчитываем общую сумму
    basket.Total = calculateTotal(basket.Items)

    // ✅ Сохраняем обновлённую корзину
    if err := bs.repo.Save(ctx, basket); err != nil {
        return nil, err
    }

    return &EmptyResponse{}, nil
}

// Функция для пересчёта общей суммы
func calculateTotal(items []BasketItem) uint64 {
    var total uint64
    for _, item := range items {
        total += item.Count * item.Price
    }
    return total
}
```

---

### **2. Оформление корзины**

```go
func (bs *BasketServer) OrderBasket(ctx context.Context, req *OrderRequest) (*EmptyResponse, error) {
    basket, err := bs.repo.Load(ctx, req.UserID)
    if err != nil {
        return nil, err
    }

    // ❌ Проверяем, не оформлена ли корзина уже
    if basket.Status == "ordered" {
        return nil, status.Errorf(codes.FailedPrecondition, "Корзина уже оформлена")
    }

    // ✅ Меняем статус корзины
    basket.Status = "ordered"

    // ✅ Сохраняем изменения
    if err := bs.repo.Save(ctx, basket); err != nil {
        return nil, err
    }

    // ✅ Отправляем сообщение в брокер
    if err := bs.producer.SendMessage(ctx, *basket); err != nil {
        return nil, err
    }

    return &EmptyResponse{}, nil
}
```

---

### **3. Обновленный интерфейс gRPC**

```proto
syntax = "proto3";

service BasketService {
    rpc AddItem (AddItemRequest) returns (EmptyResponse);
    rpc OrderBasket (OrderRequest) returns (EmptyResponse);
}

message AddItemRequest {
    uint64 user_id = 1;
    uint64 product_id = 2;
    uint64 price = 3;
    uint64 count = 4;
}

message OrderRequest {
    uint64 user_id = 1;
}

message EmptyResponse {}
```

---

## **🔥 Итог**

Теперь наш gRPC-сервис: ✔ **Корректно добавляет товары, не дублируя их**  
✔ **Пересчитывает итоговую сумму корзины**  
✔ **Не позволяет менять оформленную корзину**  
✔ **Отправляет сообщение в брокер при оформлении**

Что думаешь? Нужно ли добавить коды ошибок или логику откатов (транзакции)? 🚀