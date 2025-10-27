from dataclasses import dataclass, field
from typing import List, Dict, Optional
from abc import ABC, abstractmethod
from datetime import datetime

@dataclass(frozen=True)
class Item:
    # id - внутри системы склада
    warehouse_id: int
    # id - внутри системы поставщика
    provider_id: int
    # название товара
    name: str
    # себестоимость
    cost: float

@dataclass
class Order:
    id: int
    user: 'User'
    items: Dict[Item, int]  # товар : количество
    status: str = field(default="created")  # created, assembling, delivering, delivered, canceled
    time_create: datetime = field(default_factory=datetime.now)
    time_delivery: Optional[datetime] = None
    storekeeper: Optional['Storekeeper'] = None
    courier: Optional['Courier'] = None

class Provider:
    def __init__(self, name: str, stocks: Dict[int, int]):
        self.name = name
        self.stocks = stocks  # ключ: provider_id товара, значение - сколько есть у поставщика
    
    def send_order(self, requested_items: Dict[int, int]) -> Dict[int, int]:
        """
        Приходит запрос от склада.
        Возвращает сколько поставщик может привезти (если нет - сколько может).
        """
        delivered = {}
        for item_id, qty in requested_items.items():
            available = self.stocks.get(item_id, 0)
            deliver_qty = min(qty, available)
            if deliver_qty > 0:
                delivered[item_id] = deliver_qty
                self.stocks[item_id] -= deliver_qty
            else:
                delivered[item_id] = 0
        print(f"[Provider {self.name}] Отправил поставку: {delivered}")
        return delivered
    
    def update_stocks(self, added_items: Dict[int, int]):
        for item_id, qty in added_items.items():
            self.stocks[item_id] = self.stocks.get(item_id, 0) + qty
        print(f"[Provider {self.name}] Обновил стоки: {self.stocks}")

class Worker(ABC):
    def __init__(self, id_: int, name: str):
        self._id = id_
        self._name = name
        self._hours_worked = 0
        self._on_shift = False
    
    @abstractmethod
    def get_order(self, order: Order):
        pass
    
    def get_shift(self, hours: int):
        self._on_shift = True
        self._hours_worked += hours
        print(f"[Работник {self._name}] Получил смену на {hours} часов")
    
    def end_shift(self):
        self._on_shift = False
        salary = self._hours_worked * 300
        print(f"[Работник {self._name}] Смена окончена. Оплата: {salary} руб.")
        self._hours_worked = 0
        return salary

class Courier(Worker):
    def __init__(self, id_: int, name: str):
        super().__init__(id_, name)
        self.current_order: Optional[Order] = None

    def get_order(self, order: Order):
        if self._on_shift and self.current_order is None and order.status == "assembled":
            self.current_order = order
            order.status = "delivering"
            order.courier = self
            print(f"[Курьер {self._name}] Принял заказ #{order.id} на доставку")
            return True
        print(f"[Курьер {self._name}] Не может принять заказ #{order.id}")
        return False
    
    def deliver_order(self):
        if self.current_order:
            print(f"[Курьер {self._name}] Доставляет заказ #{self.current_order.id}")
            self.current_order.status = "del ivered"
            self.current_order.time_delivery = datetime.now()
            self.current_order = None
            print(f"[Курьер {self._name}] Заказ доставлен и курьер вернулся на склад")

class Storekeeper(Worker):
    def __init__(self, id_: int, name: str):
        super().__init__(id_, name)
        self.current_order: Optional[Order] = None
    
    def get_order(self, order: Order):
        if self._on_shift and self.current_order is None and order.status == "created":
            self.current_order = order
            order.status = "assembling"
            order.storekeeper = self
            print(f"[Кладовщик {self._name}] Принял заказ #{order.id} на сборку")
            return True
        print(f"[Кладовщик {self._name}] Не может принять заказ #{order.id}")
        return False
    
    def assemble_order(self):
        if self.current_order:
            print(f"[Кладовщик {self._name}] Собирает заказ #{self.current_order.id}")
            # Здесь логика по сбору товара
            self.current_order.status = "assembled"
            print(f"[Кладовщик {self._name}] Заказ #{self.current_order.id} собран")
            self.current_order = None

class Store:
    def __init__(self, provider: Provider):
        # stocks хранит ключ - производственный warehouse_id, значение - количество на складе
        self.stocks: Dict[int, int] = {}
        self.provider = provider
        self.orders: List[Order] = []
        self.workers: List[Worker] = []
        self.order_id_counter = 0
    
    def send_request(self, desired_items: Dict[int, int]):
        print("[Склад] Отправка запроса поставщику...")
        delivered = self.provider.send_order(desired_items)
        self.update_stocks(delivered)
    
    def update_stocks(self, items: Dict[int, int]):
        for item_id, qty in items.items():
            self.stocks[item_id] = self.stocks.get(item_id, 0) + qty
        print(f"[Склад] Запасы обновлены: {self.stocks}")
    
    def get_worker(self, worker: Worker, hours: int):
        # работник приходит на склад и получает смену
        self.workers.append(worker)
        worker.get_shift(hours)
    
    def create_order(self, user: 'User', requested_items: Dict[Item, int]) -> Order:
        # Сформировать заказ по наличию товара: убрать отсутствующее
        available_items = {}
        for item, qty in requested_items.items():
            stock_qty = self.stocks.get(item.warehouse_id, 0)
            if stock_qty > 0:
                available_items[item] = min(qty, stock_qty)
        order = Order(id=self.order_id_counter, user=user, items=available_items)
        self.orders.append(order)
        self.order_id_counter += 1
        
        # снять со склада товары в стоке (они "заморожены")
        for item, qty in available_items.items():
            self.stocks[item.warehouse_id] -= qty
        
        print(f"[Склад] Создан заказ #{order.id}, товары: {[(i.name, q) for i,q in available_items.items()]}")
        return order
    
    def set_storekeeper(self, order: Order) -> bool:
        # найдем свободного кладовщика на смене
        for w in self.workers:
            if isinstance(w, Storekeeper) and w._on_shift and w.current_order is None:
                accepted = w.get_order(order)
                if accepted:
                    return True
        print("[Склад] Нет свободного кладовщика для заказа")
        return False
    
    def set_courier(self, order: Order) -> bool:
        # найдем свободного курьера
        for w in self.workers:
            if isinstance(w, Courier) and w._on_shift and w.current_order is None:
                accepted = w.get_order(order)
                if accepted:
                    return True
        print("[Склад] Нет свободного курьера для доставки")
        return False

class User:
    def __init__(self, id_: int, name: str, address: str, store: Store):
        self._id = id_
        self._name = name
        self._address = address
        self._store = store
        self.current_order: Optional[Order] = None
    
    def make_order(self, items: Dict[Item, int]):
        print(f"[Пользователь {self._name}] Делает заказ: {[ (item.name, qty) for item, qty in items.items()]}")
        order = self._store.create_order(self, items)
        self.current_order = order
        return order
    
    def take_order(self):
        if self.current_order and self.current_order.status == "delivered":
            print(f"[Пользователь {self._name}] Получил заказ #{self.current_order.id}")
            self.current_order = None
        else:
            print(f"[Пользователь {self._name}] Заказ пока не доставлен")

# Демонстрация работы:

# Инициализируем поставщика с товарами (provider_id: qty)
provider = Provider("GlobalProvider", stocks={101: 100, 102: 50})

# Спецификация товаров
item_apple = Item(warehouse_id=1, provider_id=101, name="Яблоко", cost=10)
item_banana = Item(warehouse_id=2, provider_id=102, name="Банан", cost=15)

# Инициализируем склад
store = Store(provider)

# Склад запрашивает поставку у поставщика
store.send_request({101: 20, 102: 30})

# Инициализируем работников
kurier = Courier(1, "Иван")
klado = Storekeeper(2, "Петр")

# Работники приходят и получают смену
store.get_worker(kurier, 8)
store.get_worker(klado, 8)

# Пользователь регистрируется и делает заказ
user = User(1, "Елена", "ул. Пушкина, д. 10", store)
order = user.make_order({item_apple: 5, item_banana: 10})

# Склад назначает кладовщика на заказ
store.set_storekeeper(order)

# Кладовщик собирает заказ
klado.assemble_order()

# Склад назначает курьера на заказ
store.set_courier(order)

# Курьер доставляет заказ
kurier.deliver_order()

# Пользователь получает заказ
user.take_order()

# Работники заканчивают смену и получают оплату
klado.end_shift()
kurier.end_shift()


