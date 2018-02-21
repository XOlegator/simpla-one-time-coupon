# Simpla CMS: расширенные многоразовые купоны
Реализация для Simpla CMS возможности использовать многоразовый купон только один раз на клиента. Появится возможность для каждого купона установить флаг, является ли данный купон купоном только для одного применения клиентом. Т.е. один купон смогут применять сколько угодно клиентов, но каждый из клиентов не более одного раза.

## Установка

Simpla CMS не предусматривает возможности добавления сторонних модулей. Внедрение даной доработки потребует изменения системных файлов. Проверено на Simpla CMS 2.3.8. При обновлении CMS работа данного решания, вероятно, будет нарушена. Потребуется повторное выполнение (проверка) всех шагов установки.

### Настройки в базе данных

Для хранения дополнительного флага "Только для первой покупки" потребуется добавления столбца в таблицу купонов. Выполнить в СУБД SQL-запрос:
```sql
ALTER TABLE `s_coupons` ADD `for_first_buy` boolean;
```

### Доработка класса купонов (Coupons)

Класс Coupons - файл /api/Coupons.php
1. В методе get_coupon() добавить новое поле в перечень полей. Для базовой установки Simpla CMS строку
```php
$query = $this->db->placehold("SELECT c.id, c.code, c.value, c.type, c.expire, min_order_price, c.single, c.usages,
```

заменить на
```php
$query = $this->db->placehold("SELECT c.id, c.code, c.value, c.type, c.expire, min_order_price, c.single, c.usages, c.for_first_buy,
```

2. В методе get_coupons() добавить новое поле в перечень полей. Для базовой установки Simpla CMS строку
```php
$query = $this->db->placehold("SELECT c.id, c.code, c.value, c.type, c.expire, min_order_price, c.single, c.usages,
```
```php
$query = $this->db->placehold("SELECT c.id, c.code, c.value, c.type, c.expire, min_order_price, c.single, c.usages, c.for_first_buy,
```

3. В методе add_coupon() после строк
```php
		if(empty($coupon->single))
			$coupon->single = 0;
```
добавить строки:
```php
		if (empty($coupon->for_first_buy)) {
			$coupon->for_first_buy = 0;
		}
```

4. В конец файла (перед последней закрывающейся фигурной скобкой) добавить метод проверки купона только для первой покупки:
```php
    /**
     * Метод определяет, можно ли применять купон. Для купонов, которые только для однократного использования одним пользователем,
     * этот метод вернёт ЛОЖЬ в случае, если у клиента есть не отменённые заазы с аким купоном
     * @param integer $userId Идентификатор клиента
     * @param string $coupon Купон
     */
    public function isFirstUse($userId, $coupon)
    {
        $result = false;

        if (empty($userId) || empty($coupon)) {
            $result = true;
        } else {
            $query = $this->db->placehold("SELECT o.id FROM __orders AS o, __coupons AS c WHERE c.for_first_buy=1 AND c.code=? AND c.code=o.coupon_code AND o.user_id=? AND o.status<>3 LIMIT 1", $coupon, intval($userId));
            $this->db->query($query);
            $ids = $this->db->results();

            if (empty($ids)) {
                $result = true;
            } else {
                $result = false;
            }
        }

        return $result;
    }
```

### Доработка класса корзины (CartView)

В файле /view/CartView.php. Строки
```php
				if(empty($coupon) || !$coupon->valid)
				{
		    		$this->cart->apply_coupon($coupon_code);
					$this->design->assign('coupon_error', 'invalid');
				}
				else
				{
					$this->cart->apply_coupon($coupon_code);
					header('location: '.$this->config->root_url.'/cart/');
				}
```

заменить на
```php
				if(empty($coupon) || !$coupon->valid)
				{
		    		$this->cart->apply_coupon($coupon_code);
					$this->design->assign('coupon_error', 'invalid');
                } else if (!$this->coupons->isFirstUse($this->user->id, $coupon_code)) {
                    // Проверили, что купон только для первого использования: проверка не пройдена
                    $this->cart->apply_coupon($coupon_code);
                    $this->design->assign('coupon_error', 'only_first');
                }
				else
				{
					$this->cart->apply_coupon($coupon_code);
					header('location: '.$this->config->root_url.'/cart/');
				}
```

### Доработка внешнего вида корзины

Файл /design/default/html/cart.tpl. После строки
```php
			{if $coupon_error == 'invalid'}Купон недействителен{/if}
```
добавить строку
```php
			{if $coupon_error == 'only_first'}Вы уже использовали этот купон ранее{/if}
```

### Сохранение значения флага при сохранении купона в админке

В файле /simpla/CouponAdmin.php после обработки параметров
```php
			$coupon->value = $this->request->post('value', 'float');			
			$coupon->type = $this->request->post('type', 'string');
			$coupon->min_order_price = $this->request->post('min_order_price', 'float');
			$coupon->single = $this->request->post('single', 'float');
```
добавить строку
```php
			$coupon->for_first_buy = $this->request->post('for_first_buy', 'boolean');
```

### Отображение нового флага в форме редактирования купона

В файле /simpla/design/html/coupon.tpl после строк
```php
				<li>
					<label class=property for="single"></label>
					<input type="checkbox" name="single" id="single" value="1" {if $coupon->single==1}checked{/if}> <label for="single">одноразовый</label>					
				</li>
```
добавить строки
```php
				<li>
					<label class=property for="for_first_buy"></label>
					<input type="checkbox" name="for_first_buy" id="for_first_buy" value="1" {if $coupon->for_first_buy==1}checked{/if}> <label for="for_first_buy">только на первый заказ</label>
				</li>
```

### Отображение нового флага в списке купонов

В файле /simpla/design/html/coupons.tpl после строк
```php
	 				<div class="detail">
	 				Одноразовый
	 				</div>
```
добавить строки
```php
					{if $coupon->for_first_buy}
	 				<div class="detail">
	 				Только на первый заказ
	 				</div>
	 				{/if}
```
