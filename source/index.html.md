---
title: Revo API Factoring Documentation

language_tabs: # must be one of https://git.io/vQNgJ
  - ruby--tab: Ruby
  - java: Java

toc_footers:
- <a href='https://revo.ru/'>Revo.ru</a>
- <a href='https://revo.ru/API/en'>API Documentation in English</a>

#includes:
#  - errors

search: true
---

# Введение

API Factoring реализовано на протоколе HTTPS на основе JSON запросов.

Документация состоит из 4 основных частей:

* Описание авторизации, методов API и кодов ошибок.
* <a href="#db947828e5">Руководство по тестированию</a>.
* <a href="#6961794a3c">Описание возможных схем взаимодействия</a>.
* <a href="#guides">Руководство по реализации основных элементов</a>.

# Авторизация

## Базовые URL адреса

```javascript
BASE_URL = "https://r.revoplus.ru/"
BASE_URL = "https://demo.revoplus.ru/"
```

1. Для взаимодействия с сервисами Рево используются 2 базовых адреса:
 * https://r.revoplus.ru/ - адрес `production` сервиса.
 * https://demo.revoup.ru/ - адрес `demo` сервиса.
2. `BASE_URL` - переменная обозначающая базовый адрес.

<aside class="notice">
Подключение должно производиться только по протоколу HTTPS  - при попытке подключения по HTTP будет возникать 404 ошибка.
</aside>

## Параметры авторизации

> Пример параметров

```javascript
secret_key = "098f6bcd4621d373cade4e832627b4f6"
STORE_ID = 12
```

1. На стороне Рево формируются уникальный идентификатор магазина и секретный ключ, которые передаются партнеру:
 * `store_id` - уникальный идентификатор магазина. Для одного партнера может быть сформировано несколько уникальных идентификаторов.
 * `secret_key` - секретный ключ, который используется при формировании электронно-цифровой подписи для аутентификации (проверки подлинности) параметров запроса с целью защиты формы от запуска сторонними лицами. Длина ключа от 8 байт. Алгоритм шифрования SHA1.
2. Для авторизации партнер отправляет `POST` запрос, используя <a href="#1c37860b3b">цифровую подпись</a> `signature` и уникальный идентификатор магазина `store_id`.
3. Примеры URL запросов можно посмотреть в разделе <a href="#api"> Методы API</a>.

## Принцип формирования цифровой подписи

> Алгоритм формирования цифровой подписи

```ruby--tab
require 'digest/sha1'
secret_key = '098f6bcd4621d373cade4e832627b4f6'
data = "{\"callback_url\":\"https://shop.ru/revo/decision\",
\"redirect_url\":\"https://shop.ru/revo/redirect\",
\"current_order\":{\"sum\":\"7500.00\",\"order_id\":\"R001233\"},
\"primary_phone\":\"9268180621\"}"
SIGNATURE = Digest::SHA1.hexdigest(data + secret_key)
```

```java
import java.io.UnsupportedEncodingException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Formatter;

public class Main {

    static String secret_key = "098f6bcd4621d373cade4e832627b4f6"; // Это пример
    static String data = "{\"callback_url\":\"https://shop.ru/revo/decision\",\"redirect_url\":\"https://shop.ru/revo/redirect\",\"current_order\":{\"sum\":\"7500.00\",\"order_id\":\"R001233\"},\"primary_phone\":\"9268180621\"}";

    public static void main(String[] args) {

        String signature = encryptPassword(data + secret_key); // Тут всегда будет 40 символов по SHA1
        System.out.println(signature);
    }

    private static String encryptPassword(String password) {
        String sha1 = "";
        try {
            MessageDigest crypt = MessageDigest.getInstance("SHA-1");
            crypt.reset();
            crypt.update(password.getBytes("UTF-8"));
            sha1 = byteToHex(crypt.digest());
        } catch(NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch(UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return sha1;
    }

    private static String byteToHex(final byte[] hash) {
        Formatter formatter = new Formatter();
        for (byte b : hash) {
            formatter.format("%02x", b);
        }
        String result = formatter.toString();
        formatter.close();
        return result;
    }
}
```

К строке `data` в формате json добавляется секретный ключ `secret_key`. К получившейся строке применяется алгоритм SHA1, в результате формируется цифровая подпись `signature`.

<aside class="notice">
Необходимо обратить внимание, что при формировании signature длина всегда будет 40 символов по SHA1.
</aside>

# Методы API

## Registration

```ruby
POST BASE_URL/factoring/v1/limit/auth?store_id=STORE_ID&signature=SIGNATURE
```

Метод возвращает ссылку на iFrame для получения лимита. По завершению формы на адрес указанный в `callback_url` отправляется <a href="#callback_url2">json ответ</a> с результатом решения по лимиту клиента.

В зависимости от информации, которая есть о пользователе в системе Рево, форма будет иметь различное число шагов (для этого нужно передавать `primary_phone`):

* Если номер телефона клиента не найден в базе Рево (новый клиент), либо расчёт лимита ещё не производился, то форма будет состоять из 2 шагов: регистрации (расчёта лимита) и аутентификации по смс.
* Если номер телефона клиента найден в базе Рево (повторный клиент) и клиенту уже рассчитан лимит, то форма будет состоять из 1 шага: аутентификации по смс.

<aside class="success">
Если клиент уже заполнял личные данные на сайте партнёра, их следует передать в запросе для автозаполнения соответствующих полей формы.
</aside>

### Parameters

> Пример запроса в формате json

```jsonnet
{
  callback_url: "https://shop.ru/revo/decision",
  redirect_url: "https://shop.ru/revo/redirect",
  primary_phone: "9268180621",
  primary_email : "ivan@gmail.com",
  current_order: {
    order_id: "R001233"
  }
  person: {
    first_name: "Петр",
    surname: "Чернышев",
    patronymic: "Александрович",
    birth_date: "15.01.1975",
  }
}
```

| | | |
-:|-:|:-|:-
 |**callback_url**<br> <font color="#939da3">string</font> | <td colspan="2"> URL для ответа от Рево по решению для клиента.
 |**redirect_url**<br> <font color="#939da3">string</font>	| <td colspan="2"> URL для редиректа после нажатия на кнопку/ссылку в форме Рево "Вернуться в интернет магазин".
 |**current_order**<br> <font color="#939da3">object</font> | <td colspan="2"> Объект, содержащий информацию о заказе.
<td colspan="2" style="text-align:right">**order_id**<br> <font color="#939da3">string</font> | | Уникальный номер заказа. Не более 255 символов.
 |**primary_phone**<br> <font color="#939da3">integer, *optional*</font> | <td colspan="2"> Номер телефона клиента 10 цифр (без кода страны).
 |**primary_email**<br> <font color="#939da3">string, *optional*</font> | <td colspan="2"> Email клиента.
 |**person**<br> <font color="#939da3">object, *optional*</font> | <td colspan="2"> Объект, содержащий информацию о клиенте.
  <td colspan="2" style="text-align:right">**first_name**<br> <font color="#939da3">string, *optional*</font> | | Имя клиента.
  <td colspan="2" style="text-align:right">**surname**<br> <font color="#939da3">sring, *optional*</font> | | Фамилия клиента.
  <td colspan="2" style="text-align:right">**patronymic**<br> <font color="#939da3">string, *optional*</font> | | Отчество клиента.
  <td colspan="2" style="text-align:right">**birth_date**<br> <font color="#939da3">object, *optional*</font> | | Дата рождения клиента в формате `dd.mm.yyyy`.

### Response Parameters

> Пример ответа для успешной аутентификации.

```jsonnet
{
  status: 0,
  message: "Payload valid",
  iframe_url: "https://r.revoplus.ru/form/v1/af45ef12f4233f"
}
```

 | |
-:|:-
**status** <br> <font color="#939da3">integer</font> | Код ответа.
**message** <br> <font color="#939da3">string</font> | Короткое текстовое описание ответа.
**iframe_url** <br> <font color="#939da3">string</font>	| Cсылка на сгенерированный iFrame.

<a name="callback_url"></a>
### callback parameters

> Пример callback-а при успешном завершении формы:

```jsonnet
{
  order_id: 32423,
  decision: "approved",
  amount: 5000,
  overlimit: "true",
  mobile_phone: '89262341793',
  email: 'ivan@gmail.com',
  loan_id: 3216964
}
```

 | |
-:|:-
**order_id** <br> <font color="#939da3">string</font> | Уникальный номер заказа. Не более 255 символов.
**decision** <br> <font color="#939da3">string</font> | Решение по выдаче рассрочки. При положительном решении - значение `approved`. При отрицательном решении - `declined`.
**amount** <br> <font color="#939da3">float</font> | Сумма в рублях с копейками.
**overlimit** <br> <font color="#939da3">integer</font> | Срок рассрочки в месяцах.
**mobile_phone** <br> <font color="#939da3">integer</font> | Номер телефона клиента 10 цифр (без кода страны).
**email** <br> <font color="#939da3">string</font> | Номер телефона клиента 10 цифр (без кода страны).
**loan_id**  <br> <font color="#939da3">integer</font> | Уникальный номер заказа в системе Рево.

<aside class="success">
При `decision` равном `declined` значение `amount` будет нулевое.
</aside>

## Limit

```ruby
POST BASE_URL/api/external/v1/client/limit?store_id=STORE_ID&signature=SIGNATURE
```

Метод для получения суммы лимита клиента по номеру его телефона. Для новых клиентов получить информацию о лимите только по номеру телефона нельзя.

### Parameters

> Пример запроса в формате json

```jsonnet
{
  client:
  {
    mobile_phone: "9031234567"
  }
}
```

 | |
-:|:-
**client** <br> <font color="#939da3">object</font> | Объект, содержащий информацию о клиенте.
**mobile_phone** <br> <font color="#939da3">integer</font> | Номер телефона клиента 10 цифр (без кода страны).

### Response Parameters

> Пример ответа, когда клиент найден в базе

```jsonnet
{
    meta:
    {
      status: 0,
      message: "Payload valid"
    }
    client:
    {
      mobile_phone: "9031234567",
      limit_amount: "9500.00",
      status: "active"
    }
}
```

> Пример ответа, когда клиент найден в базе, но выдача займа невозможна

```jsonnet
{
    meta:
    {
      status: 0,
      message: "Payload valid"
    }
    client:
    {
      mobile_phone: "9031234567",
      limit_amount: "6700.00",
      status: "inactive"
    }
}
```

> Пример ответа, когда клиент не найден в базе

```jsonnet
{
    meta:
    {
      status: 0,
      message: "Payload valid"
    }
    client:
    {
      mobile_phone: "9031234567",
      limit_amount: "0.00",
      status: "new"
    }
}
```

| | | |
-:|-:|:-|:-
 |**status**<br> <font color="#939da3">integer</font> | <td colspan="2"> Код ответа.
 |**message**<br> <font color="#939da3">string</font> | <td colspan="2"> Короткое текстовое описание ответа.
 |**client**<br> <font color="#939da3">object</font> | <td colspan="2"> Объект, содержащий информацию о клиенте.
 <td colspan="2" style="text-align:right">**mobile_phone**<br> <font color="#939da3">integer</font> | | Номер телефона клиента 10 цифр (без кода страны).
 <td colspan="2" style="text-align:right">**limit_amount**<br> <font color="#939da3">float</font> | | Лимит средств, доступных клиенту, в рублях с копейками.
 <td colspan="2" style="text-align:right">**status**<br> <font color="#939da3">string</font> | | Статус пользователя. Возможные значения:<br>`active` - пользователю доступна услуга оплаты частями на сумму `limit_amount`;<br>`inactive` - пользователю не доступна услуга оплаты частями;<br>`new` - новый пользователь, которому доступна услуга оплаты частями на сумму `limit_amount`. ???????схуяли???

## Checkout
```ruby
POST BASE_URL/factoring/v1/precheck/auth?store_id=STORE_ID&signature=SIGNATURE
```

Метод возвращает ссылку на iFrame для оформления заказа клиента. По завершению формы на адрес указанный в `callback_url` отправляется <a href="#callback_url">json ответ</a> с результатом оформления.

В зависимости от информации, которая есть о пользователе в системе Рево, форма будет иметь различное число шагов (для этого нужно передавать `primary_phone`):

* Если номер телефона клиента не найден в базе Рево (новый клиент), либо расчёт лимита ещё не производился, то форма будет состоять из 3 шагов: регистрации (расчёта лимита), аутентификации по смс и оформления заказа.
* Если номер телефона клиента найден в базе Рево (повторный клиент) и клиенту уже расчитан лимит, то форма будет состоять из 2 шагов: аутентификации по смс и оформления заказа.

<aside class="success">
Если клиент уже заполнял личные данные на сайте партнёра, их следует передать в запросе для автозаполнения соответствующих полей формы.
</aside>

### Parameters

> Пример запроса в формате json

```jsonnet
{
  callback_url: "https://shop.ru/revo/decision",
  redirect_url: "https://shop.ru/revo/redirect",
  primary_phone: "9268180621",
  primary_email : "ivan@gmail.com",
  current_order: {
    order_id: "R001233",
    valid_till: "21.04.2017 12:08:01+03:00",
    term: 3,
    amount: 6700.00,
  }
  person: {
    first_name: "Петр",
    surname: "Чернышев",
    patronymic: "Александрович",
    birth_date: "15.01.1975",
  }
  cart_items: [{
    sku: '1', name: 'prod9', price: '12', quantity: '1' },
  { sku: '2', name: 'prod3', price: '7', sale_price: '5', quantity: '1' }],
  skip_result_page: true,
}
```

| | | |
-:|-:|:-|:-
 |**callback_url** <br> <font color="#939da3">string</font> |<td colspan="2"> URL для ответа от Рево по решению для клиента.
 |**redirect_url** <br> <font color="#939da3">string</font>	|<td colspan="2"> URL для редиректа после нажатия на кнопку/ссылку в форме Рево "Вернуться в интернет магазин".
 |**current_order** <br> <font color="#939da3">object</font> |<td colspan="2"> Объект, содержащий информацию о заказе.
<td colspan="2" style="text-align:right"> **order_id**<br> <font color="#939da3">string</font> | | Уникальный номер заказа. Не более 255 символов.
<td colspan="2" style="text-align:right"> **valid_till**<br> <font color="#939da3">String, *optional*</font> | | Срок, в течении которого заказ считается актуальным (срок холдирования средств). По истечении срока заказ отменяется. Формат: `dd.mm.yyyy hh:mm:ss+hh:mm`, где после  "+" указывается часовой пояс относительно GMT. По умолчанию - 24 часа.
 <td colspan="2" style="text-align:right"> **term**<br> <font color="#939da3">integer, *optional*</font> | | Срок рассрочки в месяцах.
 <td colspan="2" style="text-align:right"> **amount**<br> <font color="#939da3">float</font> | | Сумма заказа в рублях с копейками.
 |**primary_phone**<br> <font color="#939da3">integer, *optional*</font> |<td colspan="2"> Номер телефона клиента 10 цифр (без кода страны).
 |**primary_email**<br> <font color="#939da3">string, *optional*</font> |<td colspan="2"> Email клиента.
 |**person**<br> <font color="#939da3">object, *optional*</font> |<td colspan="2"> Объект, содержащий информацию о клиенте.
 <td colspan="2" style="text-align:right"> **first_name**<br> <font color="#939da3">string, *optional*</font> | | Имя клиента.
 <td colspan="2" style="text-align:right"> **surname**<br> <font color="#939da3">sring, *optional*</font> | | Фамилия клиента.
 <td colspan="2" style="text-align:right"> **patronymic**<br> <font color="#939da3">string, *optional*</font> | | Отчество клиента.
 <td colspan="2" style="text-align:right"> **birth_date**<br> <font color="#939da3">string, *optional*</font> | | Дата рождения клиента в формате `dd.mm.yyyy`.
 |**cart_items**<br> <font color="#939da3">object, *optional*</font> |<td colspan="2"> Объект, содержащий информацию о заказе.
 <td colspan="2" style="text-align:right"> **sku**<br> <font color="#939da3">string, *optional*</font> | | Складская учётная единица (stock keeping unit).
 <td colspan="2" style="text-align:right"> **name**<br> <font color="#939da3">string, *optional*</font> | | Наименование товара.
 <td colspan="2" style="text-align:right"> **price**<br> <font color="#939da3">float, *optional*</font> | | Цена товара.
 <td colspan="2" style="text-align:right"> **sale_price**<br> <font color="#939da3">float, *optional*</font> | | Цена товара со скидкой (если есть).
 <td colspan="2" style="text-align:right"> **quantity**<br> <font color="#939da3">integer</font> | | Количество товара.
 |**skip_result_page**<br> <font color="#939da3">bool, *optional*</font> |<td colspan="2"> Флаг, который определяет будет ли отображена страница с результатом оформления в iFrame: `true` - по успешному завершению оформления сразу происходит редирект по `redirect_url`; `false` - по успешному завершению оформления будет отображено окно с результатом. По умолчанию - `false`.

 <aside class="success">
 При передаче информации о предоплате клиента следует также использовать `skip_result_page`: выставлять `true` при необходимости клиентом совершить предоплату и передавать в `callback_url` адрес страницы предоплаты; выставлять `false` при наличии предоплаты.
 </aside>

### Response Parameters

> Пример ответа при успешной аутентификации.

```jsonnet
{
  status: 0,
  message: "Payload valid",
  iframe_url: "https://revo.ru/factoring/v1/form/6976174c5b6a1bb089d15b80e0a6afc62d4283fe"
}
```

<!-- > Пример ответа при неуспешной аутентификации.

```jsonnet
{
  status: 20,
  message: "Order order_id missing",
}
``` -->

 | |
-:|:-
**status** <br> <font color="#939da3">integer</font> | Код ответа.
**message** <br> <font color="#939da3">string</font> | Короткое текстовое описание ответа.
**iframe_url** <br> <font color="#939da3">string</font>	| Cсылка на сгенерированный iFrame.

<a name="callback_url"></a>
### Callback parameters

> Пример callback-а при успешном оформлении товара:

```jsonnet
{
  order_id: "R107356",
  decision: "approved",
  amount: "6700.00",
  term: 3,
  client:
  {
    primary_phone: "8880010203"
    full_name: "Иванов Иван Иванович"
  },
  schedule:
  [{
    date: "01.01.2018",
    amount: "2933.33"
  },
  {
    date: "01.02.2018",
    amount: "2933.33"
  },
  {
    date: "01.03.2018",
    amount: "2933.33"
  }]
}
```

| | | |
-:|-:|:-|:-
 |**order_id** <br> <font color="#939da3">string</font> |<td colspan="2"> Уникальный номер заказа. Не более 255 символов.
 |**decision** <br> <font color="#939da3">string</font> |<td colspan="2"> Решение по выдаче рассрочки. При положительном решении - значение `approved`. При отрицательном решении - `declined`.
 |**amount** <br> <font color="#939da3">float</font> |<td colspan="2"> Сумма в рублях с копейками.
 |**term** <br> <font color="#939da3">integer</font> |<td colspan="2"> Срок рассрочки в месяцах.
 |**client** <br> <font color="#939da3">object</font> |<td colspan="2"> Объект, содержащий информацию о клиенте.
<td colspan="2" style="text-align:right">**primary_phone** <br> <font color="#939da3">integer</font> | | Номер телефона клиента 10 цифр (без кода страны).
<td colspan="2" style="text-align:right">**full_name**  <br> <font color="#939da3">string</font> | | ФИО через пробел.
 |**schedule** <br> <font color="#939da3">object</font> |<td colspan="2"> Объект, содержащий информацию о графике платежей.
<td colspan="2" style="text-align:right">**date** <br> <font color="#939da3">string</font> | | Дата платежа в формате `dd.mm.yyyy`.
<td colspan="2" style="text-align:right">**amount** <br> <font color="#939da3">float</font> | | Сумма платежа в рублях с копейками.

<aside class="success">
При `decision` равном `declined` значение `amount` будет нулевое, а в `schedule` будет пустой массив.
</aside>

## Status

```ruby
POST BASE_URL/factoring/v1/status?store_id=STORE_ID&signature=SIGNATURE
```

Метод возвращает информацию по статусу заказа.

### Parameters

> Пример запроса в формате json

```jsonnet
{
  "order_id": "R107356"
}
```

 | |
-:|:-
**order_id** <br> <font color="#939da3">string</font> | Уникальный номер заказа. Не более 255 символов.

### Response Parameters

> Пример ответа, когда срок актуальности заказа ещё не вышел, решение по заказу (approved или declined) уже есть

```jsonnet
{
  status: 0,
  message: "Payload valid",
  current_order:
  {
    order_id: "R107356",
    expired: false,
    decision: "approved",
    amount: "6700.00",
    discount_amount: "6100.00",
    term: 3
  }
}
```

> Пример ответа, когда срок актуальности заказа ещё не вышел, решения по займу нет (клиент не прошёл процесс до конца)

```jsonnet
{
  status: 0,
  message: "Payload valid",
  current_order:
  {
    order_id: "R107356",
    expired: "false"
  }
}
```

> Пример ответа, когда срок актуальности заказа истёк, решение по займу (Approve или declined) уже есть:

```jsonnet
{
  status: 0,
  message: "Payload valid",
  current_order:
  {
    order_id: "R107356",
    expired: "true",
    decision: "approved",
    amount: "6700.00",
    discount_amount: "6100.00",
    term: 3
  }
}
```

> Пример ответа, когда срок актуальности заказа истёк, решения по займу нет (клиент не прошел процесс до конца)

```jsonnet
{
  status: 0,
  message: "Payload valid",
  current_order:
  {
    order_id: "R107356",
    expired: "true"
  }
}
```

| | | |
-:|-:|:-|:-
 |**status** <br> <font color="#939da3">integer</font> | <td colspan="2"> Код ответа.
 |**message** <br> <font color="#939da3">string</font> | <td colspan="2"> Короткое текстовое описание ответа.
 |**current_order** <br> <font color="#939da3">object</font> | <td colspan="2"> Объект, содержащий информацию о заказе.
 <td colspan="2" style="text-align:right">**order_id** <br> <font color="#939da3">string</font> | | Уникальный номер заказа. Не более 255 символов.
 <td colspan="2" style="text-align:right">**expired** <br> <font color="#939da3">bool</font> | | Флаг, отображающий статус актуальности заказа. Для актуальных заказов значение `true`.
 <td colspan="2" style="text-align:right">**decision** <br> <font color="#939da3">string</font> | | Решение по выдаче рассрочки. При положительном решении значение `approved`. При отрицательном решении - `declined`.
 <td colspan="2" style="text-align:right">**amount** <br> <font color="#939da3">float</font> | | Сумма в рублях с копейками. (СТРОКА!!! сейчас)
 <td colspan="2" style="text-align:right">**discount_amount** <br> <font color="#939da3">bool</font> | | ???
 <td colspan="2" style="text-align:right">**term** <br> <font color="#939da3">integer</font> | | Срок рассрочки в месяцах.

## Change

```ruby
POST BASE_URL/factoring/v1/precheck/change?store_id=STORE_ID&signature=SIGNATURE
```

Метод для изменения суммы уже созданного заказа.

<aside class="warning">
Находится в разработке.
</aside>

## Cancel

```ruby
POST BASE_URL/factoring/v1/precheck/cancel?store_id=STORE_ID&signature=SIGNATURE
```

Метод для отмены заказа. При отмене у клиента разблокируются ранее захолдированные средства.

### Parameters

> Пример запроса в формате json

```jsonnet
{
  order_id: "R107356",
}
```

 | |
-:|:-
**order_id** <br> <font color="#939da3">string</font> | Уникальный номер заказа. Не более 255 символов.

### Response Parameters

> Пример ответа, когда файл успешно подгружен

```jsonnet
{
  status: 0,
  message: "Payload valid",
}
```

 | |
-:|:-
**status** <br> <font color="#939da3">integer</font> | Код ответа.
**message** <br> <font color="#939da3">string</font> | Короткое текстовое описание ответа.

## Finish

```ruby
POST BASE_URL/factoring/v1/precheck/finish?store_id=STORE_ID&signature=SIGNATURE
```

Метод для финализации заказа путём передачи договора купли-продажи на обслуживание в Рево.

<aside class="notice">
При попытке финализации заявки с истекшим сроком `valid_till`, будет вызван метод `cancel`.
</aside>

### Parameters

> Пример запроса в формате json

```jsonnet
{
  order_id: "R107356",
  amount: "6700.00",
  check_number: 'ZDDS3123F'
}
@FILES'check'
```

 | |
-:|:-
**order_id** <br> <font color="#939da3">string</font> | Уникальный номер заказа. Не более 255 символов.
**amount** <br> <font color="#939da3">float</font> | Сумма в рублях с копейками.
**check_number** <br> <font color="#939da3">string</font> | Номер фискального документа (например, номер чека).
**FILES'check'** <br> <font color="#939da3">path?</font> | Фискальный документ ??? типы файлов?

### Response Parameters

> Пример ответа, когда файл успешно подгружен

```jsonnet
{
  status: 0,
  message: "Payload valid",
}
```

 | |
-:|:-
**status** <br> <font color="#939da3">integer</font> | Код ответа.
**message** <br> <font color="#939da3">string</font> | Короткое текстовое описание ответа.

## Return

```ruby
POST BASE_URL/factoring/v1/return?store_id=STORE_ID&signature=SIGNATURE
```

Метод для осуществления процедуры возврата заказа.

### Parameters

> Пример запроса в формате json

```jsonnet
{
  order_id: "R001233",
  sum: "2010.00"
}
```

 | |
-:|:-
**order_id** <br> <font color="#939da3">string</font> | Уникальный номер заказа. Не более 255 символов.
**sum** <br> <font color="#939da3">float</font> | Сумма возврата в рублях с копейками. Возврат может быть как полным, так и частичным.

### Response Parameters

> Пример ответа при успешной обработке запроса на возврат

```jsonnet
{
  status: "0",
  message: "Payload valid"
}
```

> Пример ответа при неуспешной обработке запроса на возврат

```jsonnet
{
  status: "10",
  message: "JSON decode error"
}
```

 | |
-:|:-
**status** <br> <font color="#939da3">integer</font> | Код ответа.
**message** <br> <font color="#939da3">string</font> | Короткое текстовое описание ответа.

# Коды ошибок

Код | Описание
-:|:-
**0**  | Payload valid
**10**  | JSON decode error
**20**  | Order `order_id` missing
**21**  | Wrong order `order_id` format
**22**  | Order exists
**24**  | Order with specified id not found
**30**  | Wrong order `order_sum` format
**40**  | Order `callback_url` missing
**32**  | Order amount is different from the amount specified before
**41**  | Order `redirect_url` missing
**50**  | Store id is missing
**51**  | Store not found
**60**  | `Signature` missing
**61**  | `Signature` wrong
**70**  | Phone number is different
**80**  | Unable to finish - order is already finished/canceled
**81**  | Unable to cancel - order is already finished/canceled
**100** | At the moment the server cannot process your request

# Тестирование и отладка

Тестирование и отладка интеграции производятся на demo сервере (https://demo.revoplus.ru). При заполнении номера телефона в анкете рекомендуется использовать несуществующий префикс оператора 888, чтобы sms сообщения не отправлялись реальным людям. На production сервере использовать такой префикс нельзя.

Все коды подтверждения и пин-коды `8888`.

1) префиксы для отказных клиентов

2) для тестирования на проде - код задается персонаяльно магазину (не всегда 8888)

3) в анкете идет проверка по ФИО+ДР или номер паспорта на совпадение с существующим клиентом, поэтому при тестировании необходимо вводить различные данные

# Описание iFrame REVO

ФИО - кириллица

возраст - не менее 18 лет

серия паспорта - ????

# Схемы взаимодействия

Возможны следующие схемы взаимодействия:

<table border="0" align="center">
  <tr>
    <th rowspan="2">Схема</th>
    <th colspan="3" align="center">Предоплата</th>
    <th rowspan="2">Изменение заказа</th>
  </tr>
  <tr>
    <th>Партнёр</th>
    <th>Рево</th>
    <th>Отсутствует</th>
  </tr>
  <tr>
    <td>Факторинг 1</td>
    <td>х</td>
    <td></td>
    <td></td>
    <td>х</td>
  </tr>
  <tr>
    <td>Факторинг 2</td>
    <td></td>
    <td>x</td>
    <td></td>
    <td>х</td>
  </tr>
  <tr>
    <td><a href="http://localhost:4567/#3">Факторинг 3</a></td>
    <td></td>
    <td></td>
    <td>x</td>
    <td>х</td>
  </tr>
  <tr>
    <td>Факторинг 4</td>
    <td>х</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>Факторинг 5</td>
    <td></td>
    <td>x</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>Факторинг 6</td>
    <td></td>
    <td></td>
    <td>x</td>
    <td></td>
  </tr>
</table>

## Факторинг 1

## Факторинг 2

## Факторинг 3

1. Клиент оформляет покупку на сайте партнёра и выбирает способ оплаты "оплата частями РЕВО". Дальнейшее оформление происходит в поп-апе, ссылка на который получается с помощью метода <a href="#registration">`Registration`</a>.

2. На стороне партнёра формируется предварительный заказ с тем же `order_id`, который использовался для вызова формы Рево.

3. В системе Рево заявка переводится в статус hold (блокировка средств) на срок `valid_till`. Если заказ не будет финализирована с помощью метода <a href="#finish">`Finish`</a> до этого срока, то заявка будет отменена. Информация о решении, сумме лимита и графике платежей.

4. Колл-центр партнёра связывается с клиентом для подтверждения заказа.
* Если сумма заказа изменилась и превышает лимит, необходимо отменить заявку с помощью метода <a href="#cancel">`Cancel`</a>.
* Если сумма заказа изменилась в пределах лимита, то необходимо передать обновить информацию о заказе с помощью метода .

5.

<img src="images/F3.png">

## Факторинг 4

## Факторинг 5

## Факторинг 6

# Guides

## Регистрация пользователя

Регистрация нового пользователя осуществляется с помощью метода ....

## Вызов iFrame

```javascript
REVO.Form.show(iframe_url, target_selector);
```

Полученную с помощью метода <a href="#Registration">`Registration`</a> или <a href="#Checkout">`Checkout`</a> ссылку на iFrame необходимо передать в js метод плагина REVO.

`iframe_url` – адрес открываемого iFrame, обязательный параметр.
`target_selector` – селектор элемента, внутрь которого должен быть вставлен iFrame.

Далее работает предоставленный Рево плагин js (реализован в виде модуля REVO), который вставляет ``<iframe src= iframe_url />`` и производит манипуляции с этим iFrame.

Плагин доступен по адресу: `https://{BASE_URI}/javascripts/iframe/v2/revoiframe.js` и его можно добавить на страницу интернет магазина.

```html
<script src="https://{BASE_URI}/javascripts/iframe/v2/revoiframe.js"></script>
```

Также данный плагин предоставляет возможность колбэков закрытия формы, загрузки формы и принятия решения по заявке:

```javascript
REVO.Form.onClose(function () { alert('closed'); });
REVO.Form.onLoad(function () { console.log('frame loaded'); });
REVO.Form.onResult(function() { console.log('result'); });
```
