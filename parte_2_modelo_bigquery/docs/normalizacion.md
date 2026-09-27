# Modelo de datos — Smart Solutions Dietta

E-commerce de productos tecnológicos (smartphones, portátiles, periféricos, wearables, audio…) que vende en varios países de Europa. El modelo centraliza la operativa (catálogo, pedidos, envíos, pagos) y permite el análisis del negocio (ingresos, márgenes, segmentación de clientes, valoraciones y tendencias).

Dataset en BigQuery: `smart_solutions_dietta`.

---

## 1. Tablas

### `customers`
| Campo | Tipo | Notas |
|---|---|---|
| customer_id | INT64 | **PK** |
| first_name | STRING | |
| last_name | STRING | |
| email | STRING | único |
| phone | STRING | opcional |
| country | STRING | análisis geográfico |
| city | STRING | análisis geográfico |
| acquisition_channel | STRING | organic, paid_ads, social_media, referral, email |
| registration_date | DATE | |

### `categories`
| Campo | Tipo | Notas |
|---|---|---|
| category_id | INT64 | **PK** |
| name | STRING | |
| description | STRING | |

### `products`
| Campo | Tipo | Notas |
|---|---|---|
| product_id | INT64 | **PK** |
| category_id | INT64 | **FK** → categories |
| name | STRING | |
| price | FLOAT64 | precio de venta actual |
| cost | FLOAT64 | coste (para calcular el margen) |
| stock | INT64 | |
| is_active | BOOL | activo / inactivo |

### `orders`
| Campo | Tipo | Notas |
|---|---|---|
| order_id | INT64 | **PK** |
| customer_id | INT64 | **FK** → customers |
| order_date | TIMESTAMP | |
| status | STRING | pending, confirmed, shipped, delivered, cancelled, returned |
| shipping_address | STRING | dirección de envío de este pedido |
| shipping_city | STRING | |
| shipping_country | STRING | |
| shipped_date | DATE | vacía si aún no se ha enviado |
| delivered_date | DATE | vacía si aún no se ha entregado |

### `order_items`
| Campo | Tipo | Notas |
|---|---|---|
| order_item_id | INT64 | **PK** |
| order_id | INT64 | **FK** → orders |
| product_id | INT64 | **FK** → products |
| quantity | INT64 | |
| unit_price | FLOAT64 | precio en el momento de la compra |
| discount_pct | FLOAT64 | descuento aplicado (0.10 = 10 %) |

### `payments`
| Campo | Tipo | Notas |
|---|---|---|
| payment_id | INT64 | **PK** |
| order_id | INT64 | **FK** → orders |
| payment_method | STRING | credit_card, paypal, bank_transfer, bizum |
| status | STRING | completed, refunded, pending, failed |
| amount | FLOAT64 | importe cobrado |
| payment_date | TIMESTAMP | |

### `reviews`
| Campo | Tipo | Notas |
|---|---|---|
| review_id | INT64 | **PK** |
| order_item_id | INT64 | **FK** → order_items (una valoración por línea) |
| rating | INT64 | 1–5 |
| comment | STRING | opcional |
| review_date | DATE | |

---

## 2. Relaciones y cardinalidades

| Relación | Cardinalidad |
|---|---|
| categories → products | 1 : N |
| customers → orders | 1 : N |
| orders → order_items | 1 : N |
| products → order_items | 1 : N |
| orders → payments | 1 : N |
| order_items → reviews | 1 : 0..1 |

**¿Qué relación hay entre pedidos y productos?** Es **N : M**: un pedido contiene varios productos y un producto aparece en muchos pedidos. Una relación N : M no se puede representar directamente con una FK en ninguna de las dos tablas (habría que guardar listas de ids, lo que rompe la 1NF), así que se resuelve con la **tabla intermedia `order_items`**. Además, esa tabla es el sitio natural para los datos que pertenecen a la combinación pedido–producto: cantidad, precio pagado y descuento.

---

## 3. Normalización

### 1NF — Primera Forma Normal
- Todos los atributos son **atómicos**: nombre y apellido van por separado, la dirección de envío se divide en dirección, ciudad y país, y cada línea de pedido es una fila.
- **No hay grupos repetidos**: los productos de un pedido no se guardan como lista dentro de `orders` (`product_1`, `product_2`…), sino como filas en `order_items`.
- Cada tabla tiene una **clave primaria** que identifica cada fila de forma única.

### 2NF — Segunda Forma Normal
- Cumple 1NF.
- Todas las tablas tienen **clave primaria simple** (un solo campo `*_id`), por lo que no pueden existir dependencias parciales.
- El caso a vigilar es `order_items`: si su clave fuese la compuesta `(order_id, product_id)`, guardar ahí `product_name` o `category_id` sería una **dependencia parcial**, porque dependerían solo de `product_id`. Por eso en `order_items` solo están los atributos que dependen de la línea completa (`quantity`, `unit_price`, `discount_pct`), y los datos del producto se obtienen con JOIN a `products`.

### 3NF — Tercera Forma Normal
- Cumple 2NF.
- **No hay dependencias transitivas**: ningún atributo no clave depende de otro atributo no clave.
  - `products` guarda `category_id`, no el nombre de la categoría (el nombre depende de la categoría, no del producto).
  - `orders` guarda `customer_id`, no los datos del cliente.
  - `reviews` guarda solo `order_item_id`; el cliente y el producto se obtienen a través de `order_items` → `orders`. Guardarlos también en `reviews` sería redundante.
  - El **total del pedido no se almacena**: se calcula a partir de sus líneas (`quantity × unit_price × (1 − discount_pct)`).

---

## 4. Decisiones de diseño

**¿Por qué `unit_price` está en `order_items` y no se lee de `products.price`?**
Porque son datos distintos. `products.price` es el precio **actual** y cambia con el tiempo (ofertas, subidas…). `unit_price` es el precio **que pagó el cliente en ese momento**, un hecho histórico que no debe cambiar. Si se leyera de `products`, al modificar un precio cambiarían los ingresos de todos los pedidos antiguos. No es redundancia, porque no depende del producto sino de la línea de pedido. Por el mismo motivo, la dirección de envío está en `orders`: es la de ese pedido concreto.

**¿Por qué `country` está en `customers` y no en una tabla `countries`?**
Porque no hay ningún otro atributo que dependa del país: no guardamos moneda, prefijo ni región. `country` es un valor atómico que depende únicamente del cliente, así que no genera dependencias transitivas y el modelo cumple 3NF. Una tabla `countries` solo añadiría un JOIN sin aportar información. Tendría sentido crearla si en el futuro hubiera que guardar datos propios de cada país.

**Si en `orders` guardásemos `customer_name` además de `customer_id`, ¿qué forma normal se violaría?**
La **3NF**. Existiría la dependencia transitiva `order_id → customer_id → customer_name`: el nombre depende de otro atributo no clave (`customer_id`), no de la clave del pedido. Consecuencias: el nombre se repetiría en cada pedido del cliente y, si cambiase, habría que actualizar todos sus pedidos; si alguno se quedase sin actualizar, los datos serían inconsistentes (anomalía de actualización).
